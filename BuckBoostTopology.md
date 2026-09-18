# LT8390 Buck-Boost Converter — Design & Connection Guide

### Note:
AI-generated topology, should only be used as a reference guide. There are mistakes below.

**Sheet:** `CameraPowerAdapter.kicad_sch` (page 1 of the project) — the "buck-boost" stage
**Role in the system:** takes the selected 11–36 V battery rail from the LTC4417 priority power-mux sheet and produces a jumper-selectable **12.0 V or 24.0 V** output at up to **8 A** design limit.
**Companion sheet (do not mix up):** `priority-mux.kicad_sch` (LTC4417 two-input priority mux). Its output rail is **net `VOUT`** — that is *this* sheet's input.

This document is written so the whole stage can be rebuilt from a blank canvas. It is the "source of truth" for *what* connects to *what* and *why*.

---

## 0. Reading conventions

| Term | Meaning |
|---|---|
| `VOUT` | **Input rail of this sheet** = selected battery input, 11–36 V, from the LTC4417 sheet. (Confusing name, but it is the mux output net.) |
| `VOUT_REG` | The converter's own output node before the output current-sense resistor (`R11`). This is what the IC's `VOUT` pin monitors. |
| `OutPos` | Final regulated output after `R11` — where the board output terminals, feedback divider and output caps live. |
| `GND` | Common ground. |
| `SW1` / `SW2` | Buck-side / boost-side switching nodes. |
| `TG1/BG1/TG2/BG2` | Gate drive nets for the four power MOSFETs. |

Component references below are the ones used on the sheet (`R10…R34`, `C10…C27`, `M1…M4`, `Q10…Q15`, `U10`, `TH1`, `D10…D13`, `J10`). They deliberately start at 10 to avoid colliding with the LTC4417 sheet (`R1–R7`, `C1–C7`, `Q1–Q2`, `U2`).

---

## 1. Architecture at a glance

```
  VOUT (11-36V)                                                   OutPos (12V/24V, 8A)
      │                                                                │
      ├─ C12..C16 (input ceramics)                                     ├─ C17..C21 (output ceramics + bulk)
      │                                                                │
      ▼                                                                ▲
   ┌──────┐   SW1        R10 (2mΩ)        L1 (10µH)      SW2    ┌──────┐   VOUT_REG    R11 (12mΩ)
   │  M1  ├───┬──────────[sense]───────────(((((──────────┬─────┤  M3  ├──────┬────────[sense]──────┐
   └──────┘   │            ▲   ▲                          │     └──────┘      │                    │
   (buck HS)  │           LSP LSN                        │     (boost HS)     │                    │
              │                                          │                    │                    │
   ┌──────┐   │                                          │     ┌──────┐       │              ┌─────┴─────┐
   │  M2  ├───┘                                          └─────┤  M4  │       │              │ C22 (1µF) │
   └──────┘                                                     └──────┘       │              └───────────┘
   (buck LS)                                                    (boost LS)     │
      │                                                             │          │
     GND                                                           GND        VOUT pin 24
```

- **M1/M2 = buck leg** (active when VIN > VOUT), **M3/M4 = boost leg** (active when VIN < VOUT). The LT8390 runs all four in sequence and slides between regions automatically; around VIN ≈ VOUT it operates in "buck-boost overlap" mode — the least efficient region and the reason this board is thermally stressed at 12 V out from a 14.4 V battery.
- The **inductor current** is sensed across `R10` (LSP/LSN) → sets the cycle-by-cycle peak current limit.
- The **output current** is sensed across `R11` (ISP/ISN) → sets the output current limit (constant-current loop).
- `U1` is the LT8390. On this sheet it is drawn as a **two-unit symbol** (unit A = control pins, unit B = power pins) purely for readability; a blank-canvas rebuild can use a single 29-pin symbol.

---

## 2. Bill of materials (as used) and what each part is for

### 2.1 Semiconductors

