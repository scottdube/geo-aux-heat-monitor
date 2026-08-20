# Bill of materials — Geo Aux Heat

Geothermal auxiliary-heat project. Already built - this record exists to claim the parts it consumed. Function not yet documented.

**InvenTree part** `#834`  ·  **Build** `BO-0007` (Pending)

| Qty | Part | ID | Where it lives |
|---:|---|---|---|
| 2 | Resistor 100k 1% 1/4W | `625` | Kit - EAONE Resistor 30-value |
| 2 | Resistor 3.3k 1% 1/4W | `617` | Kit - EAONE Resistor 30-value |
| 2 | PC817 Optocoupler DIP-4 | `17` | B3-R3C1 |
| 1 | Capacitor Ceramic 100nF | `645` | no stock |
| 1 | Capacitor Electrolytic 10uF 25V (5x11) | `694` | A3-R8C4 |
| 1 | Capacitor Electrolytic 220uF 16V (8x12) | `879` | no stock |
| 1 | Capacitor Electrolytic 470uF 63V (13x21mm) | `672` | no stock |
| 1 | MB10S Bridge Rectifier 1000V 0.5A SOP-4 SMD | `878` | no stock |
| 1 | LM2596 Buck Module ADJ, Vin 3-40V | `28` | B3-R3C2 |
| 1 | ESP32 ESP-32S ESP-WROOM-32 Dev Board 38-pin CP2012 USB-C | `58` | B3-R6C3 |
| 2 | KF301 Screw Terminal Block 5.08mm, 2-position | `291` | A3-R8C7 |
| 1 | KF301 Screw Terminal Block 5.08mm, 3-position | `821` | A3-R8C7 |

## Notes on individual lines

- **Resistor 100k 1% 1/4W** — R3 R4 - opto LED current limit off the rectified 24 VAC rail [as-built, from sln-geo-aux-heat.kicad_sch]
- **Resistor 3.3k 1% 1/4W** — R1 R2 - opto bias [as-built, from sln-geo-aux-heat.kicad_sch]
- **PC817 Optocoupler DIP-4** — U2 U3 - GPIO26 COMMANDED (contactor coil), GPIO27 STAGED (NCM-300 W2/E) [as-built, from sln-geo-aux-heat.kicad_sch]
- **Capacitor Ceramic 100nF** — C4 - 0.1 uF disc, bias decoupling [as-built, from sln-geo-aux-heat.kicad_sch]
- **Capacitor Electrolytic 10uF 25V (5x11)** — C2 - 10 uF, bias decoupling [as-built, from sln-geo-aux-heat.kicad_sch]
- **Capacitor Electrolytic 220uF 16V (8x12)** — C3 - 220 uF/16 V [as-built, from sln-geo-aux-heat.kicad_sch]
- **Capacitor Electrolytic 470uF 63V (13x21mm)** — C1 - 470 uF/63 V main filter. 24 VAC rectifies to ~34 V peak, so the 63 V part is correct and a 25 V would not be. [as-built, from sln-geo-aux-heat.kicad_sch]
- **MB10S Bridge Rectifier 1000V 0.5A SOP-4 SMD** — D1 - 24 VAC to DC. Custom footprint sln_geo_aux:MB10S_ADJACENT, drawn for the milled board. Comes out of the 8-value SMD bridge kit [13]. [as-built, from sln-geo-aux-heat.kicad_sch]
- **LM2596 Buck Module ADJ, Vin 3-40V** — U4 - LM2596-40V buck module, THT. 40 V ceiling is required; MP1584 / R-78 class parts top out near 28-32 V and would fail. [as-built, from sln-geo-aux-heat.kicad_sch]
- **ESP32 ESP-32S ESP-WROOM-32 Dev Board 38-pin CP2012 USB-C** — U1 - NodeMCU-32S 38-pin. ESP32 required for ADC1 plus spare GPIO. [as-built, from sln-geo-aux-heat.kicad_sch]
- **KF301 Screw Terminal Block 5.08mm, 2-position** — J1 24VAC R+C, J3 CT. Schematic footprint is MaiXu MX126 (5.0 mm); KF301 is 5.08 mm and interchangeable in practice - VERIFY against the board if exact vendor matters. [as-built, from sln-geo-aux-heat.kicad_sch]
- **KF301 Screw Terminal Block 5.08mm, 3-position** — J2 - 3-position. Same MX126 vs KF301 pitch note as J1/J3. [as-built, from sln-geo-aux-heat.kicad_sch]

---

_Generated from InvenTree on 2026-08-19 — do not edit by hand._
_Regenerate:_ `itq run link_repos.py && write_boms.py`
