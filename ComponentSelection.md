# Component Selection Rationale — LT8390 Buck-Boost Stage

Why each component in `BuckBoostTopology.md` / `SchematicBOM.md` has the value it has, grouped by function.

**Page references** are exact pages of the **LT8390 datasheet, Rev. F (30-page edition)** — the same document as the local scan `Datasheets/LTC8390.pdf` (ADI file `8390f`). The Applications Information section spans **pp. 17–25**; that is where most component decisions come from. Other revisions paginate differently (e.g., the 32-page Rev. D shifts these pages by ~+1), so stick to the local 30-page copy when checking. Where a value comes from a different datasheet (MOSFET, TL431), it says so explicitly.

---

## 1. Controller

| Part # | Why this value was chosen | LT8390 datasheet page |
|---|---|---|
| U1 (LT8390EFE) | Only a 4-switch single-inductor controller regulates with VIN above, below **or equal to** VOUT — required because the 11–36 V mux output straddles both the 12 V and 24 V settings. Also needed: peak-buck/peak-boost current mode (clean region transitions), ±1.5 % VOUT accuracy, input/output current regulation (the CC limit), spread spectrum (EMI), integrated bootstrap diodes, PGOOD, and a latch-off fault mode (required by the over-temp shutdown scheme). TSSOP-28 FE chosen for the exposed-pad thermal path. | **p. 1** (Features/Description + 12 V/4 A typical application); **p. 2** (pin configuration, θJA = 30 °C/W, "exposed pad (pin 29) is GND, must be soldered to PCB") |

## 2. Power MOSFETs (all four switches)

| Part # | Why this value was chosen | LT8390 datasheet page |
|---|---|---|
| Q3, Q4, Q5, Q6 (NCEP40T11G) | The datasheet states the gate drive is the 5 V INTVCC rail, so **logic-level MOSFETs are mandatory**. The NCEP40T11G is characterized at VGS = 4.5 V (RDS(ON) ≈ 3.3 mΩ typ, from `Datasheets/NCEP40T11G.pdf`), is fully enhanced by the ~5 V drive, and its Qg ≈ 60 nC keeps the INTVCC budget workable. Its 40 V BVDSS on the 36 V rail leaves only ~11 % ringing headroom — a flagged, bench-verify risk (BuckBoostTopology §6.6), accepted for the low RDS(ON). | **p. 19** (Power MOSFET Selection: VBR(DSS), VGS(TH), RDS(ON), CRSS; "logic-level threshold MOSFETs must be used"); **p. 21** (INTVCC dissipation = VIN·IINTVCC, TJ = TA + P·θJA — the check that drives FET Qg and 200 kHz); **p. 25** (compact placement of switches with their caps/diodes) |

## 3. Inductor current sense (cycle-by-cycle peak limit)

| Part # | Why this value was chosen | LT8390 datasheet page |
|---|---|---|
| R5 (2 mΩ) | Peak current limit = 50 mV / RSENSE → **25 A typical**, above the ~19 A worst-case inductor peak (11 V in, 24 V out, 8 A). The datasheet equations give RSENSE(BOOST) ≈ 2.64 mΩ at that corner; choosing 2 mΩ leaves **24 % margin**, satisfying the "final value 20–30 % below calculated, low-ESL" rule. The 300 W reference design uses the same 2 mΩ (×2 parallel). | **p. 18** ("duty cycle independent maximum current sense thresholds (50 mV in peak-buck and 50 mV in peak-boost)"); **p. 19** (RSENSE(BUCK)/RSENSE(BOOST) equations + 20–30 % margin + low-ESL rule); **p. 9** (LSP/LSN pin functions — Kelvin connection); **p. 25** (Kelvin layout + sense-line routing); **p. 27** (300 W app: 2 mΩ ×2) |

## 4. Output current sense (constant-current limit + monitor)

| Part # | Why this value was chosen | LT8390 datasheet page |
|---|---|---|
| R11 (12 mΩ) | With CTRL ≥ 1.35 V the ISP/ISN threshold is fixed at 100 mV full scale, so IIS(MAX) = 100 mV / RIS = **8.33 A** — the top of the 6–8 A design rating. Keeps the sense drop at 0.1 W during 8 A bursts. Same scheme as the datasheet's apps (15 mΩ → 6.7 A on the 12 V/4 A; 6 mΩ×2 → 33 A on the 300 W). | **p. 21** (RIS in the output path, Kelvin-sensed by ISP/ISN; "tie CTRL higher than 1.35 V to get the full-scale 100 mV"); **p. 22** (IIS(MAX) = 100 mV/RIS); **p. 10** (CTRL and ISP/ISN pin functions); **p. 23** (ISMON = 10·V(ISP−ISN) + 0.25 V → clean 0.25–1.25 V monitor range); **p. 26–27** (RIS precedents) |

