# ChipSat (v1), a Solar-Powered CubeSat Power Testbed

A custom 45mm x 45mm PCB built to test how firmware power-management decisions actually change the electrical behavior of a small-satellite-style system. Built as a testbed for the Duke Solar Sail project's power budget work.

<p align="center">
  <img src="images/board_photo.jpg" width="500" alt="Assembled ChipSat v1 board, 45mm x 45mm">
</p>

## What this is

Rather than trying to reproduce actual flight hardware, this board answers a narrower question: given the same physical system (solar input, battery charging, sensors, radio), how much does the current draw change just from firmware scheduling decisions? The board integrates:

- **ESP32-C3-MINI-1** MCU with native USB
- **Solar input + Li-ion charging** (BQ25185)
- **Two INA219 current/voltage monitors,** one on the solar input path, one on the system rail
- **ICM-42688-P** 6-axis IMU (SPI)
- **VEML7700** ambient light sensor
- **Switchable +3V3_SENS rail** for the sensor domain
- **36 Ω switched load** behind a solder jumper to emulate a ~300 mW ADCS-scale event (comparable to a single magnetorquer actuation)
- BLE (via the ESP32-C3's radio) as a telemetry stand-in for LoRa downlink — same 780 ms TX window and cadence, without trying to match LoRa's RF characteristics

## Repo layout

```
hardware/         KiCad project (schematic, PCB layout, design rules)
manufacturing/    Gerbers, drill files, pick-and-place, fab-house zip outputs
images/           Board photo, result figures
docs/             BOM
```

## Firmware power states

| State | MCU | IMU | +3V3_SENS | BLE proxy | Mission interpretation |
|---|---|---|---|---|---|
| Eclipse sleep | Deep sleep between events | Off | Off | Wake every 30 s, advertise 0.78 s | EARTH-SHADOW stress case, nominal telemetry retained |
| Housekeeping | Active | Off | Off | 0.78 s every 30 s | Autonomous scheduling / health checks |
| Full sensing | Active | On | On | 0.78 s every 30 s | Complete sensing window |

## Results

**Deep sleep cuts average current by 95.4%.** Running each state over a representative 30 s cycle: Eclipse averaged 2.25 mA, Housekeeping 48.59 ± 0.10 mA, Full Sensing 50.14 ± 0.11 mA. A separate switched-load test (the 36 Ω / ~300 mW ADCS emulation) showed current rising from 26.02 mA at baseline to 118.48 ± 0.87 mA with the load on — close to the 300 mW resistive-load prediction.

<p align="center">
  <img src="images/power_states.png" width="700" alt="System current across power states and switched-load test">
</p>

**BLE telemetry is short but not free.** With the board held in Full Sensing for 60 s and BLE advertising enabled for 780 ms every 30 s, baseline current stayed tight at 53.67 ± 0.10 mA across three trials, with brief spikes up to 121.5 mA (INA219-observed) during the TX windows. Radio activity works out to about a 2.6% duty cycle and shows up as a short burst on top of a stable baseline rather than dominating the whole cycle.

<p align="center">
  <img src="images/ble_telemetry.png" width="700" alt="System current during periodic BLE telemetry">
</p>

**Solar input scales with light as expected.** Using the solar-side INA219 alongside the VEML7700 for illuminance, harvested power rose from about 3.01 ± 0.68 mW at ~1669 lux to 16.77 ± 0.91 mW at ~2669 lux. The lowest-light point sits close to the INA219 / 0.1 Ω shunt's measurement floor, so that point is noisier by nature rather than a sign the panel stopped producing.

<p align="center">
  <img src="images/solar_harvesting.png" width="600" alt="Harvested solar power vs illuminance">
</p>

Every number above traces back to raw CSVs in [`test-data/`](test-data/), and [`analysis/make_figures.py`](analysis/make_figures.py) regenerates matching plots straight from that data — so nothing here is hand-pasted from a lab notebook. The full writeup with methodology is in [`docs/CUBESAT_TESTING.pdf`](docs/CUBESAT_TESTING.pdf).

## Bring-up notes

A few things that changed how I thought about the design while bringing it up:

- **I2C bus switching.** The ESP32-C3 has one I2C controller but the board uses two physical buses, so firmware remaps the controller between them. Cutting power to the sensor rail mid-transaction was a real failure mode — the fix was finishing the transaction and releasing SDA/SCL before disabling the rail.
- **Low-current measurement floor.** At weak illumination, solar current sat close enough to the INA219 / 0.1 Ω shunt's resolution and offset that the sign could drift around zero. Rather than smoothing that over, the lowest-light point is called out as sitting near the measurement floor.
- **BLE stack overhead.** Just initializing the BLE stack raised the active baseline noticeably, even with advertising off. That made "advertising disabled" and "BLE fully deinitialized" two very different power states worth distinguishing in firmware.

## Manufacturing

Gerbers, drill files, and the original fab submission archives for the 2-layer board are in [`manufacturing/`](manufacturing/). The KiCad project is the source of truth. Regenerate fab outputs from `hardware/chipsat_v1.kicad_pcb` if you need a different fab house's format.

## BOM

Full bill of materials with LCSC part numbers is in [`docs/BOM.csv`](docs/BOM.csv).

## Opening the project

Requires KiCad 8+. Open `hardware/chipsat_v1.kicad_pro`.

## License

MIT — see [LICENSE](LICENSE). The hardware design files are included for reference and reuse; a credit back to this repo is appreciated if you build on the board itself.
