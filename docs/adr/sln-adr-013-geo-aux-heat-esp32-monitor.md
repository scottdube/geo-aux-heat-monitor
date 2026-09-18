> **COPY — canonical version lives in `scottdube/sln-ha-config`
> (`docs/decisions/013-geo-aux-heat-esp32-monitor.md`).** Amend there first,
> then re-sync this copy. Snapshot date: 2026-09-18.

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

---

## Amendment 2026-07-04 — heat-call tap relocated to W2 (24 V **DC**) at the CXM P3 / EHB harness

On-site tracing (ClimateMaster TSV048) resolved where the aux-heat command actually
lives, and it's cleaner than the NCM-300 tap: it's **inside the unit's heat cabinet**,
co-located with the CT.

**Signal path (per the unit wiring diagram):** CXM main board **P3** = pins
`EH2 / EH1 / 24V DC` → a **RED / WHT / BLK** harness (plug-terminated both ends) →
Electric Heat board (**EHB**, ClimateMaster 17B0003N02, 4× Tyco T9AV1D22-22 relays,
2 used) connector **P2** = `W2 / W1 / +V`. So:
- **RED = W2** (stage-2 aux call)
- **WHT = W1** (stage-1 aux call)
- **BLK = 24 V DC** rail (reference/common)

**COMMANDED tap (channel C) — revised:** Post-Tap / insulation-displacement tap on
the **RED (W2)** wire (the harness is plug-to-plug, no screw terminal), referenced to
**BLK (24 V DC)**. Front-end is now **DC**, not AC — **no bridge rectifier**: just a
~2.2–3.3 kΩ current-limit resistor into a PC817 → ESP GPIO. **Polarity set on site:**
measure RED relative to BLK with heat *called* — if it's active-LOW (RED → 0 V on call,
+24 V idle, the usual CXM scheme) the opto goes **anode→BLK, cathode→R→RED**; if
active-HIGH, reverse. Consider tapping **WHT (W1)** instead/also — on this single-stage
AGL10A, W1 is likely the stage that actually engages.

**DRAWING (channel B):** unchanged in intent — CT on a 240 V element leg through the
EHB relays (or the incoming 240 V leg). Single-stage → one CT.

**Supersedes:** the NCM-300 SYSTEM `W2/E` tap (Amendment 2026-06-15b) and the separate
strip-contactor-coil tap. Everything — W2 signal, 240 V for the CT, and 24 V(DC) for
powering the ESP — is in the one heat cabinet. The NCM-300 stays untouched.

**Note on ESP power:** the ESP still needs 5 V. If the CXM's 24 V is **DC** and
accessible in the cabinet, a buck 24 VDC→5 V is even simpler (no rectifier). Confirm the
24 V DC rail can source the ESP's ~0.5 W without disturbing the control logic (or keep a
separate 24 VAC→buck tap if the DC rail is current-limited).

---

## Amendment 2026-07-08 — DESIGN SETTLED (manufacturer-documented). Supersedes all prior amendments.

Everything below is from **ClimateMaster's own documentation**, not inference:
**CXM manual 97B0003N12** (Table 5 I/O reference + troubleshooting section) and unit
wiring diagram **96B0016N02**. Several earlier amendments here were built on guesswork
and are **wrong** — this supersedes them.

### The signal interface (documented)

CXM (**17B0001N01**) connector **P3**, silk `[24V DC][blank][EH1][EH2]`:

| Wire | CXM terminal | Manual Table 5 | Lands on EHB P2 |
|------|--------------|----------------|-----------------|
| BLK  | **24VDC** | *"24 VDC supply to electric heat module"* (Output) | `+V` |
| WHT  | **EH1**   | *"Output terminal for stage 1 electric heat"* | `W1` |
| RED  | **EH2**   | *"Output terminal for stage 2 electric heat"* | `W2` |

The manual's troubleshooting section states the method outright:
> *"Electric heat outputs are **24VDC** and require a voltmeter set for DC to verify
> operation. When troubleshooting, **measure from 24VDC terminal to EH1 or EH2 terminals**."*

**ACTIVE-LOW confirmed.** The EHB (**17B0003N02**, 4× TE **T9AV1D22-22**, 2 used) is a
**passive** relay board: each coil sits between `+V` and its W line, so **the W line is
the coil's return** — which is why the EHB has **no local DC common, by design**. The CXM
energizes a coil by pulling that W line toward the DC common.