## 5. Main inductor

| Part # | Why this value was chosen | LT8390 datasheet page |
|---|---|---|
| L1 (10 µH, ≥18 A sat) | 10 µH at 200 kHz gives ≈3 A ripple at the worst corner and satisfies every datasheet minimum with margin: slope-compensation stability needs L > 10·VOUT·RSENSE/f ≈ 2.4 µH, and the LBUCK/LBOOST equations are met in both regions. Ferrite, low-DCR construction per the selection guidance. 18 A saturation vs ~19 A corner peak is the documented height-budget tradeoff (§9 of the build guide). | **p. 18** (Inductor Selection: ΔIL equations, LBUCK/LBOOST minimums, slope-compensation inequality, "low core loss, such as ferrite… low DC resistance") |

## 6. Switching frequency

| Part # | Why this value was chosen | LT8390 datasheet page |
|---|---|---|
| R12 (226 kΩ) | Table 1 maps 200 kHz ← **226 kΩ, 1 %**. 200 kHz is the thermal/inductor-size compromise: "selection of switching frequency is a tradeoff between efficiency and component size." Spread spectrum (SYNC/SPRD tied to INTVCC) then modulates ±15 % around 200 kHz for EMI, with no downside for this load. | **p. 17** (RT selection + Table 1 "Switching Frequency vs RT Value (1 % Resistor)" + Spread Spectrum Frequency Modulation section); **p. 5** (EC table: RT = 226 kΩ → 190/200/210 kHz); **p. 10** (SYNC/SPRD pin function: "tie to INTVCC for ±15 % triangle spread spectrum") |

## 7. Enable / UVLO divider

| Part # | Why this value was chosen | LT8390 datasheet page |
|---|---|---|
| R8 (365 kΩ), R9 (56.2 kΩ) | VIN(UVLO+) = 1.233 V·(R1+R2)/R2 + 2.5 µA·R1 ≈ **10.15 V on**; VIN(UVLO−) = 1.220 V·(R1+R2)/R2 ≈ **9.14 V off**. Deliberately below the LTC4417's 11 V floor so the priority mux arbitrates its inputs first. This exact divider (365 k/56.2 k) is used by the datasheet's own 9–36 V application. Figure 12 also shows the NMOS-pulls-EN/UVLO-low shutdown topology our over-temp latch (Q10) copies. | **p. 21** (UVLO programming formulas + Figure 12 external shutdown); **p. 3** (EC: EN/UVLO falling threshold 1.220 V, hysteresis current); **p. 9** ("force below 0.3 V to shut down"; "above 1.233 V for normal operation"); **p. 27** (300 W app: same 365 k/56.2 k) |

## 8. INTVCC rail bypass

| Part # | Why this value was chosen | LT8390 datasheet page |
|---|---|---|
| C15 (4.7 µF) | INTVCC regulator supplies ~110 mA gate-drive peaks and **must** be bypassed with a minimum 4.7 µF ceramic — value taken verbatim from both the pin function and the INTVCC Regulator section. (This is the cap that was miswired to SYNC/SPRD; now INTVCC↔GND.) | **p. 9** (pin function: "minimum 4.7 µF ceramic capacitor"); **p. 21** (INTVCC Regulator: 110 mA, bypass requirement, VIN·IINTVCC self-heating check) |
| C28 (1 µF) | The layout checklist recommends an **additional 1 µF immediately next to the INTVCC pin and power ground** to "improve noise performance substantially" — added on top of the 4.7 µF. | **p. 25** (PCB layout checklist: additional 1 µF at INTVCC pin) |

## 9. Bootstrap capacitors

| Part # | Why this value was chosen | LT8390 datasheet page |
|---|---|---|
| C10, C18 (100 nF, X7R) | Bootstrap caps must store ~**100× the gate charge** of the top switches (A and D); "in most applications a 0.1 µF to 0.47 µF, X5R or X7R… is adequate." 100 nF X7R sits at the recommended value for the NCEP40T11G's ~60 nC gates (≈6× the stored-charge requirement). Placed hard against U1. | **p. 21** (bootstrap paragraph: 100× gate charge, 0.1–0.47 µF X5R/X7R); **p. 9** (BST1 pin function — integrated bootstrap diode) |

