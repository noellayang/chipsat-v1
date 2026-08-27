# Manufacturing outputs

- `fab_outputs/` — gerbers, drill files, silkscreen, solder mask, and V-cut data exported for fabrication.
- `fab_archives/` — the zipped fab-house submission packages, kept as-is for reference.

The KiCad project in `../hardware/` is the source of truth. Regenerate these from
`hardware/chipsat_v1.kicad_pcb` (Plot > Gerbers, then Generate Drill Files in KiCad)
if you need a different panelization or a different fab house's naming convention.