**The 24VDC supply FLOATS relative to chassis.** This caused a long detour: measuring to
chassis gave nonsense (BLK +12 V, WHT −10 V, "15 VAC" on everything) because a floating
supply self-biases to ~±12 V. Subtracting reconciles it: BLK→WHT = +12 −(−10) = **~22–24 V
across the coil**, and idle RED sitting **at BLK potential** = coil off. **Chassis is NOT
the DC common — never reference the optos to it.**

### Design

- **DRAWING:** spare Emporia 50 A CT (voltage-output, burden internal) on a 240 V element
  leg → **ADC1 / GPIO34** → amps → W (`I × 240`). Stage-agnostic ground truth.
- **CMD W1:** PC817 — **anode → BLK (24VDC)**, **cathode → 2.2–3.3 kΩ → WHT (EH1)** → GPIO26 (pullup, LOW = commanded).
- **CMD W2:** PC817 — **anode → BLK**, **cathode → 2.2–3.3 kΩ → RED (EH2)** → GPIO27 (pullup, LOW = commanded).
- **Sense BOTH W1 and W2** — independent of how the CXM/NCM-300 stages. Optos are
  self-contained on the harness (no ground needed) and isolate the floating DC control
  circuit from the AC-powered ESP domain.
- **Tap:** insulation-window T-splice on BLK/WHT/RED **in the heat cabinet**; do **not**
  sever conductors. Opto ~8 mA vs a 42 mA coil (T9A 24 V = 1 W / 576 Ω) — invisible to staging.
- **Host:** ESP32-WROOM DevKit (`esp32dev`). ADC1/GPIO32–39 must stay free for the CT.

### Power (resolved)

**R + C (24 VAC)** at the CXM low-voltage block → **MB10S** bridge → **470 µF/63 V** →
**LM2596HVS** buck set to **5.0 V** → ESP32 VIN.

Wiring-diagram **note 7**: *"Transformer secondary ground via CXM/DXM board standoffs &
screws to control box. (Ground available from top two standoffs as shown.)"* — so **C is a
grounded common**, and the CXM manual confirms *"the control board is grounded through two
of the metal standoffs."* CXM draws 5–9 VA of a ≥40 VA transformer → ample headroom.
**Pull R + C up alongside the existing harness** (2 conductors).

**A plain LM2596 (40 V max) is NOT safe** — 26–28 VAC rectifies to ~38–40 V peak. Use the
HV part. Set the buck output to **5.00 V with a meter, no load, BEFORE** connecting the ESP.
Since C is grounded, a **non-isolated** buck is fine; the optos provide the isolation that matters.

### CXM behavior that shapes the logic (manual)

- **Heating Stage 2:** EH1 **immediately**; EH2 **after 10 minutes**.
- **Emergency Heat:** EH1 **immediately**; EH2 **after 5 minutes**.
- **`G` must be active or `W` is ignored entirely.**
- EH2 inhibited/dropped if **LT1 > 45 °F AND LT2 > 110 °F**.
- **DIP 4 must be ON** (= "EH2 Normal"). If OFF, EH2 becomes a **DDC fault-code output** and
  W2 would monitor garbage. **Verified ON 2026-07-08** (all 5 DIPs ON = factory default).

**Consequence:** EH2 legitimately lags EH1 by 5–10 min, so **fault logic keys off ANY
command (W1 OR W2), never W2 alone.** This also explains why a ~60 s Em Heat test could only
ever show EH1 — nothing was wrong.

### Fault matrix (the actual deliverable — FL-away awareness)

| Fault | Condition | Means |
|---|---|---|
| **Commanded not drawing** | `commanded && !drawing` >60 s | open element / tripped heater breaker / failed EHB relay → **call for service** |
| **Drawing not commanded** | `drawing && !commanded` >60 s | welded/stuck relay energizing elements → cost leak **and safety flag** |
| *(HA-side)* excessive aux runtime | long `drawing` | geo side underperforming → early warning |

### Supersedes

- **NCM-300 `SYSTEM W2/E` tap** (Amdt 2026-06-15b) — dropped; NCM-300 untouched.
- **Strip-contactor-coil tap** (WILSPEC CT3XQ04CG) as COMMANDED — dropped; the coils are the
  EHB's, and the T9A datasheet confirms coils are **PCB-soldered** (mounting code 2: QCs are
  **contacts only**, 240 V — never tap those for logic).
- **Channel A current switch** — dropped; DRAWING comes from the CT.
- **"Chassis is the DC common"** and **"no wire pull needed"** — both wrong.
- **Single-stage assumption** — the interface is genuinely 2-stage (EH1/EH2).

### Parts