## 10. Input rail: VIN bypass + input capacitors

| Part # | Why this value was chosen | LT8390 datasheet page |
|---|---|---|
| C8 (1 µF) | The VIN pin (sense input to the regulator) **must be locally bypassed with a minimum 1 µF ceramic** — placed close to U1/Q6. | **p. 9** (VIN pin function: "minimum 1 µF ceramic capacitor") |
| C11–C14 (4 × 10 µF, 50 V, X7R/X7S 1210) | Input capacitance suppresses the ripple from discontinuous input current; the datasheet's worst-case input RMS is **IOUT/2 at VIN = 2·VOUT** (≈4 A at the 8 A burst rating), shared across four 1210 ceramics. 50 V rating for the 36 V rail (50 V X7R still derates ~40 % at 36 V — four parts in parallel cover it). The 9–36 V, 300 W reference app uses the same 10 µF/50 V ×4 (+ bulk). | **p. 20** (CIN and COUT Selection + IRMS ≈ IOUT(MAX)·√(VOUT/VIN·(VIN/VOUT−1)), max IOUT/2); **p. 24** (efficiency: CIN filters the large buck-region RMS input current, low ESR required); **p. 27** (300 W app: 10 µF 50 V ×4) |

## 11. VOUT pin bypass / region detect

| Part # | Why this value was chosen | LT8390 datasheet page |
|---|---|---|
| C24 (1 µF) | The VOUT pin senses the output for **region detection** (buck ↔ buck-boost ↔ boost transitions) and powers the LOADTG drive, so it must be locally bypassed with a minimum 1 µF ceramic, wired to the VOUT_REG node. | **p. 10** (VOUT pin function: region detect + "minimum 1 µF ceramic capacitor") |

## 12. Output capacitors

| Part # | Why this value was chosen | LT8390 datasheet page |
|---|---|---|
| C19–C22 (4 × 10 µF, 50 V) + C23 (180 µF polymer) | The datasheet prescribes a **parallel combination** of ceramics (suppress HF switching spikes, placed near the regulator) plus **low-ESR bulk** (OS-CON/POSCAP named — hence the 180 µF polymer) to handle the large boost-region output RMS current without AC I²R loss. Same structure as the reference apps (ceramics + 560 µF bulk on the 300 W). | **p. 20** (CIN and COUT Selection: parallel combination, low ESR/high ripple rating, "ceramic capacitors should be placed near the regulator input and output"); **p. 24** (efficiency: COUT filters large RMS output current in boost; low ESR); **p. 27** (300 W app: 47 µF ×4 + 560 µF ×2) |

## 13. VREF rail and its ties

| Part # | Why this value was chosen | LT8390 datasheet page |
|---|---|---|
| C16 (470 nF) | VREF is a 2.00 V, 1 mA-capable reference; the pin function specifies **0.47 µF ceramic bypass** — value verbatim. | **p. 9** (VREF pin function: 2 V/1 mA + "0.47 µF ceramic capacitor") |
| R10 (499 kΩ, SS→VREF) | Selects **latch-off on output short**: with 499 kΩ the LT8390 "stops switching until the EN/UVLO pin is pulled low and high to restart" — exactly what the over-temp latch does. (No resistor = hiccup; 100 k = keep-running.) | **p. 10** (SS pin function: three fault modes, 499 kΩ = latch-off); **p. 24** (fault-mode details: hiccup/latch-off/keep-running) |
| CTRL + LOADEN ties (no parts) | CTRL must not be left open — tie to VREF (gives the full-scale 100 mV threshold, group 4). LOADEN unused → tie to VREF or INTVCC. Both are direct wires to the VREF pin, so no resistor is needed. | **p. 22** ("CTRL pin should not be left open (tie to VREF if not used)"); **p. 9** (LOADEN pin function: "if the load switch control is not used, tie this pin to VREF or INTVCC") |

## 14. Soft-start

| Part # | Why this value was chosen | LT8390 datasheet page |
|---|---|---|
| C9 (100 nF) | tSS = 1 V·CSS/12.5 µA ≈ **8 ms** — gentle enough to limit inrush through R5 without hitting the 25 A peak limit. The datasheet calls 0.1 µF "a good starting point" and requires **CSS ≥ 5–10× the VC compensation capacitor** (100 nF vs 15 nF = 6.7×, satisfied). | **p. 23** (soft-start: 12.5 µA charge, 0.25 V→1 V ramp); **p. 24** (CSS 5–10× CVC rule + 0.1 µF starting point + SS as fault timer); **p. 10** (SS pin function) |

