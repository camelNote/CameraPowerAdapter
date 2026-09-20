# Schematic BOM — LT8390 Buck-Boost Stage

Component list extracted from `BuckBoostTopology.md`. Reference designators match `PCB/CameraPowerAdapter.kicad_sch`. Values, packages and passives per that guide; "Status" reflects the sheet as of 2026-09-19.

> **Update 2026-09-19:** all listed passives, `D5`/`D6` and `U3` are now placed. The deferred actives (`Q7`–`Q11`, `D1`–`D4`, `J1`, `TH1`) are still unplaced — see **Not complete** in `BuckBoostTopology.md`. The Status column below was written before that placement pass, so treat the deferred rows as authoritative and the passive rows as now `placed`.

**Status legend:** `placed` = already in the schematic · `to add` = prescribed by the guide, not yet placed · `optional` = fit only if needed · `DNP` = do not populate.

---

## Semiconductors — ICs

| Ref | Part | Package | Purpose | Status |
|---|---|---|---|---|
| U1 | LT8390 | TSSOP-28 + exposed pad 29 | 4-switch buck-boost controller | placed |
| U3 | TL431 (B-grade preferred) | SOT-23-3 | Over-temp comparator (check pinout, §4.4) | to add |

## Semiconductors — MOSFETs

| Ref | Part | Role | Package | Status |
|---|---|---|---|---|
| Q3 | NCEP40T11G | Boost high-side (TG2) | DFN 5×6 | placed |
| Q4 | NCEP40T11G | Buck low-side (BG1) | DFN 5×6 | placed |
| Q5 | NCEP40T11G | Boost low-side (BG2) | DFN 5×6 | placed |
| Q6 | NCEP40T11G | Buck high-side (TG1) | DFN 5×6 | placed |
| Q7 | BC807 | Over-temp latch trigger (PNP) | SOT-23 | to add |
| Q8 | BC807 | SCR equivalent, PNP half | SOT-23 | to add |
| Q9 | BC817 | SCR equivalent, NPN half | SOT-23 | to add |
| Q10 | BC817 | Latch shutdown, pulls EN/UVLO low | SOT-23 | to add |
| Q11 | BC817 | Red "output fault" LED driver | SOT-23 | to add |

## Semiconductors — Diodes & LEDs

| Ref | Part / value | Purpose | Notes | Status |
|---|---|---|---|---|
| D1 | Green LED | Output-OK indicator | R18 series | to add |
| D2 | Red LED | Output-fault indicator (PGOOD) | R19 series | to add |
| D3 | BZX84C5V1 | 5.1 V zener for VZ rail | fed by R20 | to add |
| D4 | Red LED | Over-temp latch indicator | R29 series, feeds off SCR_A | to add |
| D5 | Schottky | Dead-time / reverse-recovery across Q4 | tight against Q4 | optional |
| D6 | Schottky | Dead-time / reverse-recovery across Q3 | tight against Q3 | optional |

## Magnetics

| Ref | Value | Purpose | Notes | Status |
|---|---|---|---|---|
| L1 | 10 µH, ≥18 A sat | Main inductor | corner peak ≈19 A (accepted, §9) | placed |

## Resistors