On hand: ESP32 dev board · PC817 ×2 · resistors · Emporia 50 A CT · 470 µF/63 V · enclosure ·
splice materials · **SMD bridge kit (MB10S)** · **LM2596HVS**.
Prototype on a **mixed SMD/through-hole protoboard** (e.g. BusBoard SMT3U).

### Open

- CT `calibrate_linear` is a **placeholder** — do a clamp-meter pass with elements energized.
- **Measure R→C (AC) before finalizing the buck** — if meaningfully >28 VAC, recheck margins. *(Resolved 2026-08-01: 26.4 VAC — see amendment below.)*
- Verify GPIO26/27 pin choices + opto pull-up behavior on the bench.

---

## Amendment 2026-08-01 — KiCad schematic captured; **40 V buck validated by measurement** (supersedes the "use the HV part" note)

Schematic capture of the monitor is complete in `electronics/kicad/sln-geo-aux-heat/`; **ERC is clean** (0 errors, 0 warnings). Two substantive changes from the 2026-07-08 design:

**1. Buck: standard 40 V LM2596, not LM2596HVS.** Measured **R→C = 26.4 VAC** at the CXM low-voltage block. Rectified DC ≈ (26.4 × 1.414) − ~1.8 V (two MB10S drops) = **~35.5 V** at the buck input — under the standard LM2596's 40 V max with **~4.5 V margin**. So the in-stock **40 V** part is used. This **supersedes the 2026-07-08 claim** that "a plain LM2596 (40 V) is NOT safe … use the HV part," which assumed 26–28 VAC *without* the measurement and *without* subtracting the bridge drops.
- **High-line caveat:** sustained +10 % line (~29 VAC) → ~39 V DC — under 40 V but thin (45 V abs max; the cap rides transient spikes). If the buck ever runs hot or drops out, suspect this. If the transformer ever measures ≥28 VAC in service, revert to the HVS.
- Still **set the buck to 5.00 V, no load, before connecting the ESP.**

**2. D1 bridge — package + symbol pin-map nailed down.** MB10S is the **MBS package = JEDEC TO-269AA**; footprint `Package_TO_SOT_SMD:TO-269AA`. The device pinout is **1=+, 2=~, 3=−, 4=~** (+ and − on the diagonal), so the schematic symbol must be `Device:D_Bridge_+A-A` (the generic `+-AA` mis-maps − onto an AC leg). The footprint's `+`/`−` silk is generic decoration and does **not** match MB10S — **orient the physical part by its pin-1 marker to pad 1**, not by the silk.

**Footprints locked (as-built):**

| Ref | Footprint |
|-----|-----------|
| U1 NodeMCU-32S | ⚠ **still unassigned** — import a 38-pin FP; measure row pitch first (22.86 vs 25.4 mm) |
| U2, U3 PC817 | `Package_DIP:DIP-4_W7.62mm` |
| D1 MB10S | `Package_TO_SOT_SMD:TO-269AA` |
| C1 470 µF/63 V | `Capacitor_THT:CP_Radial_D14.0mm_P5.00mm` |
| C2 10 µF | `Capacitor_THT:CP_Radial_D5.0mm_P2.00mm` |
| R1–R4 | `Resistor_THT:R_Axial_DIN0207_L6.3mm_D2.5mm_P7.62mm` |
| U4 LM2596 module | `sln_geo_aux:LM2596_Module_THT` (custom, 17.75×40.5 mm hole pattern, 1.0 mm drill) |
| J1, J3 (2-pos) / J2 (3-pos) | `TerminalBlock:TerminalBlock_MaiXu_MX126-5.0-0{2,3}P…P5.00mm` (5.0 mm FP; 5.08 mm parts drop in) |

**Supersedes:** the 2026-07-08 "Power (resolved)" LM2596**HVS** call and its "plain LM2596 not safe" line; the Parts list's `LM2596HVS`.

**Next:** assign U1's footprint, then PCB layout (do with KiCad's board editor state in mind — footprint libs read on demand). Bench build still gated on the 5.00 V no-load set and the CT `calibrate_linear` pass.

---

## Amendment 2026-08-10 — BUILT AND BENCH-VALIDATED

Board milled in-house on the Tormach, assembled, reworked, and validated end to end. Full narrative in `docs/reference/geo-aux-heat-pcb-fab.md`; pad reference in `docs/reference/geo-aux-heat-u1-pad-map.md`.

**All stages pass:**

| Stage | Result |
|---|---|
| Power chain (24 VAC/DC → bridge → buck → 5 V) | ✅ stable |
| ESP boots, joins IoT VLAN 30, live in HA | ✅ |
| **CMD W1 / W2** optos, active-low, isolation barrier | ✅ both flip correctly |
| **CT** current sense | ✅ agrees with the shop Vue to **60 W on ~3600 W (1.7%)** |

