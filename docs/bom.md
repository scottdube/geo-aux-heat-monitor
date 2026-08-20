# BOM — geo-aux-heat-monitor

Derived from SLN ADR-013 "Revised BOM" (amendment 2026-06-15) plus amendment
2026-06-15b, which upgraded channel C to a dual tap (STAGED + COMMANDED) —
hence qty 2 on the opto front-end parts. The ADR's original single-opto line
predates that amendment.

## Core (required)

| Item | Qty | Function | Status | Notes |
|---|---|---|---|---|
| ESP32-WROOM DevKit | 1 | Host MCU | Have | ESP32 required (ADC1 + spare GPIO); ESP8266 rejected in ADR |
| Emporia Vue Gen 3 50 A split-core CT | 1 | Ch. B — kW sensing | Have (spare from shop Vue 16-kit) | 333 mV-class, burden built in (~20 Ω); ferrule leads |
| Resistor, 10 kΩ ¼ W | 2 | Ch. B — Vcc/2 bias divider | Have | |
| Capacitor, 10 µF | 1 | Ch. B — bias decoupling | Have | |
| Capacitor, 100 nF | 1 | Ch. B — bias decoupling | Have | |
| Screw terminal, 2-position | 1 | CT lead landing | Have | Ferrule-terminated leads — NOT a 2.5 mm jack (corrected in amendment) |
| Bridge rectifier (or 4× 1N400x) | 1 | 24 VAC → DC power | Bin (verify) | Skip entirely if tap turns out to be 24 VDC |
| Capacitor, ~470 µF / 50 V | 1 | Power filter | Bin (verify) | 24 VAC rectifies to ~34 V peak — 50 V rating required |
| Buck regulator module, ≥40 V-in → 5 V | 1 | ESP32 VIN supply | Bin (verify) | LM2596 (40 V) OK; MP1584 / R-78 class NOT OK (~28–32 V ceiling) |
| Printed enclosure + cable gland | 1 | Housing | To print | Fillets/chamfers per shop-device pattern |
| Hookup wire | — | — | Have | |

## Channel C — dual opto taps (optional; decide at build)

| Item | Qty | Function | Status | Notes |
|---|---|---|---|---|
| PC817 optocoupler | 2 | GPIO26 COMMANDED (coil), GPIO27 STAGED (NCM-300 W2/E) | Buy / check bin | Qty 2 per amendment 2026-06-15b |
| Small bridge or diode (1N4148/1N4007) | 2 | AC front-end for each opto | Bin | |
| Resistor, ~10–15 kΩ dropping | 2 | Opto LED current limit off 24 VAC | Bin | Read the contactor coil-voltage stamp FIRST — resize front-end if not 24 VAC |

## Channel A — hardware current switch (optional; likely dropped)

| Item | Qty | Function | Status | Notes |
|---|---|---|---|---|
| Split-core dry-contact current switch, ~50 A (Functional Devices RIBXGTA or NK Technologies AS-series) | 1 | GPIO25 no-cal on/off trigger | Buy only if kept | ADR: CT-only makes this ~$0; template on Ch. B amps replaces it |

## Dropped

- Hi-Link mains 5 V module (no mains conversion in the cabinet)
- 2.5 mm audio jack (CT leads are ferrules)

## Purchase decisions gating the BOM

1. Confirm 24 V**AC** (not DC) at the air-handler tap → bridge + filter needed or not.
2. Read strip-contactor coil stamp → confirms 24 VAC opto front-end sizing.
3. Keep or drop channel A → the only potentially non-trivial purchase.
4. Bin check: bridge, 470 µF/50 V cap, LM2596, PC817 ×2 — buy list is empty if all present.
