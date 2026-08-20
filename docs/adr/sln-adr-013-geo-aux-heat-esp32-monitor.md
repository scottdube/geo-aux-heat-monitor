> **COPY — canonical version lives in `scottdube/sln-ha-config`
> (`docs/decisions/013-geo-aux-heat-esp32-monitor.md`).** Amend there first,
> then re-sync this copy. Snapshot date: 2026-07-05.

# SLN ADR-013 — Geothermal aux-heat ESP32 monitor

**Status:** Accepted — design; not yet built/flashed
**Date:** 2026-06-15
**Decider:** Scott Dube
**Related:**
- SLN ADR-012 (energy monitoring) — this replaces the dropped Panel 2 / Geo Heat Vue CT
- SLN ADR-001 / ADR-009 (ESPHome device patterns, IoT VLAN placement)
- Hardware: ClimateMaster Tranquility 20 TSV048 geo HP; EWC NCM-300 zone panel (W2 stager)

---

## Context

SLN's geothermal system is a **ClimateMaster Tranquility 20 TSV048** (4-ton,
208/230-1Ph water-source HP). Its **electric-resistance backup** is a **ClimateMaster Genesis AGL10A** kit —
**9.6 kW / 40 A at 240 V, single-stage** (confirmed nameplate) — on a separate
**50 A 2-pole** circuit (Panel 2 "Geo Heat"). The strip conductors and a
**WILSPEC CT3XQ04CG** contactor (3-pole, 40 FLA / 50 A DP) live in the **unit's
control/heat compartment** alongside the CXM and Electric Heat (EHB) boards.
That contactor is the **inferred (not yet traced) strip contactor** — its 40 FLA
rating matches the AGL10A's 40 A, and it sits with the EHB — but a **second (gray)
contactor** and the compressor **run cap (WILSPEC CA370)** are also in the box, so
**confirm which contactor switches the strip before tapping** (see Open questions). The
aux stage is staged upstream by an **EWC NCM-300 zone panel** (SYSTEM-block W2/E
output, 24 VAC). The strip breaker is normally **kept off** (disarmed). The future use case: when the house is unattended (Scott in FL),
arm the backup as a freeze/comfort safety net, and get a **trustworthy alert
when it actually energizes** — which doubles as a "geo heat pump can't keep up
/ extreme cold" signal.

ADR-012 originally put a Vue CT on this circuit, then dropped it: metering Panel 2
from the Panel-1 Vue meant a low-voltage CT lead across two separate service
enclosures (messy, inspection-unfriendly).

The obvious software path — read the ecobee's `equipment_running` for `auxHeat1`
via the HA ecobee integration — is **rejected**: Scott does not trust the ecobee,
and its HA integration is not currently working. (The integration's `aux_heat`
attribute is also a known false-negative; only `equipment_running` is reliable.)
We want a signal that is independent of the thermostat and proves the strip is
**actually drawing current**.

## Decision

A **dedicated ESP32/ESPHome monitor** (mounted in the HVAC air handler — see
Amendment 2026-06-15; originally Panel 2) that senses the strip directly
and reports to HA over the native API. Three sensing channels:

- **A — current switch (authoritative on/off).** A self-powered split-core
  **current switch** (adjustable, ~50 A range, dry-contact output — e.g.
  Functional Devices RIBXGTA or NK Technologies AS-series) clamped on **one leg**
  of the Geo Heat circuit → ESP32 digital input (pullup) → `binary_sensor`
  `geo_aux_heat_running`. No calibration, galvanically isolated, can't drift.
  Trip set low (~5–10 A); the strip is all-or-nothing.
- **B — spare Emporia 50 A CT (kW data).** Reuse a leftover **Emporia
  voltage-output CT** from the shop Vue's kit (ships with 16, uses ~7). They are
  333 mV-class with the **burden built in** (~20 Ω), full-scale sub-1 V, so they
  drop into the same path as any voltage-output CT: one leg → ESP32 **ADC1**
  through a Vcc/2 bias network → ESPHome `ct_clamp` → RMS amps. Power = `I × 240`
  (single 240 V load, **no ×2**). No external burden; safe open-circuit.
  **Saves buying an SCT-013-050** (which stays the fallback if no spare is free —
  its larger 1 V swing gives slightly better resolution).
- **C — aux "commanded" sense (optional diagnostic).** Opto-isolate **across the
  heat-strip relay/contactor coil** in the air handler (parallel, high-impedance)
  → digital input → `binary_sensor` `geo_aux_called`. Tapping the *coil* (rather
  than the thermostat W) means "the board commanded the strip ON right now," so the
  fault template **commanded but not drawing** for >60 s = stuck/failed contactor,
  open element, or strip breaker off — with no false alarms from normal
  anti-short-cycle / sequencer delays. **Confirm coil voltage** (usually 24 VAC →
  PC817 + bridge + ~10–15 kΩ dropping resistor; if a 240 V-coil contactor, size the
  opto front-end for mains; if a thermal *sequencer*, tap its 24 VAC heater winding
  and set the fault debounce above its ramp time). Trade-off: a coil tap won't catch
  a board that ignores W — add a second GPIO on the thermostat W terminal for that.