| Ref | Part | Package | LCSC | Purpose / why this part |
|---|---|---|---|---|
| `U1` | **LT8390EFE#PBF** | TSSOP-28 + EP | — | 4-switch synchronous buck-boost controller, 4–60 V, 150–650 kHz, internal 5 V gate-drive rail (INTVCC), PGOOD, spread spectrum. |
| `M1` | **NCEP40T11G** | DFN 5×6 (8-lead) | C182465 | Buck high-side NMOS. 40 V, **3.9 mΩ @ Vgs = 4.5 V**, 110 A. Must be **logic-level** because the LT8390 gate drive is only 5 V. |
| `M2` | NCEP40T11G | DFN 5×6 | C182465 | Buck low-side NMOS (same part — keeps the BOM short and both see ≤ 36 V). |
| `M3` | NCEP40T11G | DFN 5×6 | C182465 | Boost high-side NMOS (sees VOUT ≤ 24 V + margin; 40 V gives headroom). |
| `M4` | NCEP40T11G | DFN 5×6 | C182465 | Boost low-side NMOS. |
| `U10` | **TL431AQDBZR** | SOT-23-3 | C105255 | Precision shunt reference used as the **over-temperature comparator** (Vref = 2.495 V, ±1 %). |
| `Q10` | **BC807-40** (PNP) | SOT-23 | C181152 | Over-temp **trigger** transistor (converts the TL431's "cathode low" into base current for the latch). |
| `Q11` | BC807-40 (PNP) | SOT-23 | C181152 | **SCR latch, PNP half** — holds the latch ON without any IC power. |
| `Q12` | **BC817-40** (NPN) | SOT-23 | C181151 | **SCR latch, NPN half** — pulls the EN node low when latched. |
| `Q13` | BC817-40 (NPN) | SOT-23 | C181151 | Latch **output** — pulls `EN/UVLO` to ground and keeps it there. |
| `Q14` | BC817-40 (NPN) | SOT-23 | C181151 | Red fault **LED sink**. |
| `Q15` | BC807-40 (PNP) | SOT-23 | C181152 | **PGOOD inverter** (PGOOD is open-drain pulling low; Q15 turns that into "drive Q14"). |
| `D10` | **LTST-C191KGKT** | 0603 LED | C125098 | Green **Power** LED (output present). |
| `D11` | **LTST-C191KRKT** | 0603 LED | C125099 | Red **Fault** LED. |
| `D12` | **BZX84C5V1** | SOT-23 | C2858518 | 5.1 V zener — creates the cheap, stable "VZ rail" that powers the NTC divider and the TL431 cathode so the trip point does not move with battery voltage. |
| `D13` | **1N4148W** | SOD-123 | — | Isolation diode: lets the latch light the fault LED but **blocks the PGOOD path from back-feeding into the SCR trigger** (otherwise every PGOOD glitch would latch the converter off). |
| `TH1` | **CMFA103J3500HANT** | 0603 NTC | C107464 | 10 kΩ B = 3500 K board-temperature sensor, mounted near the MOSFETs/inductor. |
| `L1` | **BMMN00171770100MX2** | 17.6 × 17.2 mm SMD | C5252365 | 10 µH, 10 mΩ DCR, 18 A sat, 14 A Irms shielded inductor. |

### 2.2 Passives

| Ref | Value | Package | Why |
|---|---|---|---|
| `R10` | **2 mΩ, 2 W, 1 %** (RLP25FEER002, C160590) | 2512 | Inductor current sense. With the LT8390's 50 mV peak threshold → **25 A peak limit**. Low-ESL/TCR part; Kelvin-connected (see §6). |
| `R11` | **12 mΩ, 3 W, 1 %** (LR123WF120MT4E, C19633815) | 2512 | Output current sense. With the 100 mV full-scale (CTRL = VREF) → **≈ 8.3 A output limit**. |
| `R12` | 226 k, 1 % | 0603 | `RT` — sets switching frequency to **200 kHz** (from the datasheet Table 1). |
| `R13` | **499 k**, 1 % | 0603 | `SS → VREF` — programs the **latch-off** output-short-circuit fault mode. |
| `R14` | 15 k | 0603 | `VC` compensation series resistor (start value; bench-tune). |
| `C26` | 15 nF | 0603 | `VC` compensation capacitor (series with R14). |
| `R15` | 365 k, 1 % | 0603 | `EN/UVLO` divider top (from VOUT). |
| `R16` | 56.2 k, 1 % | 0603 | `EN/UVLO` divider bottom → ≈ 10 V turn-on / ≈ 9.1 V turn-off. |
| `R17` | **0 Ω** | 0603 | Ties `CTRL` to `VREF` = full-scale 100 mV current-sense threshold. |
| `R18` | 110 k, 1 % | 0603 | FB divider top for **12 V** selection. |
| `R19` | 120 k, 1 % | 0603 | FB divider extra top resistor for **24 V** (110 k + 120 k = 230 k). |
| `R20` | **10.0 k, 1 %** | 0603 | FB divider bottom. VFB = 1.00 V ⇒ 12 V and 24 V exactly. |
| `J10` | 3-pin, 2.54 mm header | — | Voltage select: **1‑2 = 12 V, 2‑3 = 24 V**. |
| `R21` | 4.7 k | 0603 | Green LED series resistor (≈ 2.1 mA at 12 V, ≈ 4.7 mA at 24 V). |
| `R22` | 3.3 k | 0603 | Red LED series resistor (≈ 3 mA at 12 V, ≈ 10 mA at 36 V). |
| `R23` | **2.2 k, 1 W** | 2512 | Feeds the 5.1 V zener. At 36 V in: ≈ 14 mA ⇒ **0.44 W** — that is why 1 W, not 0603. |
| `R24` | 1.33 k, 1 % | 0603 | NTC divider bottom → sets the ≈ 85 °C trip point (see §7.6). |
| `R25` | 4.7 k | 0603 | TL431 cathode load / cathode current path from VIN (1.8–7 mA over 11–36 V). |
| `R26` | 47 k | 0603 | SCR "off" hold resistor — keeps SCR_A near VIN so the PNP half stays off when cold. |
| `R27` | 4.7 k | 0603 | Q11 (SCR PNP) emitter resistor — limits latch current. |
| `R28` | 4.7 k | 0603 | Q10 (trigger PNP) emitter resistor. |
| `R29` | 1 k | 0603 | Q10 base resistor from the TL431 cathode node. |
| `R30` | 1 k | 0603 | Q13 base resistor from the SCR node. |
| `R31` | 100 k | 0603 | Pull-down on the `FAULTB` node so the LED sink cannot float on. |
| `R32` | 4.7 k | 0603 | Series resistor from the SCR node to the isolation diode D13. |
| `R33` | 10 k | 0603 | PGOOD → Q15 base resistor. |
| `R34` | 1 k | 0603 | Q15 collector → `FAULTB` resistor. |

### 2.3 Capacitors

| Ref | Value | Package | Why / notes |
|---|---|---|---|
| `C10` | 100 nF, **≥ 25 V**, X7R | 0603 | Bootstrap `BST1 → SW1`. Must be **close to the IC pins** and have low ESR/ESL (it carries the gate-drive pulse). |
| `C11` | 100 nF, ≥ 25 V, X7R | 0603 | Bootstrap `BST2 → SW2`. |
| `C12…C15` | 4 × 10 µF, **50 V**, X7R/X7S | 1210 | Input high-frequency bulk right at the buck leg. 50 V rating is mandatory (36 V max input + spikes). Low ESR. |
| `C16` | 1 µF, 50 V | 0805 | `VIN` pin bypass (datasheet requires ≥ 1 µF). |
| `C17…C20` | 4 × 10 µF, 50 V, X7R | 1210 | Output high-frequency bulk (after the sense resistor, at `OutPos`). |
| `C21` | **180 µF, 35 V** polymer/aluminium-polymer | *footprint TBD* | Output bulk for camera-motor inrush. Use a **low-ESR** polymer type. (On the sheet this wears a placeholder 1210 footprint — assign the real can footprint in layout.) |
| `C22` | 1 µF, 50 V | 0805 | `VOUT` pin bypass (datasheet requires ≥ 1 µF) — on `VOUT_REG`. |
| `C23` | **4.7 µF, 25 V, X7R** | 0805 | `INTVCC` bypass (datasheet: minimum 4.7 µF ceramic, low ESR, close to the pin). |
| `C24` | **470 nF, 25 V** | 0603 | `VREF` bypass (datasheet: 0.47 µF). |
| `C25` | 100 nF, 50 V | 0603 | `SS` soft-start capacitor (12.5 µA internal charge current ⇒ ≈ 8 ms to 1 V). |
| `C27` | 100 nF, 50 V | 0603 | `ISMON` filter/monitor capacitor. |

> The input bulk (2 × 2200 µF) lives on the power-mux sheet, on the `VOUT` net, so this sheet only needs the local ceramics above.

---

## 3. LT8390 pin-by-pin connections (start here)

Pin numbers follow the TSSOP-28 (FE) package; pin 29 is the exposed pad.

| # | Pin | Connects to | Why / notes |
|---|---|---|---|
| **1** | `BG1` | **M2 gate** (net `BG1`) | Buck low-side gate drive (0→INTVCC). |
| **2** | `BST1` | **C10 pin 1** | Bootstrap floating supply for the buck high-side driver. C10's other pin goes to `SW1`. |
| **3** | `SW1` | Buck switch node: **M1 source (pins 1‑3)**, **M2 drain (pins 5‑9)**, **R10 pin 1**, **C10 pin 2** | Return for the bootstrap cap and the buck-side switch node. |
| **4** | `TG1` | **M1 gate** (net `TG1`) | Buck high-side gate drive (SW1 … BST1). |
| **5** | `LSP` | **Same node as `SW1`** | Positive Kelvin terminal of the inductor sense resistor. Electrically identical to SW1; keep it as a **separate trace** at PCB level. |
| **6** | `LSN` | **R10 pin 2** + **L1 pin 1** | Negative Kelvin terminal of the sense resistor = inductor side. |
| **7** | `VIN` | **Input rail `VOUT`** + C12‑C16 + C16 1 µF | Also tells the IC which operating region it is in. Bypass locally with ≥ 1 µF. |
| **8** | `INTVCC` | **C23 (4.7 µF)** and **SYNC/SPRD (pin 22)** | Internal 5 V rail; gate-drive supply. Not used to power anything else. |
| **9** | `EN/UVLO` | **R15/R16 divider node** + **Q13 collector** | Programmable UVLO, and the pin the over-temp latch pulls low. |
| **10** | `TEST` | **GND** | Factory test pin — **must be grounded** or the part will not run properly. |
| **11** | `LOADEN` | **VREF** | The high-side load-switch feature is unused; datasheet says tie it to VREF or INTVCC. |
| **12** | `VREF` | **C24 (470 nF)**, **R17 (0 Ω → CTRL)**, **R13 (499 k → SS)** | 2.000 V reference, 1 mA capable. |
| **13** | `CTRL` | **R17 → VREF** | At ≥ 1.35 V the ISP/ISN threshold is the full 100 mV. `CTRL < 0.3 V` stops switching (an alternative shutdown path). |
| **14** | `ISP` | **`VOUT_REG`** (converter side of R11) | Output current-sense positive terminal. Kelvin it. |
| **15** | `ISN` | **`OutPos`** (load side of R11) | Output current-sense negative terminal. Kelvin it. |
| **16** | `ISMON` | **C27 (100 nF) → GND** | Buffered current monitor: V(ISMON) = 10 × V(ISP−ISN) + 0.25 V. |
| **17** | `PGOOD` | **R33 (10 k) → Q15 base** | Open-drain, low when out of ±10 %. **Do not pull this above 6 V** — that is why the LED is driven through transistors instead of a VIN-referenced pull-up. |
| **18** | `SS` | **C25 (100 nF)** + **R13 (499 k → VREF)** | Soft-start cap. The 499 k resistor selects **latch-off** on output short (no resistor = hiccup, 100 k = keep-running). |
| **19** | `FB` | **R20 top / J10 pin 2** | 1.00 V regulation point. Also sets OVP (1.1 V) and short (0.25 V) thresholds. |
| **20** | `VC` | **R14 (15 k) → C26 (15 nF) → GND** | Error-amplifier output / loop compensation. |
| **21** | `RT` | **R12 (226 k) → GND** | Switching frequency = **200 kHz** (datasheet Table 1). |
| **22** | `SYNC/SPRD` | **INTVCC** | Ties spread-spectrum ON (±15 % triangle dither) for lower EMI. Tie to GND instead for fixed-frequency operation; drive with a clock to sync. |
| **23** | `LOADTG` | **No connect** | Load-switch P-gate driver unused. |
| **24** | `VOUT` | **`VOUT_REG`** + **C22 (1 µF)** | Tells the IC the output voltage/region; also the return rail for the LOADTG driver. Bypass with ≥ 1 µF. |
| **25** | `TG2` | **M3 gate** (net `TG2`) | Boost high-side gate drive. |
| **26** | `SW2` | Boost switch node: **M3 source (pins 1‑3)**, **M4 drain (pins 5‑9)**, **L1 pin 2**, **C11 pin 2** | |
| **27** | `BST2` | **C11 pin 1** | Bootstrap for the boost high-side driver (C11's other pin → `SW2`). |
| **28** | `BG2` | **M4 gate** (net `BG2`) | Boost low-side gate drive. |
| **29** | `GND` (EP) | **GND** | Exposed pad: solder to the ground plane and **add thermal vias** (this is the IC's main heat path). |

---

## 4. Block-by-block wiring (net by net)

### 4.1 Power train

**Buck leg (`M1`/`M2`)**
- `M1`: **drain (5‑9) → `VOUT`**, **source (1‑3) → `SW1`**, **gate (4) → `TG1`**.
- `M2`: **drain (5‑9) → `SW1`**, **source (1‑3) → `GND`**, **gate (4) → `BG1`**.
- Both semiconductors' drain pins are wired together as a small bus — these are multi-pin power packages; all of them must be connected.

**Inductor current sense (`R10`) + inductor (`L1`)**
- `R10` sits **between `SW1` and `L1`**: `R10 pin 1 → SW1` (= LSP side), `R10 pin 2 → LSN` (= `L1 pin 1`).
- `L1 pin 2 → SW2`.
- `U1/LSP` and `U1/LSN` are the 4-wire (Kelvin) sense terminals across `R10`.

**Boost leg (`M3`/`M4`)**
- `M3`: **drain (5‑9) → `VOUT_REG`**, **source (1‑3) → `SW2`**, **gate (4) → `TG2`**.
- `M4`: **drain (5‑9) → `SW2`**, **source (1‑3) → `GND`**, **gate (4) → `BG2`**.

**Bootstraps**
- `C10` across `BST1`↔`SW1`; `C11` across `BST2`↔`SW2`.

### 4.2 Output path and current sense

- `VOUT_REG` = `M3` drains + `U1/VOUT` (pin 24) + `C22` (1 µF) + `R11 pin 2` (ISP).
- `R11 pin 1` = `OutPos`: `C17…C21`, `R18` (FB top), `R21` (green LED), `U1/ISN` (pin 15).
- **Order matters:** the sense resistor `R11` sits **between the converter output and the output capacitors/load**, so the loop can limit current into a shorted load and the caps still buffer motor inrush.

### 4.3 Input

- `VOUT` rail: `U1/VIN` (7), `C12…C15` (4 × 10 µF), `C16` (1 µF), and the upper ends of `R15`, `R22`, `R23`, `R25`, `R26`, `R27`, `R28` (all the VIN-referenced pull-ups).
- `C12…C15` bottoms and `C16` bottom → `GND`.

### 4.4 Feedback divider and voltage select (`J10`)

```
OutPos ──[R18 110k]──┬──[R19 120k]── FB_B
                     │                │
                    FB_A ──── J10 ────┘        J10 pins: 1 = FB_A, 2 = FB, 3 = FB_B
                              │
                              FB ──[R20 10.0k]── GND  ── U1/FB (19)
```

- Jumper **1‑2** (12 V): top resistor = `R18` = 110 k ⇒ VOUT = 1.00 × (110 k + 10 k)/10 k = **12.0 V**.
- Jumper **2‑3** (24 V): top resistor = `R18 + R19` = 230 k ⇒ VOUT = 1.00 × (230 k + 10 k)/10 k = **24.0 V**.
- The divider senses the **load side** (`OutPos`), so the regulated voltage is what the camera actually sees.
- Using a jumper instead of a trim pot was deliberate: no drift, lower height, and it cannot be bumped inside a sealed enclosure.

### 4.5 Frequency, soft-start, compensation

- `U1/RT` (21) → `R12` 226 k → `GND` (200 kHz).
- `U1/SS` (18) → `C25` 100 nF → `GND`, and `R13` 499 k from `SS` to `VREF` (latch-off fault mode).
- `U1/VC` (20) → `R14` 15 k → `C26` 15 nF → `GND` (Type-2-ish compensation; **bench-tune**).
- `U1/INTVCC` (8) → `C23` 4.7 µF → `GND`.
- `U1/VREF` (12) → `C24` 470 nF → `GND`.

### 4.6 Enable / UVLO

- Divider `VOUT → R15 (365 k) → EN node → R16 (56.2 k) → GND`.
- This gives ≈ **10 V turn-on / ≈ 9.1 V turn-off** (the hysteresis comes from the IC's accurate 2.5 µA pull-down current acting on the 365 k ∥ 56.2 k source).
- It is deliberately set **below** the LTC4417's 11 V per-channel cutoff, so the mux decides input validity and the LT8390 threshold is only a secondary sanity limit.
- The latch output `Q13` also drives this node (see §4.8).

### 4.7 Current-limit programming

| Function | Parts | Effect |
|---|---|---|
| Peak inductor current limit | `R10` = 2 mΩ (LSP/LSN) | 50 mV / 2 mΩ = **25 A peak** |
| Output (average) current limit | `R11` = 12 mΩ (ISP/ISN) with `CTRL` = `VREF` (0 Ω) | 100 mV / 12 mΩ ≈ **8.3 A** |
| Monitor output | `ISMON` = 10 × V(ISP−ISN) + 0.25 V, filtered by C27 | Can be read externally |

### 4.8 Over-temperature latch (the project-specific block)

Goal: **when the board gets hot (~85 °C), shut the converter down and keep it down until the input power is physically cycled.** The IC's own thermal shutdown only protects the controller die (auto-resume), while the MOSFETs and inductor are the real heat sources — hence this external, latching circuit.

```
VOUT ──[R23 2.2k 1W]──┬── VZ (5.1V)
                     │
                   [D12 zener]        ← clamps a stable 5.1 V rail
                     │
                    GND

VZ ──[TH1 10k NTC]──┬── TH ─── TL431 REF (U10 pin 2)
                    │
                 [R24 1.33k]
                    │
                   GND

VOUT ──[R25 4.7k]── KC ─── TL431 cathode (U10 pin 1)
                          TL431 anode (U10 pin 3) ── GND

KC ──[R29 1k]── Q10 base   (Q10 = BC807 PNP, emitter via R28 4.7k to VOUT)
Q10 collector ── SCR_B

SCR (cross-coupled):
   Q11 (PNP, BC807): E → VOUT via R27 4.7k,  B → SCR_A,  C → SCR_B
   Q12 (NPN, BC817): E → GND,               B → SCR_B,  C → SCR_A
   R26 47k from VOUT → SCR_A   (holds the SCR off when cold)

Q13 (NPN, BC817): B → R30 1k → SCR_B,  C → EN/UVLO node,  E → GND
```

Operation:
1. Cold: `TH` ≈ 0.6 V < 2.495 V ⇒ TL431 off ⇒ `KC` sits near VIN ⇒ Q10 off ⇒ SCR off (`SCR_A` held high by R26) ⇒ Q13 off ⇒ EN set by R15/R16 ⇒ converter runs.
2. Board heats: NTC resistance falls, `TH` rises; at ≈ 85 °C `TH` crosses 2.495 V.
3. TL431 conducts → `KC` collapses to ≈ 2.5 V → Q10 turns on → injects base current into `SCR_B` → SCR latches → Q13 turns on hard → **EN pulled below 0.3 V** (IC in shutdown) → output collapses.
4. **Self-holding:** the SCR feeds itself from the raw input rail (VIN), not from the IC, so it stays latched even though INTVCC dies. It releases only when VIN is removed. The TL431 keeps firing (its divider is fed from the VZ rail, not from INTVCC), so the fault LED stays lit.

Why the zener rail: the NTC divider must give the *same* trip temperature at 11 V and at 36 V. Feeding it directly from the battery would move the trip point with state of charge.

### 4.9 Fault / power indication

```
Green:  OutPos ──[R21 4.7k]── D10 anode ── D10 cathode ── GND
        (lit whenever regulated output is present)

Red:    VOUT ──[R22 3.3k]── D11 anode ── D11 cathode ── (FAULT_LED node)
        FAULT_LED node ── Q14 collector      (Q14 = BC817 NPN, E → GND)

Q14 base (FAULTB node) is driven by two paths:
  a) PGOOD low (IC reports a fault while running):
        PGOOD ──[R33 10k]── Q15 base   (Q15 = BC807 PNP, E → INTVCC)
        Q15 collector ──[R34 1k]── FAULTB
  b) Over-temp latch engaged:
        SCR_B ──[R32 4.7k]── D13 anode ── D13 cathode ── FAULTB
  Plus R31 100k from FAULTB to GND (keeps the sink off when neither path is active).
```

Why the extra transistors and the diode:
- `PGOOD` may only be pulled up to **6 V max**, and when the latch pulls `EN` low the IC loses INTVCC — so a simple "VIN → resistor → LED → PGOOD" chain cannot work and cannot stay lit through the outage.
- The LED is therefore **fed from the raw input rail** (`VOUT`), and the *sink* is provided either by the PGOOD inverter (`Q15`/`Q14`) or by the latch (`SCR_B` → `D13` → `Q14`). That is what makes the red LED latch ON and stay lit for the whole shutdown.
- `D13` is essential: without it, the PGOOD path raising `FAULTB` would push current back into `SCR_B` and could trigger the SCR (a nuisance permanent shutdown).

### 4.10 Housekeeping

| Pin | Connection | Reason |
|---|---|---|
| `TEST` (10) | GND | Required. |
| `LOADEN` (11) | VREF | Load-switch feature unused. |
| `LOADTG` (23) | no connect | unused |
| `SYNC/SPRD` (22) | INTVCC | spread spectrum ON (EMI) |
| `VOUT` (24) | `VOUT_REG` + C22 | region detection + bypass |
| `INTVCC` (8) | only C23 (+ SYNC/SPRD) | 5 V gate drive rail |
| `VREF` (12) | C24, R17, R13 | 2 V reference |

---

## 5. Complete net list (verbatim, as verified)

```
BG1        : M2/4, U1/1
BG2        : M4/4, U1/28
BST1       : C10/1, U1/2
BST2       : C11/1, U1/27
CTRL       : R17/1, U1/13
DF_A       : D13/2, R32/2
EN         : Q13/3, R15/2, R16/1, U1/9
ENB        : Q13/1, R30/1
FAULTB     : D13/1, Q14/1, R31/1, R34/2
FAULT_LED  : D11/1, Q14/3
FB         : J10/2, R20/1, U1/19
FB_A       : J10/1, R18/2, R19/1
FB_B       : J10/3, R19/2
INTVCC     : C23/1, Q15/2, U1/22, U1/8
ISMON      : C27/1, U1/16
KC         : R25/2, R29/1, U10/1
LSN        : L1/1, R10/2, U1/6
OutPos     : C17/1, C18/1, C19/1, C20/1, C21/1, R11/1, R18/1, R21/1, U1/15
PGOOD      : R33/1, U1/17
PGOOD_B    : Q15/1, R33/2
Q10_B      : Q10/1, R29/2
Q10_E      : Q10/2, R28/2
Q11_E      : Q11/2, R27/2
Q15_C      : Q15/3, R34/1
RT         : R12/1, U1/21
SCR_A      : Q11/1, Q12/3, R26/2
SCR_B      : Q10/3, Q11/3, Q12/1, R30/2, R32/1
SS         : C25/1, R13/1, U1/18
SW1        : C10/2, M1/1, M1/2, M1/3, M2/5, M2/6, M2/7, M2/8, M2/9, R10/1, U1/3, U1/5
SW2        : C11/2, L1/2, M3/1, M3/2, M3/3, M4/5, M4/6, M4/7, M4/8, M4/9, U1/26
TG1        : M1/4, U1/4
TG2        : M3/4, U1/25
TH         : R24/1, TH1/2, U10/2
VC         : R14/1, U1/20
VOUT       : C12/1, C13/1, C14/1, C15/1, C16/1, M1/5..9, R15/1, R22/1, R23/1,
             R25/1, R26/1, R27/1, R28/1, U1/7
VOUT_REG   : C22/1, M3/5..9, R11/2, U1/14, U1/24
VREF       : C24/1, R13/2, R17/2, U1/11, U1/12
VZ         : D12/1, R23/2, TH1/1
GND        : C12/2 C13/2 C14/2 C15/2 C16/2 C17/2 C18/2 C19/2 C20/2 C21/2 C22/2
             C23/2 C24/2 C25/2 C26/2 C27/2 D10/1 D12/2 M2/1 M2/2 M2/3 M4/1 M4/2 M4/3
             Q12/2 Q13/2 Q14/2 R12/2 R16/2 R20/2 R24/2 R31/2 U1/10 U1/29 U10/3
internal   : R14/2 – C26/1        (compensation node)
internal   : R21/2 – D10/2        (green LED anode node)
internal   : R22/2 – D11/2        (red LED anode node)
no-connect : U1/23 (LOADTG)
```

---

## 6. Special considerations

**Sensing and layout-critical items**
1. **Kelvin (4-wire) sense on `R10` (LSP/LSN) and `R11` (ISP/ISN).** The sense traces must tap the *inside* of the pad pair and be routed as a tight differential pair, away from the switch node.
2. Use **low-ESL, low-TCR current-sense resistors**. `R10` dissipates up to 25 A² × 2 mΩ ≈ 1.25 W at the current limit (2 W part chosen); `R11` dissipates ≈ 8.3 A² × 12 mΩ ≈ 0.83 W (3 W part chosen).
3. **Keep the power loops tiny**: VIN → M1 → M2 → GND, and SW2 → M3/M4 → output. The high-di/dt loop area sets both EMI and switching loss.

**Capacitors**
4. **All ceramics in the power path must be low-ESR, high-ripple-rated X7R/X7S, 50 V.** Do not use Y5V or 25 V parts on the input or output.
5. `C23` (INTVCC, 4.7 µF) and the bootstrap caps `C10`/`C11` belong **right at the IC pins**; the gate-drive current is a fast pulse and trace inductance here degrades drive and efficiency.
6. `C21` (180 µF output bulk) must be a **low-ESR polymer** type; its job is camera-motor inrush and load transients, not ripple. `R11` upstream of it means the current limit sees inrush only briefly while the caps supply the motor.
7. `C25` (SS, 100 nF) set soft-start ≈ 8 ms; longer reduces inrush into the input mux, shorter improves camera start-up time. Treat as tunable.

**Gate drive and MOSFETs**
8. **Logic-level MOSFETs are mandatory** — the LT8390 driver swings only 0 → 5 V (INTVCC). `NCEP40T11G` is specified at Vgs = 4.5 V (3.9 mΩ). A standard 10 V-gate FET would run hot and may not fully enhance.
9. 40 V VDS parts are used for both legs: 36 V max input (buck side) and 24 V output (boost side) with margin. Higher-voltage parts would raise RDS(on) and cost.
10. The IC prevents **reverse inductor current** (no current from output back to input), which is what allows efficient light-load/discontinuous behaviour and safe parallel operation.

**Thermal (the primary design driver)**
11. Thermal vias from the IC's exposed pad and from each MOSFET's pad to the bottom copper pour; the pour terminates in the exposed 45 × 40 mm pad on the back for enclosure conduction. Low-profile finned heatsinks sit on `U1`, `M1–M4` (and the mux FETs on the other sheet).
12. **No fans** — all cooling is conductive/convective. Consequences: keep the switching frequency at the low end (200 kHz chosen; 150 kHz would be slightly cooler but needs a bigger inductor), and lean on the over-temp latch as the backstop rather than on airflow.
13. Mount `TH1` so it is thermally coupled to the MOSFETs/inductor copper — it must sense *board* temperature, not air.
14. Remember the intrinsic trap: **a 14.4 V battery into a 12 V output sits in buck-boost overlap mode**, where all four switches are active and efficiency dips. This is a per-watt thermal cost that the 24 V setting does not have.

**Fault logic**
15. **PGOOD must never be pulled above 6 V** (absolute max). This is why the red LED is fed from the input rail but is *sunk* by transistors, not by PGOOD.
16. The latch is **power-cycle-to-reset by design**. If that is ever undesirable, the alternative is to have the latch pull `CTRL` below 0.3 V instead of `EN/UVLO`: the IC then stops switching but stays biased, INTVCC survives, and PGOOD alone can drive the red LED directly (several fewer parts). That path was not taken because the requirement explicitly says "pull EN/UVLO low".
17. The IC's own **thermal shutdown auto-resumes**; the board-level latch does not. Expect both to exist and to behave differently in a thermal event (the latch wins because it holds EN low).

**Component ratings to respect**
18. `R23` (2.2 k zener feed) dissipates ≈ 0.44 W at VIN = 36 V — keep the 1 W / 2512 part.
19. The 0603 LEDs run 2–10 mA here; do not reduce `R22` below ≈ 2.2 k or the red LED will exceed its rating at 36 V.
20. `L1` is rated **18 A saturation**. The electrical current limit is 25 A peak (set by `R10`), so at the extreme corner (VIN ≈ 11 V, VOUT = 24 V, 8 A) the inductor will softly saturate *before* the current limit trips. At the nominal 14.4 V input the 8 A case peaks at ≈ 15 A, inside the rating with margin. If you want full 25 A capability you need a ≥ 25 A-sat inductor (e.g. Coilcraft SER2918H-class) — it does not fit the height budget, which is why it was not used.

---

## 7. Design calculations (so you can change values safely)

### 7.1 Switching frequency (`R12`)
Use the datasheet Table 1 rather than a formula. At `SYNC/SPRD` = GND:

| fOSC | RT (1 %) |
|---|---|
| 150 kHz | 309 k |
| **200 kHz** | **226 k** |
| 300 kHz | 140 k |
| 400 kHz | 100 k |
| 500 kHz | 75 k |
| 600 kHz | 59 k |
| 650 kHz | 51.1 k |

Lower f = less switching loss (cooler), larger inductor. Higher f = smaller magnetics.

### 7.2 Output voltage (`R18`, `R19`, `R20`)
`VOUT = 1.00 V × (Rtop + Rbot) / Rbot`
- 12 V: `Rtop = 110 k`, `Rbot = 10.0 k`
- 24 V: `Rtop = 230 k` (110 k + 120 k), `Rbot = 10.0 k`

OVP trips at ≈ 1.1 × VOUT; output-short threshold ≈ 0.25 × VOUT.

### 7.3 Inductor current sense (`R10`) and peak limit
Thresholds are **50 mV peak** in both buck and boost regions.
`RSENSE(BUCK) = 2 × 50 mV / (2 × IOUT(MAX) + ΔIL(BUCK))`
`RSENSE(BOOST) = 2 × 50 mV × VIN(MIN) / (2 × IOUT(MAX) × VOUT + ΔIL(BOOST) × VIN(MIN))`
with
`ΔIL(BUCK) = VOUT × (VIN(MAX) − VOUT) / (f × L × VIN(MAX))`
`ΔIL(BOOST) = VIN(MIN) × (VOUT − VIN(MIN)) / (f × L × VOUT)`

With L = 10 µH, f = 200 kHz, VIN = 11–36 V, VOUT = 24 V, IOUT = 8 A:
`ΔIL(BOOST) ≈ 3.0 A`, `RSENSE(BOOST) ≈ 2.6 mΩ` → **2 mΩ chosen** (≈ 25 A peak limit, ≈ 20 % margin).

### 7.4 Output current limit (`R11`)
`V(ISP−ISN) threshold = min(VCTRL − 0.25 V, 1 V) / 10`
With `CTRL` = `VREF` = 2 V → **100 mV** full scale.
`RIS = 100 mV / ILIM` → for 8.3 A: **12 mΩ** (choose 10 mΩ for 10 A, 15 mΩ for 6.7 A — the latter matches the datasheet's front-page 12 V/4 A design).

### 7.5 Inductor minimum values
`LBUCK > VOUT × (VIN(MAX) − VOUT) / (f × IOUT(MAX) × ΔIL% × VIN(MAX))`
`LBOOST > VIN(MIN)² × (VOUT − VIN(MIN)) / (f × IOUT(MAX) × ΔIL% × VOUT)`
Slope-compensation stability requires `L > 10 × VOUT × RSENSE / f` (≈ 2.4 µH here, easily met).
Also verify **saturation** > peak current, and DCR for conduction loss.

### 7.6 Over-temperature trip (`TH1`, `R24`)
`V(TH) = VZ × R24 / (R_NTC + R24) = 2.495 V` at the trip point, with `VZ = 5.1 V`:
`R_NTC(trip) = R24 × (5.1/2.495 − 1) = 1.044 × R24`
For `R24 = 1.33 k` ⇒ `R_NTC(trip) ≈ 1.39 k`.
For a 10 kΩ B = 3500 K thermistor, `R(T) = 10 k × exp(3500 × (1/T − 1/298.15))` ⇒ **T ≈ 84 °C**.

Change `R24` to move the trip: larger `R24` = lower trip temperature.

### 7.7 Enable / UVLO (`R15`, `R16`)
`VIN(turn-on) ≈ 1.233 V × (R15 + R16) / R16` (+ hysteresis from the 2.5 µA pin current)
`365 k / 56.2 k` ⇒ ≈ **10 V on / ≈ 9.1 V off**.
Keep this **below** the LTC4417's 11 V threshold so the mux arbitrates inputs first.

### 7.8 Latch / LED resistor sanity checks
- TL431 cathode current through `R25`: (11 − 2.5)/4.7 k ≈ 1.8 mA to (36 − 2.5)/4.7 k ≈ 7.1 mA — above the 1 mA minimum, below the 100 mA maximum.
- Zener current through `R23` at 36 V: (36 − 5.1)/2.2 k ≈ 14 mA; dissipation ≈ 0.44 W (1 W part).
- Red LED at 36 V: (36 − 2 − 0.1)/3.3 k ≈ 10 mA; at 12 V ≈ 3 mA.
- Green LED at 24 V ≈ 4.7 mA; at 12 V ≈ 2.1 mA.

---

## 8. Blank-canvas build order

1. **Place the IC** (`U1`). Decide up front whether you draw it as one 29-pin symbol or two units (power pins / control pins) — the pin *connections* below are identical either way.
2. **Ground the required pins**: `TEST` (10) → GND, `GND` (29) → GND.
3. **Power pins**: `VIN` (7) → input rail; `INTVCC` (8) + `C23` 4.7 µF; `VOUT` (24) → `VOUT_REG` + `C22` 1 µF; `VREF` (12) + `C24` 470 nF.
4. **Power train**: `M1`/`M2` (buck leg) and `M3`/`M4` (boost leg), then `R10` + `L1` between the legs. Add bootstrap caps `C10`/`C11`.
5. **Wire gate nets** `TG1`, `BG1`, `TG2`, `BG2` from the IC to the four gates.
6. **Output current sense**: `R11` from `VOUT_REG` to the output node `OutPos`; `ISP`/`ISN` across it.
7. **Input/output capacitor banks**: `C12…C16` on the input; `C17…C21` + `C22` on the output.
8. **Current-limit programming**: `R17` 0 Ω from `CTRL` to `VREF`.
9. **Frequency**: `R12` 226 k from `RT` to GND.
10. **Soft-start + fault mode**: `C25` from `SS` to GND, `R13` 499 k from `SS` to `VREF`.
11. **Compensation**: `R14` + `C26` from `VC` to GND.
12. **Feedback divider + jumper**: `R18`, `R19`, `R20`, `J10` as in §4.4, `FB` back to the IC.
13. **Enable divider**: `R15`/`R16` from the input rail to `EN/UVLO`.
14. **Housekeeping**: `LOADEN` → `VREF`; `SYNC/SPRD` → `INTVCC`; `LOADTG` → no-connect flag.
15. **Over-temp latch**: build the NTC/TL431/SCR chain of §4.8 and connect its output transistor (`Q13`) collector to the `EN/UVLO` node.
16. **Fault LED logic**: `R33`, `Q15`, `R34`, `R31`, `R32`, `D13`, `Q14`, `R22`, `D11` per §4.9; green `R21` + `D10` on `OutPos`.
17. **Re-check** every pin of §3 against your drawing, then run ERC. Expect only two benign warning classes if you use EasyEDA-imported symbols: "pin type unspecified" (fix by setting the imported pins to `passive`) and off-grid pins (snap symbol origins to the 1.27 mm grid).

---

## 9. Design decisions and their rationale (for review)

| Decision | Rationale |
|---|---|
| LT8390 (vs another buck-boost) | 4-switch synchronous, 4–60 V, integrated bootstrap diodes, PGOOD, spread spectrum, well-documented reference designs; the LTC3789-style alternative was rejected because its valley-current scheme and narrower input range fit worse. |
| 200 kHz | Compromise between switching loss (thermal priority) and inductor size/height. 150 kHz would be marginally cooler but needs ~15 µH. |
| 10 µH inductor, 18 A sat | Keeps ripple ≈ 25–30 % and fits the height budget. Full 25 A peak capability does not fit (see §6.20). |
| Jumper presets instead of a trim pot | No drift, cannot be bumped, lower height, smaller footprint. |
| Feedback divider on the load side | Regulates what the camera sees; load drop across R11 is included. |
| `CTRL` → `VREF` (0 Ω) | Maximum (100 mV) output-current threshold; `R11` then defines the actual limit. |
| 499 k `SS`→`VREF` | Latch-off on output short — consistent with the "no auto-recovery on a real fault" philosophy. |
| Spread spectrum enabled | Lower EMI in a film-production environment; no downside for this load. |
| `LOADEN` → `VREF`, `LOADTG` unused | No load switch is needed; the camera is switched by its own power input. |
| Over-temp latch pulls **EN/UVLO** (as specified) | Assured complete shutdown; the cost is the extra LED-steering transistors (§4.9). |
| Zener-fed NTC divider | Trip temperature independent of battery voltage, and alive after the latch engages. |
| Board-level latch in addition to the IC's thermal shutdown | The die protects itself only; the MOSFETs and inductor are the real heat sources. |

---

## 10. Known limitations / follow-ups

1. **Inductor saturation vs current limit** — 18 A sat against a 25 A peak limit; only matters at the unrealistic 11 V-in / 24 V-out / 8 A corner.
2. **Loop compensation values** (`R14`/`C26`) are starting values from ADI's high-power demo; verify with a load transient and adjust.
3. **`C21` footprint** — 180 µF/35 V polymer needs its real can footprint in layout (the schematic uses a 1210 placeholder).
4. **Kelvin sense and thermal vias** are layout-phase requirements, not schematic items — do not forget them.
5. **`R23` runs warm** (≈ 0.44 W at 36 V). Keep it 1 W and away from the NTC.
6. If you ever change `R11`, re-derive the LED/limit numbers in §7 and re-check the 8 A target.
7. The red LED is only latched for the **over-temp** case by design; the PGOOD path is a live indication while the IC is running. Both are lit from the input rail so a latched fault stays visible.
