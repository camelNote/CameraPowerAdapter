# LT8390 Buck-Boost Stage — Build Guide

**Sheet:** `PCB/CameraPowerAdapter.kicad_sch`
**Input:** net `VOUT` — the LTC4417 priority-mux output, 11–36 V. (Badly named: `VOUT` is this stage's *input*.)
**Output:** jumper-selected **12 V or 24 V**, rated 6–8 A (real duty cycle ~2 A continuous, ~4 A bursts).
**Reference designators match the current schematic.** Parts already placed keep their refs; new parts continue the project numbering (R11+, C18+, Q7+, D1+, U3, J1, TH1).

> Verified against the LT8390 datasheet Rev. F — every equation, pin and table value re-checked in the 2026-09-19 audit. §7 lists all corrections (to the earlier draft **and** to the first rewrite); §8 lists the bugs still present in the schematic.

---

## Not complete

**Status 2026-09-19:** the power train, controller support, output stage, feedback/voltage-select resistors, compensation, sense filters, optional Schottkys (`D5`/`D6`) and `U3` (TL431) are placed and wired; values match this guide. ERC: 0 errors.

The analog support blocks are **not yet implemented**:

| Ref | Part | Block |
|---|---|---|
| Q7 | BC807 PNP | over-temp latch trigger |
| Q8 | BC807 PNP | over-temp latch (SCR pair) |
| Q9 | BC817 NPN | over-temp latch (SCR pair) |
| Q10 | BC817 NPN | latch shutdown → `EN/UVLO` |
| Q11 | BC817 NPN | output-fault LED driver |
| D1 | green LED | power indicator |
| D2 | red LED | output-fault indicator |
| D3 | BZX84C5V1 | `VZ` rail zener |
| D4 | red LED | over-temp indicator |
| J1 | 2-pin shunt | 12 V / 24 V select |
| TH1 | 10 k NTC | over-temp sensor |

The passives those blocks need **are** already placed (`R18`–`R27`, `R29`–`R31`), carrying stub net labels (`LED_*`, `VZ`, `KC`, `TH`, `SCR_A`, `SCR_B`, `Q7_*`, `Q8_E`, `Q10_B`, `Q11_B`) ready for the devices above to dock. `U3`'s feed/divider parts (`R20`–`R22`, `R26`) are in too — only `D3`, `TH1` and `Q7`–`Q10` are missing to complete the latch.

---

## 1. Topology at a glance

```
VOUT(in) ─┬─ Q6 [buck HS, TG1] ── SW1 ── R5 (2mΩ) ── L1 (10µH) ── SW2 ── Q3 [boost HS, TG2] ─┬─ VOUT_REG ── R11 (12mΩ) ── OutPos ─→ load
          │                        │                                                          │
          │                 Q4 [buck LS, BG1]                                          Q5 [boost LS, BG2]
          │                        │                                                          │
         GND                      GND                                                        GND
```

- Standard single-inductor **4-switch synchronous buck-boost**. Buck leg (`Q6`/`Q4`) runs when VIN > VOUT; boost leg (`Q3`/`Q5`) when VIN < VOUT; both run in the narrow overlap band around VIN ≈ VOUT — the hot, least-efficient corner (e.g. a 14.4 V battery into a 12 V output).
- **R5** (SW1↔L1) senses inductor current at `LSP`/`LSN` → cycle-by-cycle peak limit.
- **R11** (VOUT_REG↔OutPos) senses output current at `ISP`/`ISN` → constant-current limit and monitor.
- **U1** = LT8390 (drawn as a two-unit symbol: unit A = control, unit B = power).
- **Over-temperature latch:** NTC + TL431 + SCR that pulls `EN/UVLO` low and holds it until input power is cycled.

---

## 2. U1 pin map (TSSOP-28 + pad 29)

| # | Pin | Connect to | Note |
|---|---|---|---|
| 1 | BG1 | `Q4` gate | buck low-side drive |
| 2 | BST1 | `C10` → `SW1` | bootstrap for `Q6` |
| 3 | SW1 | `Q6` source, `Q4` drain, `R5`, `C10`, `LSP` | buck switch node |
| 4 | TG1 | `Q6` gate | buck high-side drive |
| 5 | LSP | `R5` SW1-side pad | Kelvin — separate trace from SW1 |
| 6 | LSN | `R5` L1-side pad + `L1` | Kelvin |
| 7 | VIN | `VOUT` rail + `C8`, `C11–C14`, and the VIN-referenced pull-ups (`R8`, `R20`, `R22`, `R23`, `R24`, `R25`) | ≥1 µF local bypass |
| 8 | INTVCC | `C15` + 1 µF; also `SYNC/SPRD` (22) | 5 V gate-drive rail |
| 9 | EN/UVLO | `R8`/`R9` divider node + `Q10` collector | UVLO + latch shut-off |
| 10 | TEST | GND | **required** |
| 11 | LOADEN | VREF | load-switch unused |
| 12 | VREF | `C16`, `R10` (499k→SS); `CTRL` and `LOADEN` tie here directly | 2.00 V ref, 1 mA capable |
| 13 | CTRL | ties to VREF | ≥1.35 V = full 100 mV sense threshold |
| 14 | ISP | `VOUT_REG` | Kelvin |
| 15 | ISN | `OutPos` | Kelvin |
| 16 | ISMON | `C17` → GND | 10·V(ISP−ISN) + 0.25 V |
| 17 | PGOOD | `R30` pull-up → INTVCC; node → `R31` → `Q11` base | open-drain, **low = output good** (FB within ±10%), high-Z = fault; pull up to INTVCC only, never >6 V |
| 18 | SS | `C9` + `R10` (499k) → VREF | `R10` selects latch-off |
| 19 | FB | `R17` bottom / `J1` | 1.00 V regulation |
| 20 | VC | `R13` → `C25` → GND | loop compensation |
| 21 | RT | `R12` (226k) → GND | 200 kHz |
| 22 | SYNC/SPRD | INTVCC | spread-spectrum ON |
| 23 | LOADTG | no-connect | unused |
| 24 | VOUT | `VOUT_REG` + `C24` | region detect + bypass |
| 25 | TG2 | `Q3` gate | boost high-side drive |
| 26 | SW2 | `Q3` source, `Q5` drain, `L1`, `C18` | boost switch node |
| 27 | BST2 | `C18` → `SW2` | bootstrap for `Q3` |
| 28 | BG2 | `Q5` gate | boost low-side drive |
| 29 | GND (pad) | GND plane + thermal vias | main heat path |

---

## 3. Wiring steps

**Power train**
1. `Q6` (buck HS): drain → `VOUT`, source → `SW1`, gate → `TG1`.
2. `Q4` (buck LS): drain → `SW1`, source → GND, gate → `BG1`.
3. `R5` between `SW1` and `L1`: `R5` SW1-side → `SW1`; `R5` other side → `L1`. Tie `U1/LSP` to the SW1-side pad and `U1/LSN` to the L1-side pad with **separate Kelvin traces**.
4. `Q3` (boost HS): drain → `VOUT_REG`, source → `SW2`, gate → `TG2`.
5. `Q5` (boost LS): drain → `SW2`, source → GND, gate → `BG2`.
6. Bootstrap: `C10` across `BST1`↔`SW1`; `C18` across `BST2`↔`SW2`. Both **100 nF X7R** (datasheet: 0.1–0.47 µF, ≥100× the top-FET gate charge). Place both hard against `U1`.
7. Input caps: `C11–C14` (4×10 µF 50 V) + `C8` (1 µF) from `VOUT` to GND, close to `Q6`/`U1`.
8. Output: `R11` from `VOUT_REG` to `OutPos`; `C24` (1 µF) on `VOUT_REG`; `C19–C22` (4×10 µF 50 V) + `C23` (180 µF polymer) on `OutPos` to GND.

**Control**
9. `R12` (226k) `RT`→GND; `C9` (100n) `SS`→GND; `R10` (499k) `SS`→`VREF` → **latch-off on output short** (no resistor = hiccup; 100k = keep-running).
10. `CTRL` and `LOADEN` both tie straight to the `VREF` pin (no jumper resistor needed); `C16` (470 n) `VREF`→GND; `C15` (4.7 µF) `INTVCC`→GND **plus a second 1 µF from `INTVCC` to GND right at the pin** (datasheet noise recommendation); `C24` and `C8` as above.
11. `R13` (15k) + `C25` (15 nF) in series from `VC` to GND (bench-tune). Datasheet rule: `C9` ≥ 5–10× `C25` — 100 nF / 15 nF = 6.7×, OK.
12. `C17` (100 n) `ISMON`→GND. Optional: `C26` across `ISP`/`ISN`, `C27` across `LSP`/`LSN`; Schottky `D5` across `Q4` and `D6` across **`Q3`** (the boost *high-side* — see §7).
13. `U1/LOADEN`→`VREF`; `U1/SYNC-SPRD`→`INTVCC`; `U1/TEST`→GND; `U1/LOADTG` no-connect.

**Feedback / voltage select** (see §4.1)
14. `R15` (110k) `OutPos`→`FB_A`; `R16` (120k) `FB_A`→`FB`; `R17` (10k) `FB`→GND; `U1/FB` at the `FB` node.
15. `J1` (2-pin) shunts `R16`: **fitted = 12 V, removed = 24 V.** A missing jumper must not disconnect the divider (see §7).

**UVLO / enable** (see §4.2)
16. `R8` (365k) `VOUT`→`EN`; `R9` (56.2k) `EN`→GND.

**Over-temp latch** (see §4.4)
17. Build the NTC/TL431/SCR chain of §4.4; `Q10` collector → the `EN` node.

**LEDs** (see §4.5)
18. Green `R18` + `D1` on `OutPos`; red `R19` + `D2` + `Q11` (output fault, PGOOD-driven) and red `R29` + `D4` (over-temp, driven directly off the latch) from `VOUT` (see §4.5).

---

## 4. Sub-circuits

### 4.1 Output voltage select
`VOUT = 1.00 V × (Rtop + Rbot)/Rbot`, `Rbot = R17 = 10.0 k`.
- 12 V: `Rtop = R15 = 110 k`.
- 24 V: `Rtop = R15 + R16 = 230 k` (jumper removed).
The divider senses `OutPos` (the load side), so the drop across `R11` is included.

### 4.2 Enable / UVLO
`VIN(turn-on) = 1.233 V × (R8+R9)/R9 + 2.5 µA × R8`, `VIN(turn-off) = 1.220 V × (R8+R9)/R9`.
With `R8 = 365k`, `R9 = 56.2k` → **≈10.2 V on / ≈9.1 V off**, deliberately below the LTC4417's 11 V so the mux arbitrates inputs first.

### 4.3 Frequency, soft-start, compensation
- `R12 = 226k` → **200 kHz** (datasheet Table 1). Lower f = cooler but larger inductor.
- Soft-start `t = 1 V × C9 / 12.5 µA` → `C9 = 100 nF` ≈ 8 ms.
- Compensation `R13`/`C25` are starting values only — verify with a load transient at both 12 V and 24 V.

### 4.4 Over-temperature latch
Trip when the NTC divider crosses the TL431's 2.495 V. Fed from a 5.1 V zener so the trip temperature does not move with battery voltage.

```
VOUT ──[R20 2.2k 1W]──┬── VZ
                      │
                    [D3 BZX84C5V1]
                      │
                     GND

VZ ──[TH1 10k NTC]──┬── TH ─── U3 REF   (U3 = TL431 SOT-23-3; TI DBZ is
                     │                    1=CATHODE, 2=REF, 3=ANODE — the KiCad
                   [R21 1.33k]            `TL431DBZ` symbol matches. The TL432 is
                     │                    pinned differently — don't substitute it.)
                    GND

VOUT ──[R22 4.7k]── KC ─── U3 CATHODE
                           U3 ANODE     ── GND

KC ──[R26 1k]── Q7 base        (Q7 = BC807 PNP; emitter via R24 4.7k to VOUT)
Q7 collector ── SCR_B

Q8 (PNP, BC807): E via R25 4.7k to VOUT, B = SCR_A, C = SCR_B
Q9 (NPN, BC817): E = GND,               B = SCR_B, C = SCR_A
R23 47k VOUT→SCR_A  (holds the SCR off when cold)

Q10 (NPN, BC817): B via R27 1k to SCR_B, C = EN/UVLO, E = GND
```

Operation: cold → TL431 off → `KC` high → `Q7` off → SCR off → `Q10` off → EN set by `R8`/`R9`. Hot → TL431 conducts → `KC` drops → `Q7` injects base current into `SCR_B` → SCR latches → `Q10` pulls EN low → converter shuts down. The latch pair (`Q8`/`Q9`) is fed from `VOUT`, not the IC, so it holds after INTVCC dies and only resets when input power is removed. When latched, **`SCR_B` sits at only ~0.85 V** (Q9's base-emitter) — never assume rail-level swing on that node (this is what broke the old LED steering, §7).

Trip ≈ **85 °C nominal, ±8 °C realistic** (zener ±5%, TL431 grade, NTC B-tolerance); use a B-grade TL431 (±0.5%) if the point matters. At 11 V input the zener runs near its knee (~0.8 mA spare) and the trip drifts a few degrees hotter — acceptable for a rough protector. While latched, `R24` and `R25` dissipate ~0.24 W / ~0.26 W **continuously** at 36 V input — same treatment as `R22`: 2010/2512 or two 4.7 k in series.

### 4.5 Fault / power LEDs
```
Green (output OK):   OutPos ──[R18 4.7k]── D1 ── GND

Red A (output fault): VOUT ──[R19 6.8k]── D2 ── Q11 collector   (Q11 = BC817 NPN, E = GND)
                       PGOOD ──[R30 10k]── INTVCC   (the only pull-up PGOOD gets)
                       PGOOD node ──[R31 1k]── Q11 base

Red B (over-temp):    VOUT ──[R29 6.8k]── D4 ── SCR_A   (direct — no transistor needed)
```

**PGOOD polarity matters:** the LT8390 pulls PGOOD **low when the output is good** (FB within ±10%) and releases it (high-Z) on fault. So on fault, `R30` pulls the PGOOD node — and Q11's base — up to INTVCC and red A lights; in normal operation PGOOD holds the node near 0 V and the LED is dark. No inverter transistor needed.

**Red B needs no drive transistor:** `SCR_A` idles at `VOUT` (pulled up by `R23`), so no current flows and the LED is dark. When the SCR latches, `Q9` pulls `SCR_A` low and the LED lights — its ~10 mA simply adds to Q9's collector load. The LED is a load, not a driver: it *cannot* pull `SCR_A` low by itself, so it can never false-trigger the latch (the reason the old `D4` steering diode existed — see §7). Fed from `VOUT`, red B stays lit after INTVCC collapses.

Diagnostic readout: green = output in regulation; red A = running but out of regulation (also covers short/hiccup); red B = over-temp latch. When the latch fires, the converter stops, INTVCC dies and red A stays dark — the two reds are unambiguous.

Resistor sizing (continuous 36 V): `R19`/`R29` at 6.8 kΩ drop ~34 V → **0.17 W** → 1206 minimum (68% of rating), 2010 preferred. 3.3 kΩ would be 0.35 W — only viable in a 2512. `R18` at 24 V out ≈ 0.10 W → use a **1206** (an 0805 runs at 82% continuously).

---

## 5. Key values and formulas

**Current limits**
- Inductor peak: `50 mV / R5` → `R5 = 2 mΩ` ⇒ **25 A peak**.
- Output CC: `I = min(VCTRL − 0.25 V, 1 V) / (10 · R11)`, `CTRL = VREF` ⇒ 100 mV full scale. `R11 = 12 mΩ` ⇒ **≈8.3 A**.
- Monitor: `V(ISMON) = 10·V(ISP−ISN) + 0.25 V`.

**Peak / RMS sizing** (datasheet equations)
- `ΔIL(BUCK) = VOUT·(VIN(MAX)−VOUT) / (f·L·VIN(MAX))`
- `ΔIL(BOOST) = VIN(MIN)·(VOUT−VIN(MIN)) / (f·L·VOUT)`
- `RSENSE(BUCK) = 2·50mV / (2·IOUT + ΔIL(BUCK))`
- `RSENSE(BOOST) = 2·50mV·VIN(MIN) / (2·IOUT·VOUT + ΔIL(BOOST)·VIN(MIN))`
- `LBUCK > VOUT·(VIN(MAX)−VOUT) / (f·IOUT·ΔIL%·VIN(MAX))`
- `LBOOST > VIN(MIN)²·(VOUT−VIN(MIN)) / (f·IOUT·ΔIL%·VOUT²)`  ← note the squared `VOUT`
- Slope-compensation stability: `L > 10·VOUT·RSENSE / f` (≈2.4 µH here).
- **RSENSE margin check** (datasheet: final value 20–30% *below* calculated): worst case (24 V out, 11 V in, 8 A) gives `RSENSE(BOOST) = 2.64 mΩ` → the chosen 2 mΩ carries **24% margin** ✓. Buck region is far less demanding (≈5 mΩ calculated).
- **Input-cap RMS current:** `I_RMS ≈ IOUT·√(D·(1−D))` (buck region), worst at `VIN = 2·VOUT` → ≈4 A at the 8 A burst rating (~1–2 A at the real 2–4 A duty), shared across `C11–C14` — keep them 1210 X7R/X7S.
- **INTVCC self-heating:** four gates × ~50 nC × 200 kHz ≈ 40–50 mA from the internal LDO → `P ≈ 36 V × 45 mA ≈ 1.6 W` inside the IC → roughly **+70 °C junction rise** at θJA ≈ 43 °C/W. Verify `TJ = TA + P·θJA` at max VIN and max ambient; if marginal, drop to 150 kHz or use lower-Qg FETs.

**Switching frequency (`R12`)** — use Table 1, `SYNC/SPRD` = GND:

| f | RT | f | RT |
|---|---|---|---|
| 150 kHz | 309 k | 400 kHz | 100 k |
| **200 kHz** | **226 k** | 500 kHz | 75 k |
| 300 kHz | 140 k | 600 kHz | 59 k |
| | | 650 kHz | 51.1 k |

**Over-temp trip** — `R_NTC(trip) = R21 × (VZ/2.495 − 1) = 1.044 × 1.33k ≈ 1.39 k` → **≈85 °C nominal** for a 10 kΩ / B = 3500 K NTC (±8 °C realistic — zener ±5% alone moves it ±3.5 °C). Increase `R21` to trip cooler.

---

## 6. Layout must-dos

1. **Kelvin-sense `R5` (LSP/LSN) and `R11` (ISP/ISN).** Tap the inside of each pad, route as a tight pair away from the switch nodes. Add the small filter caps (`C26`, `C27`) near the IC.
2. **Keep the power loops tiny:** VIN → `Q6` → `Q4` → GND and `SW2` → `Q3`/`Q5` → output. Loop area sets EMI and switching loss.
3. **All power-path ceramics low-ESR X7R/X7S, 50 V.** No Y5V, no 25 V parts on input/output.
4. **`C15` (INTVCC 4.7 µF) and the bootstraps `C10`/`C18` right at the IC pins**, plus the extra 1 µF INTVCC cap immediately against the pin/power-ground pair.
5. **Thermal:** vias from `U1`'s exposed pad and each MOSFET's pad into the bottom pour, terminating in the exposed rear pad for enclosure contact. Mount `TH1` on the MOSFET/inductor copper, not in free air.
6. **Logic-level MOSFETs are mandatory** — the gate drive is 0→5 V only. `NCEP40T11G` is characterised at Vgs = 4.5 V. Note its 40 V BVDSS on a 36 V rail leaves only ~11% headroom for switch-node ringing on the buck leg — check ringing on the bench, snubber if needed.
7. **Grounding (datasheet checklist):** dedicated ground-plane layer with *no signal traces*, as close as possible to the FET layer; small-signal grounds return to the exposed pad *from the bottom*, then tie to power GND near the sources of `Q4`/`Q5`; output-cap (−) terminals as close as possible to input-cap (−) terminals; flood all unused copper.
8. **Keep `SW1`, `SW2`, `BST1`, `BST2`, `TG1`, `TG2` (high-dV/dt) traces away from all small-signal nodes.** If fitted, `D5` sits tight against `Q4`; `D6` against `Q3`, grouped with `Q5` and the output caps in one compact area.

---

## 7. Corrections and audit log

### 7.1 vs the previous draft (verified)

1. **`LBOOST` equation** had the wrong denominator — it must be `VOUT²`, not `VOUT` (fixed in §5).
2. **`R19` red-LED resistor was 3.3 kΩ 0603.** At 36 V input it dissipates **≈0.35 W** — 3.5× an 0603's rating. Final fix: **6.8 kΩ in a 1206** (0.17 W, dimmer but safe) or 3.3 kΩ in a 2512 only.
3. **`R22` TL431 cathode resistor was 4.7 kΩ 0603.** At 36 V it dissipates **≈0.24 W**. Keep 4.7 kΩ (needed to feed `Q7`'s base current at low VIN) but use a **2512, or two 0603s in series** — a single 1206 runs at 98% of rating. Same treatment now applies to `R24`/`R25` (continuous while latched, §4.4).
4. **`R18` green-LED resistor** reaches ~0.10 W at 24 V — use a **1206** (an 0805 runs at 82% of rating, continuously).
5. **Output-sense filtering:** because `R11` sits between `C24` (decoupling) and `C19–C22` (bulk), the datasheet recommends an `R_F`/`C_F` filter across `ISP`/`ISN` (`C26` here).
6. **Optional Schottkys `D5`/`D6`** for dead-time/reverse-recovery: `D5` across `Q4` (switch B) and `D6` across **`Q3` (switch D, the boost *high-side*)** — *not* `Q5`; the datasheet pairs `D_D` with the boost top FET, grouped output-side.
7. **No-jumper runaway:** in the old 3-pin scheme, removing the jumper disconnected FB from the divider entirely → the output would run away. The rewrite uses a 2-pin shunt across `R16` so a missing jumper defaults to **12 V**.
8. **`Q12` (PGOOD inverter) base floated when PGOOD was high-Z** — added `R32`. *Superseded by §7.2 item 1: the inverter is gone entirely.*
9. **LED power was never checked** in the old §7.8 — only current. See items 2–4.

### 7.2 final audit (2026-09-19)

1. **PGOOD polarity was inverted.** The LT8390 pulls PGOOD **low when the output is GOOD** (high-Z on fault). The §4.5 inverter lit the red LED in normal operation and kept it dark on fault. Fixed: direct pull-up (`R30`) + NPN (`Q11`), no inverter — §4.5 rewritten. `Q12`, `R28`, `R32` are retired; refs freed.
2. **The latch could never light the LED.** Old `D4` needed ~1.4 V at the anode, but a latched `SCR_B` clamps at ~0.85 V (Q9's V_BE) — the diode never conducted. Fixed: red B (`R29`/`D4`) now hangs off `SCR_A`, which goes *low* when latched; a load cannot false-trigger the latch, so no steering diode is needed. Bonus: two reds now distinguish output fault from over-temp.
3. **TL431 vs TL432 SOT-23 pinout (corrected).** TI's **TL431** DBZ is 1=CATHODE, 2=REF, 3=ANODE — which the KiCad `TL431DBZ` symbol now placed matches. The **TL432** DBZ is pinned differently, so don't substitute it. (The first audit had TL431/TL432 swapped; corrected against TI SLVS543 Table 5-1.)
4. **Missing bootstrap values** — `C10`/`C18` now specified (100 nF X7R; datasheet 0.1–0.47 µF).
5. **Missing datasheet design rules, now checked/documented:** second 1 µF INTVCC cap; `C9` ≥ 5–10× `C25` (6.7×, OK); RSENSE 20–30% margin (24%, OK); input-cap RMS current (§5).
6. **New risk documented: INTVCC self-heating** — ~1.6 W in the IC at 36 V/200 kHz gate drive (§5). Verify TJ before committing to the enclosure.
7. **VZ rail near zener knee at 11 V input** — only ~0.8 mA spare at the trip point; trip drifts a few degrees hotter at minimum VIN (§4.4). Acceptable for a protector, but no longer claimed as exact.
8. **NCEP40T11G is 40 V BVDSS on a 36 V rail** — ~11% ringing headroom on the buck leg; bench-check, snubber if needed (§6.6).

---

## 8. Issues to fix in the current schematic

*Verified against the sheet's netlist, 2026-09-19.*

**Fixed:** `C15` rewired to INTVCC↔GND and `SYNC/SPRD` tied to INTVCC; `Q3`/`Q4`/`Q5`/`Q6` fully wired; `R5` = 2 mΩ, `C9`/`C10` = 100 n, `C11–C14` = 10 µF, `C16` = 470 n, `C17` = 100 n, `L1` = 10 µH; `U1` pins 14/15 (`ISP`/`ISN`), 20 (`VC`), 24 (`VOUT`) connected and 23 (`LOADTG`) no-connect; output stage, feedback divider, compensation, sense filters and `U3` placed (**0 ERC errors**).

**Still open:**
- The analog support blocks listed under **Not complete** above.
- **No footprints assigned** — every part still reads "(No footprint)". Assign the packages noted in §5/§7 (e.g. 1210 for `C11–C14`, 1206+ for `R19`/`R22`/`R29`, 1206 for `R18`, 1 W for `R20`).
- ~48 ERC **warnings** remain, all of the form `Unspecified ↔ …` from the EasyEDA symbols (`U1`, `Q3–Q6`) having every pin typed "unspecified". Cosmetic; clears if the symbol pins are retyped.

---

## 9. Design decisions

| Decision | Rationale |
|---|---|
| LT8390 4-switch buck-boost | 4–60 V, integrated bootstrap diodes, PGOOD, spread spectrum, well-documented reference designs. |
| 200 kHz | Thermal compromise; 150 kHz is marginally cooler but needs a bigger inductor. |
| 10 µH / 18 A-sat inductor | Fits the height budget. Note: at the corner (11 V in, 24 V out, 8 A) peak ≈19 A exceeds 18 A sat before the 25 A limit trips — a knowingly accepted limitation of the height target. |
| Jumper presets, not a trim pot | No drift, can't be bumped, lower height. |
| Spread spectrum ON | Lower EMI; no downside for this load. |
| `LOADEN`→VREF, `LOADTG` unused | No external load switch needed. |
| Latch pulls **EN/UVLO** | Assured full shutdown; the latch lights its own LED directly off `SCR_A` (§4.5). |
| Two red LEDs (fault / over-temp) | Cheaper and simpler than a steered single LED, and the two fault types are distinguishable; PGOOD is high-Z on fault, so a pull-up + NPN suffices. |
| Zener-fed NTC divider | Trip temperature roughly independent of battery voltage, alive after the latch engages (±8 °C, §4.4). |
