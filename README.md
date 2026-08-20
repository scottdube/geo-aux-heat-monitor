# geo-aux-heat-monitor

ESP32/ESPHome monitor for the SLN geothermal electric-resistance backup heat
(ClimateMaster Tranquility 20 TSV048 + Genesis AGL10A 9.6 kW / 40 A strip kit).
Senses the strip directly — independent of the ecobee — and reports to Home
Assistant over the native API.

**Status:** Board designed, milled, and populated — this was the first PCB cut
on the Tormach 1100MX. Flashing/commissioning state not recorded here.

The board is a **milled single-sided PCB**, not a fab-house order. Design files
live in `sln-ha-config`, not this repo:

- `sln-ha-config/electronics/kicad/sln-geo-aux-heat/sln-geo-aux-heat.kicad_sch`
- `…/sln-geo-aux-heat.kicad_pcb` and `…/Gerbers/`

The same gerbers are checked in as `pcbmill/tests/fixtures/` — this board is the
regression fixture for the milling toolchain.

## Sensing architecture (three-level state machine)

| Level | Signal | Tap point | GPIO |
|---|---|---|---|
| STAGED | 24 VAC on NCM-300 SYSTEM `W2/E` → `C` | EWC NCM-300 zone panel | 27 |
| COMMANDED | Strip contactor coil energized | WILSPEC CT3XQ04CG coil (verify — see ADR open questions) | 26 |
| DRAWING | CT amps > ~5 A (template) | Spare Emporia Vue Gen 3 50 A CT on one strip leg | 34 (ADC1) |
| (optional A) | Dry-contact current switch | Same strip leg | 25 |

Fault logic: COMMANDED & !DRAWING >60 s = contactor/element/breaker fault;
STAGED & !COMMANDED >120 s = control fault. FL-away alert lives in HA, not ESPHome.

Power: air-handler 24 VAC tap → bridge rectifier → 470 µF filter → buck
(≥40 V-in, LM2596-class) → 5 V into ESP32 VIN.

## Repo layout

- `docs/adr/sln-adr-013-geo-aux-heat-esp32-monitor.md` — copy of the design ADR.
  **Canonical version lives in `scottdube/sln-ha-config`** (`docs/decisions/013-…`);
  amendments happen there first.
- `docs/bom.md` — bill of materials with quantities and have/buy status.
- `esphome/sln-geo-aux-heat.yaml` — **reference copy.** The deployed source of
  truth is `sln-ha-config/esphome/sln-geo-aux-heat.yaml`; `esphome run` happens
  from that repo per the standard deploy workflow. Kept byte-identical so
  `diff` against canonical is clean — do not edit here first.

## Related (not copied)

- SLN ADR-012 — energy monitoring architecture (dropped Panel 2 Vue CT that
  this project replaces)
- `sln-ha-config/docs/reference/energy-monitoring-ct-map.md`
- Unit wiring diagram 96B0005N36; ClimateMaster TS20 Technical Guide §5
  (TS124 unit electrical, TS132 aux-heat)