With B now a free reused CT, **channel A is optional**: the on/off can instead be
a template `binary_sensor` on B's amps (>~5 A). Keep the hardware current switch
only if you want a calibration-independent trigger for the safety alert — then the
BOM is one current switch; CT-only makes the BOM ~$0.

**Host: ESP32 dev board (e.g., ESP32-WROOM DevKit), not ESP8266.** Channel B needs
a clean analog read; the ESP8266's single 0–1 V `A0` would be fully consumed and is
noisy. ESP32 gives ADC1 (ADC2 is unusable with WiFi) plus spare GPIO for A and C.

Device on the **SLN IoT VLAN (192.168.30.0/24)**, powered from the air-handler
24 VAC (see Amendment 2026-06-15), printed enclosure (fillets + cable gland, per
the shop-device pattern). The
**FL-away alert lives in HA** (vacation-gated automation on `geo_aux_heat_running`
→ `notify.scott_and_ha`), not in ESPHome.

## Consequences

**Positive:** Trustworthy, thermostat-independent proof the strip is drawing;
kW/runtime trend; a failure-mode catch (called-but-cold). No cross-panel CT lead;
no ecobee dependency. Squarely in Scott's ESPHome wheelhouse, parts mostly on hand.

**Negative:** A new mains-adjacent build (current switch + CT in a live 50 A
panel; opto on 24 VAC). One more ESPHome device on the IoT VLAN. ESP32 ADC
nonlinearity means `ct_clamp` needs a real `calibrate_linear` pass.

**Neutral:** Only relevant while the backup is armed; idle when the breaker is off.

## Implementation

- Staged config: `esphome/sln-geo-aux-heat.yaml` (unflashed). Pinout to verify on
  bench: CT/ADC = GPIO34 (ADC1, input-only); current switch (opt A) = GPIO25;
  contactor coil (COMMANDED) = GPIO26; NCM-300 W/E (STAGED) = GPIO27 — all pullup,
  optos pull low. `geo_aux_drawing` is a template on CT amps (>5 A).
- Calibration (B): the Emporia CT's exact mV/A isn't published (333 mV-class,
  ~6–20 mV/A depending on ratio), so calibrate empirically — `0.0 -> 0.0` plus one
  clamp-meter point with the strip energized. Smaller full-scale than an SCT = a
  coarser ADC swing; fine for an alert + rough kW.
- Secrets: reuses the repo's shared `!secret api_encryption_key` / `ota_password` / `wifi_*` / `fallback_ap_password` (same pattern as the other SLN ESPHome nodes) — none new.
- Network (to network-docs): DHCP reservation on IoT VLAN 30 after first join.
- HA: vacation-gated alert automation + optional `total_daily_energy` for kWh.

## Open questions

- **Which contactor is the strip contactor? (not yet traced)** Confirm the WILSPEC
  CT3XQ04CG switches the *heat strip*, not the compressor, before tapping its coil:
  (a) coil wires land on the **EHB**, not the CXM; (b) load side feeds the **heat
  elements**, not the compressor (the compressor leg carries the CA370 run cap);
  (c) line side is the **Panel 2 "Geo Heat" 50 A** feed. No-meter discriminator:
  with the Geo-Heat breaker **off**, the strip contactor stays dead / won't cycle,
  while the compressor contactor still clicks on a cooling/heating call. Documentary basis (unit wiring
  diagram **96B0005N36**): the schematic distinguishes **CC** (compressor contactor)
  from **CBS** (the optional **AG-series** electric-heat contactor, wired to the EHB
  with element relays ER1–4 / limits TS1–3 / elements HE1–3). The WILSPEC's **40 FLA**
  matches the 40 A heat (CBS), not the 20.5 RLA compressor (CC = the smaller gray
  contactor by the CA370 run cap). The diagram's **Control Box Layout pictorial is
  unlabeled**, so it shows positions but doesn't tag the physical part. **Manufacturer
  electrical data closes it** (TS20 Technical Guide §5: TS124 unit data, TS132 aux-heat):
  AGL10A is a **single-circuit, single-stage, 40.0 A** heater, vs the compressor's
  **20.5 RLA** — so the 40 FLA WILSPEC is the *heat* contactor by rating, full stop.
  Only the physical wire tie-out (coil→EHB / breaker-off test) remains, and that's a
  10-second visual, not an open design question.