| Ref | Value | Purpose | Package / notes | Status |
|---|---|---|---|---|
| R5 | 2 mΩ | Inductor current sense (SW1↔L1) | low-ESL; Kelvin to LSP/LSN | placed (no value) |
| R8 | 365 k | UVLO divider top | ~10.2 V turn-on | placed |
| R9 | 56.2 k | UVLO divider bottom | ~9.1 V turn-off | placed |
| R10 | 499 k | SS→VREF, selects latch-off | SS fault mode | placed |
| R11 | 12 mΩ | Output current sense | ≈8.3 A CC limit | to add |
| R12 | 226 k | RT → GND, 200 kHz | Table 1 | to add |
| R13 | 15 k | VC compensation (series R) | bench-tune | to add |
| R15 | 110 k | Feedback top (12 V) | OutPos→FB_A | to add |
| R16 | 120 k | Feedback, shunted by J1 | removed → 24 V | to add |
| R17 | 10 k | Feedback bottom | FB→GND | to add |
| R18 | 4.7 k | Green LED series | **1206** (0805 runs at 82%) | to add |
| R19 | 6.8 k | Red "output fault" LED series | **1206 min, 2010 preferred** | to add |
| R20 | 2.2 k, 1 W | VZ zener feed from VOUT | 0.43 W at 36 V | to add |
| R21 | 1.33 k | NTC divider bottom | sets ~85 °C trip | to add |
| R22 | 4.7 k | TL431 cathode | **2512 or 2×0603** | to add |
| R23 | 47 k | SCR_A hold-off (VOUT→SCR_A) | keeps SCR off when cold | to add |
| R24 | 4.7 k | Q7 emitter | **2010/2512 or 2× series** | to add |
| R25 | 4.7 k | Q8 emitter | **2010/2512 or 2× series** | to add |
| R26 | 1 k | Q7 base | — | to add |
| R27 | 1 k | Q10 base | — | to add |
| R29 | 6.8 k | Red "over-temp" LED series | **1206 min, 2010 preferred** | to add |
| R30 | 10 k | PGOOD pull-up → INTVCC | ≤5 V | to add |
| R31 | 1 k | Q11 base | — | to add |

## Capacitors

| Ref | Value | Purpose | Notes | Status |
|---|---|---|---|---|
| C8 | 1 µF | VIN pin local bypass | close to Q6/U1 | placed |
| C9 | 100 nF | Soft-start | ≈8 ms; ≥5–10× C25 | placed (no value) |
| C10 | 100 nF X7R | BST1 bootstrap | ≥100× top-FET Qg | placed (no value) |
| C11–C14 | 4× 10 µF, 50 V | Input caps | X7R/X7S, **1210** — sheet shows "100n", fix | placed (wrong value) |
| C15 | 4.7 µF | INTVCC bypass | **miswired** — see §8 | placed |
| C16 | 470 nF | VREF bypass | — | placed (no value) |
| C17 | 100 nF | ISMON filter | — | placed (no value) |
| C18 | 100 nF X7R | BST2 bootstrap | ≥100× top-FET Qg | to add |
| C19–C22 | 4× 10 µF, 50 V | Output bulk | X7R/X7S | to add |
| C23 | 180 µF | Output polymer | low-ESR | to add |
| C24 | 1 µF | VOUT pin bypass / region detect | — | to add |
| C25 | 15 nF | VC compensation (series C) | bench-tune | to add |
| C26 | TBD | ISP/ISN sense filter | optional (Figure 14b case) | optional |
| C27 | TBD | LSP/LSN sense filter | optional, near IC | optional |
| *(new ref)* | 1 µF | Extra INTVCC decoupling | immediately at pin/PGND | to add |

## Connectors & Thermistor

| Ref | Value | Purpose | Notes | Status |
|---|---|---|---|---|
| J1 | 2-pin shunt | Voltage select across R16 | fitted = 12 V, removed = 24 V | to add |
| TH1 | 10 k NTC, B = 3500 K | Over-temp sensor | mount on FET/inductor copper | to add |

## Retired / Do Not Populate

Not in the final circuit; references freed or reserved.

| Ref | Was | Why retired |
|---|---|---|
| Q12 | PGOOD inverter (PNP) | PGOOD is low = good; replaced by direct pull-up + Q11 (§7.2) |
| R14 | 0 Ω CTRL→VREF | CTRL ties straight to VREF |
| R28 | 100 k FAULTB→GND | FAULTB node eliminated |
| R32 | 100 k Q12 base→INTVCC | Q12 eliminated |

---

### Notes

- **Quantity:** R20 is the only 1 W part; R19/R29/R22/R24/R25 carry the 36 V derating burden.
- **Optional parts** (C26, C27, D5, D6) are not counted in the must-build set.
- Re-check references against the sheet after each placement batch — parts are being added incrementally.