## 15. Loop compensation

| Part # | Why this value was chosen | LT8390 datasheet page |
|---|---|---|
| R13 (15 kΩ) + C25 (15 nF) | Series R-C from VC to GND is the datasheet's compensation network form. Starting values **copied from the datasheet's own 300 W application (15 k + 15 nF)**, which shares our 9–36 V input range; to be bench-tuned with load transients at both 12 V and 24 V. Must sit close to the IC, between VC and signal ground. | **p. 10** (VC pin function: "compensate the control loop with an external RC network"); **p. 25** (layout checklist: VC network close to IC); **p. 27** (300 W app: 15 nF + 15 k — the value source) |

## 16. Sense-line filters (optional, placed)

| Part # | Why this value was chosen | LT8390 datasheet page |
|---|---|---|
| C26 (100 nF, ISP↔ISN) | R11 sits **between the output decoupling and bulk capacitors** — the exact Figure 14b case where "a low pass filter formed by RF and CF is recommended to reduce the current ripple and stabilize the current loop." 100 nF provisional pending bench check. | **p. 22** (RF/CF filter recommendation, Figure 14b case) |
| C27 (100 nF, LSP↔LSN) | Layout checklist: "the filter capacitor between LSP and LSN should be **as close as possible to the IC**" — added to keep switching-node noise out of the 50 mV sense signal. | **p. 25** (PCB layout checklist: LSP–LSN filter cap) |

## 17. Optional dead-time Schottkys (placed)

| Part # | Why this value was chosen | LT8390 datasheet page |
|---|---|---|
| D5 (across Q4), D6 (across Q3) | The datasheet's optional **DB (parallel with switch B) and DD (parallel with switch D)** conduct during dead time, preventing the synchronous switches' body diodes from storing charge — "they improve converter efficiency and reduce switch voltage stress." Mapped to our refs: DB = Q4 (buck low-side), DD = Q3 (boost high-side). Any 40 V-class, low-Qrr Schottky of adequate current rating works; exact part TBD at ordering. | **p. 20** (Schottky Diode (DB, DD) Selection); **p. 25** (placement: "Place CIN, switch A, switch B and DB in one compact area. Place COUT, switch C, switch D and DD in one compact area") |

## 18. PGOOD pull-up and fault-LED drive