- **C worth wiring?** The called-but-cold fault check needs tapping 24 VAC at the
  air handler. Decide at build — skip if the panel-only A+B is enough.
- **Keep channel A at all?** With a free reused CT on B, decide whether the
  hardware current switch is worth it (robust, no-cal trigger) or drop it and
  derive on/off from B's amps. If kept, confirm the split-core fits and brackets ~40 A.
- **Power for the ESP32** — resolved by the 2026-06-15 amendment (24 VAC tap →
  rectifier + buck). Still confirm it's 24 VAC (not DC) and the buck is rated ≥40 V.


---

## Amendment 2026-06-15 — relocate to the HVAC air handler; power from 24 VAC

Moved the monitor from the Panel 2 electrical enclosure to the **ClimateMaster
TSV048 electric-heat compartment (base panel of the unit)**, where the three things
we need coexist: the resistance strip's **power leg**, **24 VAC**, and the strip's
**contactor coil**.

- **Power.** Tap 24 VAC at the air handler → **bridge rectifier + filter cap →
  buck regulator → 5 V** into the ESP32 VIN. No mains, and **no 5 V conversion
  inside the electrical panel**. Buck must be rated **≥40 V input** — 24 VAC
  rectifies to ~34 V peak, above the ~28–32 V ceiling of MP1584 / Recom R-78-class
  parts; an LM2596 (40 V) or higher-input buck is the safe pick.
- **CT (B).** Clamp the strip power leg **here** (same current as Panel 2). Vue
  Gen 3 CT leads are **ferrule-terminated, not 2.5 mm plugs** — land them in a
  **2-position screw terminal** (the earlier 2.5 mm-jack note was wrong).
- **Channel C** taps the **heat-strip contactor coil** right here (its load side =
  the strip leg the CT clamps). The EWC NCM-300's W/E (W2) output is an upstream
  fallback, but tapping the unit contactor co-locates all three channels.

**Benefits:** out of the inspectable panel space; no in-panel voltage conversion;
a single cabinet for power + both sensing taps.

**Cautions / verify:** heat kit is **single-stage** (AGL10A, 9.6 kW/40 A) → **one**
WILSPEC CT3XQ04CG contactor; clamp the CT on one strip leg off its load side (T1/T2).
**Read the contactor coil-voltage stamp** — almost certainly 24 VAC (driven by the
EHB off the unit's 24 V control; opto as designed). Confirm 24 V**AC** for ESP power
(if 24 VDC, skip the rectifier); buck input rating ≥40 V; transformer VA headroom is
fine (~0.5 W).

### Revised BOM
- **Have / arriving:** ESP32 dev board; spare Emporia 50 A CT (Vue 16-kit); bias
  passives (2× ~10 kΩ, 10 µF + 100 nF); 2-pos screw terminal; hookup wire;
  printed enclosure + gland.
- **Power bits (likely in bin):** bridge rectifier (or 4 diodes), ~470 µF/50 V
  filter cap, buck reg ≥40 V-in → 5 V (e.g., LM2596 module).
- **Channel C (optional):** PC817 opto + bridge/diode + ~10–15 kΩ dropping resistor.
- **Channel A (optional):** split-core ~50 A dry-contact current switch (Functional
  Devices RIBXGTA / NK AS-series) — only if keeping the hardware trigger.
- **Dropped:** Hi-Link mains 5 V module; 2.5 mm audio jack.


---

## Amendment 2026-06-15b — co-located dual-tap (staged + commanded)

The EWC NCM-300 zone panel and the ClimateMaster electric-heat compartment are
**≤1.5 m apart / effectively the same box**, so one ESP reaches all signals with no
CT-lead extension. Upgraded channel C from a single "commanded" tap to a
**three-level state machine**, since both tap points are now in reach for the cost
of one extra GPIO + opto:

- **STAGED** — NCM-300 **SYSTEM-block `W2/E` → `C`** (24 VAC, the white wire; GPIO27)
  → the zone panel staged aux. The board's on-board **W2/E LED** confirms assertion.
- **COMMANDED** — strip **contactor coil** (GPIO26) → the contactor engaged.
- **DRAWING** — template on the **CT amps** (>5 A, GPIO34) → strip actually pulling.

Two split fault sensors localize the failure:
- **Fault (Contactor)** = commanded & **not** drawing >60 s → dead contactor, open
  element, or strip breaker off.
- **Fault (Control)** = staged & **not** commanded >120 s → unit control / relay
  never engaged the contactor (debounce allows the downstream engage delay).

The optional hardware current switch (A) remains a no-cal on/off; the fault logic
keys off `geo_aux_drawing` (CT-derived) so it works with or without A. HA FL-away
alert keys off `geo_aux_drawing` (or A); both fault sensors notify regardless of
vacation state.