**Two build faults found and fixed (both documented in the fab brief):**

1. **Pin-assignment erratum.** Schematic nets were attached by symbol pin *number* instead of pin *name*, so the optos landed on **P32/P33** (not P26/P27) and the CT on **P13** (not P34). Firmware now uses **GPIO32/GPIO33**; the CT was fixed with a trace cut at P13 + a jumper from **P34 → J3 pin 1**. V4 fixes the schematic (pads 29/30/24).
2. **The female headers were defective** — almost certainly solder wicking inside the housing, bridging contacts. Produced `invalid header` bootloops that looked exactly like a dead ESP32 flash chip and cost most of a day. **Fix: module soldered directly to the board.** V4: direct-solder or machined-pin sockets only, and no pads under the flash pins.

**CT calibration resolved:** measured **Emporia 50 A CT = 6.93 mV/A** (previously an unpublished unknown) → ~277 mV RMS at the AGL10A's 40 A, comfortably above the ADC noise floor. The original placeholder `0.266 -> 40 A` proved within 4% and is **left as-is**. Optional in-situ trim with the elements energized.

**Remaining before service:** install in the heat cabinet — insulation-window T-splice on BLK/WHT/RED, CT on a 240 V element leg, R+C for power — then arm the HA fault automations.

---

## Amendment 2026-09-18 — RF at the install location is marginal; wired uplink is open

**Measured:** ~**-80 dBm** at the air-handler location, *outside* the sheet-metal
heat compartment. That is not a Faraday-cage problem — the mechanical room itself
has no AP covering it. SLN's three APs are First Floor, Workout Room and Shop
(2× U6-Lite + U6-IW); none reach it. At -80 dBm an ESPHome device associates and
then reconnect-loops. This is a measurement, not yet a decision.

**Second fact, and it reframes the problem:** the **rack is in the mechanical
room**. The switch is feet from the sensor. Bridging that gap with a new AP would
be spending a radio to cross a distance a patch cable already covers.

**Option under consideration — wire it, drop Wi-Fi.** A **W5500 SPI Ethernet**
module (ESPHome `ethernet: type: W5500`, supported on the esp-idf framework this
board already uses) on the free VSPI pins — GPIO18 CLK / 23 MOSI / 19 MISO / 5 CS.
**No collision with the as-built GPIO32/33/34.** Rejected alternative: LAN8720
RMII — also pin-compatible with 32/33/34, but it needs 9 wires and a length-fussy
50 MHz clock versus W5500's 5, and the ESP32 module is **soldered directly to the
board** (no socket — see the 2026-08-10 amendment), so every added signal is a
flying lead. W5500 wins on wire count alone.

**Consequences if taken:**
- ESPHome does not permit `wifi:` and `ethernet:` in one config. Wired means
  wired only — no Wi-Fi fallback.
- That is an *upgrade* for this device specifically: its entire purpose is a
  trustworthy freeze/aux-heat alert while the house is unattended (Scott in FL).
  A cable to a switch in the same room beats a -80 dBm link as the path that
  alert depends on.
- **Keep the 24 VAC supply.** It is built and validated, and powering from the
  air handler's own transformer means the ESP goes unavailable when control power
  dies — which HA sees, and which is itself signal. Do not replace it with a PoE
  splitter.
- Best folded into a **rev-2 board**, which is wanted anyway: rev-1 carries the
  pin erratum as a trace cut plus a P34→J3 jumper.

**Open:** not chosen yet. The competing option is an AP in the mechanical room —
but that should be justified on its own merits (phone coverage while working on
equipment down there, future wireless sensors), not charged to this project.

**If an AP is bought, the nanoHD at ~$62 is the right one** — corrected
2026-09-18 after checking live prices. The earlier "buy a U6-Lite to match the
fleet" line here was wrong on both counts: **the U6 Lite is delisted** (U6 Pro
sold out, the U6 line is winding down), and the nearest current equivalent is
the **U7 Lite at $99** (verified at store.ui.com 2026-09-18), not less than the
nanoHD. Fleet consistency was worth having when it looked free; it is not worth
$37 on a utility AP whose only permanent client is a 2.4 GHz 1x1 ESP32. The
nanoHD's real drawbacks stand — Wi-Fi 5, discontinued, oldest radio on the
network and first in line for a support drop — they just do not cost $37 here.
If the $62 unit is used rather than new-old-stock, factory-reset before
adoption.