| Part # | Why this value was chosen | LT8390 datasheet page |
|---|---|---|
| R30 (10 kΩ), R31 (1 kΩ) | PGOOD is **open-drain and requires an external pull-up**; allowed to INTVCC (≤6 V abs max). Critically, PGOOD is pulled **low when FB is within ±10 %** (low = good, high-Z = fault), so R30 is the only pull-up: on a fault it lets the node (and Q11's base, via R31) rise to INTVCC and light the red LED; in regulation PGOOD holds it at ~0 V. R30/R31 also set the base current (~0.4 mA) for the ~10 mA LED. | **p. 10** (PGOOD pin function: "pulled low when the FB pin is within ±10 % of the final regulation voltage"); **p. 23** ("requires an external pull-up… to INTVCC or an external voltage source of up to 6 V") |

## 19. Feedback divider / output-voltage select

| Part # | Why this value was chosen | LT8390 datasheet page |
|---|---|---|
| R15 (110 kΩ), R16 (120 kΩ), R17 (10 kΩ), J1 (2-pin shunt) | Output is set by VOUT = 1 V·(R3+R4)/R4 with VFB regulated to **1.00 V**. R17 = 10 kΩ bottom (datasheet-style divider current ~1 mA at 12 V keeps it stiff against FB bias yet low power); 12 V ← Rtop = 110 k; 24 V ← Rtop = 110 k + 120 k = 230 k. J1 shunts R16 so a **missing jumper defaults to 12 V** instead of disconnecting FB (runaway protection — a design-safety choice, not a datasheet value). | **p. 23** (FB divider equation VOUT = 1 V·(R3+R4)/R4; FB also sets PGOOD/short/OV thresholds); **p. 10** (FB pin function: regulates VFB to 1.00 V; short <0.25 V, OV >1.1 V thresholds the jumper scheme must respect) |

## 20. Analog support blocks (datasheet hooks only)

These are custom circuits the LT8390 datasheet does not dictate; only the pins they touch are informed by it. Values derive from the system design (BuckBoostTopology §4.4/§4.5) and the TL431/BC817/BZX84 datasheets.

| Part # | Why this value was chosen | LT8390 datasheet page (hook) |
|---|---|---|
| Q10, R27 (1 kΩ) — latch shutdown pull-down | Q10 must pull EN/UVLO below the **0.3 V hard-off threshold** (not just the 1.220 V falling threshold) for a guaranteed <2 µA shutdown; Figure 12 shows this exact NMOS-shutdown topology. | **p. 9** ("force below 0.3 V to shut down"); **p. 21** (Figure 12 NMOS grounds EN/UVLO for shutdown) |
| U3 (TL431), TH1, D3, R20 (2.2 k), R21 (1.33 k), R22 (4.7 k) — over-temp sense/latch | Zener-fed divider so the ~85 °C trip is independent of the 11–36 V battery rail; R21 sets the trip point (R_NTC ≈ 1.39 k at 85 °C); R22 sized to keep the TL431 above its 1 mA minimum cathode current at 11 V. TL431 SOT-23 pinouts differ (TL431 vs TL432) — verify the part (BuckBoostTopology §4.4). | **p. 22** (CTRL-with-thermistor overtemperature trick — the concept this latch replaces with a hard shutdown); own datasheets: TI SLVS543 (TL431) |
| Q7–Q9, R23–R26, D4, R29 — discrete SCR latch + indicator | Cross-coupled PNP/NPN latch fed from VOUT holds after INTVCC dies; LED hangs off SCR_A (a load cannot false-trigger the latch). No LT8390 content; BC807/BC817 datasheets only. | — (none; EN/UVLO hook as per Q10 row) |
| R18 (4.7 k), R19 (6.8 k), D1, D2 — panel LEDs | Indicator brightness vs dissipation at 36 V: 6.8 k keeps R19/R29 ≤0.17 W in a 1206 (BuckBoostTopology §4.5 power audit). Green on OutPos, red A on output fault (PGOOD). | **p. 18** (PGOOD as the fault source — see group 18); no LT8390 pages for LED values |

---

## Quick page index (Rev. F, 30-page edition)

| Page | Content |
|---|---|
| 1 | Features, description, 12 V/4 A 48 W typical application |
| 2 | Abs max, pin configuration (FE), θJA |
| 3–5 | Electrical Characteristics (incl. EN/UVLO 1.220 V, SS 12.5 µA, INTVCC 4.85–5.15 V, RT = 226 k → 200 kHz) |
| 6–8 | Typical performance characteristics |
| 9–10 | **Pin Functions** (VIN/VOUT 1 µF, VREF 0.47 µF, INTVCC 4.7 µF, SS fault modes 499 k, CTRL ≥1.35 V, PGOOD ±10 %, ISMON 10×+0.25 V, LOADEN tie-off, SYNC/SPRD→INTVCC) |
| 11–16 | Block diagram, Operation (regions, UP/PRE/TRY states) |
| 17 | **RT selection + Table 1 + Spread Spectrum Frequency Modulation** |
| 18 | **Inductor Selection** (ΔIL, LBUCK/LBOOST, 50 mV thresholds, slope comp) |
| 19 | **RSENSE equations + 20–30 % margin + Power MOSFET Selection** |
| 20 | **Schottky DB/DD Selection + CIN and COUT Selection + input IRMS formula** |
| 21 | **INTVCC Regulator (TJ check) + bootstrap caps + UVLO programming (Fig. 12) + RIS current limit** |
| 22 | **CTRL modes (100 mV/RIS) + RIS placement + RF/CF filter (Fig. 13a/14a/14b)** |
| 23 | **ISMON + PGOOD + FB divider VOUT = 1 V·(R3+R4)/R4 + soft-start** |
| 24 | **CSS 5–10× CVC rule + fault modes + efficiency (CIN/COUT low ESR)** |
| 25 | **PCB layout checklist** (ground plane, extra INTVCC 1 µF, LSP–LSN filter cap, VC net, DB/DD placement) |
| 26–27 | **Typical Applications** — 12 V/4 A (p. 26) and 300 W 12 V/25 A (p. 27: 365 k/56.2 k UVLO, 15 k/15 nF comp, 2 mΩ sense) |
| 28–30 | Package drawings, related parts |
