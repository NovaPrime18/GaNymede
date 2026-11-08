# GaNymede — loss budget, copper pours and heatsink sizing

Analysis of `GaNymede/GaNymede.kicad_pcb` at **60 V bus, 100 A phase peak for 1 minute,
50 A phase peak continuous**, 3-phase FOC into a direct-drive hub.

The copper part is the same method as `nitride-nano/firmware/analysis/plane-dc-losses`
(extract every copper item of a net → rasterise → sheet-resistance network with via
barrels → 1 A solve → `P = I²R`), retargeted at this 4-layer board. The semiconductor
part is a port of your `gan-matlab-main/Switching_Losses/motorDriveLoss.m` with the
EPC2361 data from `EPC2361.m` and the gate drive from this schematic.

---

## The short answers

| Question | Answer |
|---|---|
| Total loss at 100 A peak | **≈ 32 W** at 20 °C copper, **≈ 34 W** hot (99.4 %) |
| Total loss at 50 A sustained | **≈ 8.8 W** (99.7 %) |
| Loss the **heatsink** must take | **≈ 5 W continuous, up to ≈ 20 W by the end of the 1-minute burst (≈ 14 W average)** |
| Loss the **PCB** must take | ≈ 4 W continuous, ≈ 15 W during the burst |
| Are the copper pours enough? | **Yes at 50 A.** At 100 A they burn 7.5 W, and the switch-node pours are the weak spot. |
| Junction temperature | 43–49 °C sustained, 60–84 °C at the end of a 100 A minute — **with a good thermal pad**. |
| Heatsink needed | **1.5–2 °C/W** natural convection, or bolt it to the frame. |

Two things dominate and neither is the GaN: the **0.5 mΩ phase shunts** (7.5 W at
100 A) and the **60 V / GND copper** (5.1 W at 86.6 A battery current). The six
EPC2361s together are 16.5 W.

![loss breakdown](.analysis/losses/fig/loss_breakdown.png)

---

## 1. Operating point and assumptions

| Item | Value | Why |
|---|---|---|
| Bus | 60 V | as asked |
| Phase current | 100 A / 50 A **amplitude** (70.7 / 35.4 A RMS) | VESC-style "phase amps" |
| Modulation | m = 1.0 | worst case for battery current and bus-copper loss |
| Power factor | 1.0 | current in phase with back-EMF |
| Switching | 20 kHz | **assumed** — sweep below |
| Dead time | 20 ns | **firmware-set, see §5** |
| Device | EPC2361, 0.75 mΩ typ @ 25 °C, ×6 | schematic |
| Rds temperature | +0.36 %/°C + quadratic → 0.99 mΩ @ 100 °C | datasheet Fig. 9 |
| Shunts | 2 × 1 mΩ 2512 in parallel = **0.5 mΩ per phase** | R97/R100 … |
| Gate drive | 5 V, 2.2 Ω, LMG1210 | R13/R16/… |
| Ambient | 30 °C | summer e-bike |
| Output power | 5.20 kW at 100 A, 2.60 kW at 50 A | 1.5·V<sub>pk,ph</sub>·I<sub>pk</sub> |

If "100 phase amps" meant 100 A RMS rather than 100 A amplitude, I<sub>rms</sub> is √2
higher and every I²R term doubles: conduction ~30 W, shunts 15 W, copper ~15 W,
total ~62 W, and the junctions would run past 150 °C. I have assumed amplitude
throughout.

---

## 2. Loss budget

At 20 kHz, devices at 100 °C, copper at 20 °C (see the note below the table):

| | 100 A pk (1 min) | 50 A pk (sustained) |
|---|---|---|
| GaN conduction (6 devices) | 14.8 W | 3.7 W |
| GaN switching (E<sub>on</sub>+E<sub>off</sub>) | 1.0 W | 0.5 W |
| Coss (hard-switched) | 0.35 W | 0.35 W |
| Dead-time reverse conduction | 0.34 W | 0.16 W |
| **GaN total** | **16.5 W** | **4.7 W** |
| Phase shunts (0.5 mΩ × 3) | 7.5 W | 1.9 W |
| 60 V + GND copper | 5.04 W | 1.27 W |
| Switch-node + phase copper | 2.42 W | 0.61 W |
| Snubbers (2.2 nF + 1 Ω × 3) | 0.24 W | 0.24 W |
| Bus-cap ESR (estimate) | 0.40 W | 0.10 W |
| Gate drive | 0.02 W | 0.02 W |
| **Total** | **32.1 W** | **8.8 W** |
| Efficiency | 99.38 % | 99.66 % |

Conduction is 3·R<sub>ds(on)</sub>·I<sub>rms</sub>², i.e. each device carries the full
phase current for half the time — 2.47 W per device at 100 A, 0.62 W at 50 A.
Switching is anchored to EPC's measured data: at 48 V/20 A the model gives
E<sub>on</sub> = 2.20 µJ against EPC23102's 2.5 µJ, and E<sub>on</sub>+E<sub>off</sub> = 2.54 µJ
against 2.65 µJ.

**Switching frequency sweep** (100 A peak): 10 kHz → 31.1 W, 20 kHz → 32.1 W,
30 kHz → 33.0 W, 40 kHz → 34.0 W. Roughly +0.1 W per kHz. Not a reason to avoid
40 kHz if the motor benefits from it, but there is no reward for going above 20 kHz
either — with a DD hub the ripple current is only 0.33 A RMS at 200 µH.

**Max-Rds devices** (1.0 mΩ rather than 0.75 mΩ): conduction 19.7 W, total 37.0 W at
100 A. Still fine thermally, but budget for it.

**Temperature note.** The copper numbers are quoted at 20 °C. At the temperatures the
board actually reaches (≈50 °C at 50 A, ≈85 °C in the burst) copper resistance is
12–25 % higher, so the burst total is closer to **33–34 W** and the sustained total to
**8.3 W** (the sustained figure is lower than the table because the dies sit at 50 °C,
not 100 °C, so R<sub>ds(on)</sub> is smaller).

---

## 3. Copper pours

Every net was solved with the poured copper as it is in the file (filled zones,
thermal spokes, antipads, 0.5/0.3 mm vias, 2 oz on all four layers, 20 °C):

| Net | Layers | Copper area | R<sub>eff</sub> | 100 A pk | 50 A pk |
|---|---|---|---|---|---|
| `Vdrive` (60 V) | F.Cu + In1.Cu | 2 716 mm² | **0.387 mΩ** | 86.6 A → 2.90 W | 43.3 A → 0.73 W |
| `GND` | all 4 | 12 136 mm² | **0.286 mΩ** | 86.6 A → 2.14 W | 43.3 A → 0.54 W |
| `Net-(U3-+)` switch node U | F.Cu | 57 mm² | 0.105 mΩ | 70.7 A → 0.53 W | 35.4 A → 0.13 W |
| `Net-(U4-+)` switch node V | F.Cu | 58 mm² | 0.119 mΩ | → 0.59 W | → 0.15 W |
| `Net-(U5-+)` switch node W | F.Cu | 57 mm² | 0.173 mΩ | → 0.86 W | → 0.22 W |
| `/Phase/Phase` U out | all 4 | 300 mm² | 0.022 mΩ | → 0.11 W | → 0.03 W |
| `/Phase1/Phase` V out | all 4 | 292 mm² | 0.034 mΩ | → 0.17 W | → 0.04 W |
| `/Phase2/Phase` W out | all 4 | 295 mm² | 0.033 mΩ | → 0.17 W | → 0.04 W |
| | | | **total** | **7.47 W** | **1.87 W** |

![copper](.analysis/losses/fig/copper.png)

**Verdict.**

* **The phase-output pours are excellent.** Four layers, 21 vias, ~300 mm² and
  22–34 µΩ: 0.1–0.2 W each at 100 A. Nothing to do here.
* **The switch-node pours are the bottleneck.** They are F.Cu only, 57 mm², and
  carry the full phase current whenever either device conducts. 0.53–0.86 W each.
  The W phase is 65 % worse than U purely because of layout — if you ever want more
  current, fix W first (it looks like the pour is longer/narrower between Q5/Q6 and
  R99/R102).
* **The 60 V feed is 0.39 mΩ and 88 % of that is F.Cu.** The In1 plane only carries
  ~11 % of the bus current (its share of the loss is 0.043 of 0.387 mΩ) because it is
  a stub joined to F.Cu by 61 scattered vias, most of them at capacitor pads. A via
  fence along the plane boundary would let In1 take 30–40 %, which is worth roughly
  1 W at 100 A — optional, not a defect.
* 86.6 A battery current at full speed is also close to an XT90's rating; worth
  checking the connector, not the PCB.

At 50 A sustained the whole copper set is 1.9 W — about 0.07 % of throughput. **Yes,
you have enough copper for 50 A indefinitely.**

---

## 4. Junction temperature and the heatsink

Heat leaves the EPC2361 two ways: top (R<sub>θJC</sub> = 0.2 °C/W) through your pad
into the heatsink, and bottom (R<sub>θJB</sub> = 1.5 °C/W) into the PCB. Solved as a
3-node network (junction → sink / board, 30 °C ambient, 6 °C/W board-to-air,
0.5 °C/W contact, 12 mm² contact area on the 3 × 5 mm top):

| | 50 A sustained | end of 1 min @ 100 A | 100 A steady (hypothetical) |
|---|---|---|---|
| **0.5 mm, 3 W/mK pad** | | | |
| T<sub>j</sub> | 50 °C | 84 °C | 114 °C |
| heatsink | 38 °C | 40 °C | 62 °C |
| board | 50 °C | 85 °C | 115 °C |
| **0.25 mm, 6 W/mK pad** | | | |
| T<sub>j</sub> | 43 °C | 60 °C | 83 °C |
| heatsink | 39 °C | 43 °C | 66 °C |
| board | 44 °C | 62 °C | 85 °C |

![thermal](.analysis/losses/fig/thermal.png)

**The pad, not the heatsink, sets the junction temperature.** A 0.5 mm silicone pad
is ~14 °C/W per device — 70× the die's own junction-to-case resistance, and larger
than a whole heatsink. Go to 0.25 mm / ≥6 W/mK (or a thin polyimide/graphite pad),
and keep the contact area as close to the full 3 × 5 mm as the gate land allows.
Going from 0.5 mm/3 W/mK to 0.25 mm/6 W/mK is worth **24–31 °C** at 100 A.

**How much must the heatsink dissipate?**

* **Continuous (50 A):** 4.0 W is made in the dies; **≈ 5–6 W** arrives at the
  heatsink (it also pulls ~1–2 W up out of the PCB).
* **1-minute burst (100 A):** 16.5 W is made in the dies; **≈ 6 W at the start
  rising to ≈ 20 W by the end** arrives at the heatsink (≈ 14 W average), because
  early on most of the heat still goes down into the cold PCB.

![transient](.analysis/losses/fig/transient.png)

**The 1-minute rating is a thermal-mass rating, not a heatsink rating.** The 200 g
aluminium sink gains only ~3–4 °C in the minute. The PCB is what heats up: it carries
the shunts and the bus copper (~15 W), and rises 20–35 °C in the burst (pad dependent). That is fine
once, but repeated 100 A pulls will walk the board up towards its 100 °C steady state.

**Heatsink recommendation:** 1.5–2 °C/W natural convection — a 100 × 80 × 10 mm finned
block, or simply bolt the plate to the bike frame (≪ 0.5 °C/W, and then the pad is the
only meaningful resistance). Even a 3 °C/W sink keeps T<sub>j</sub> at 49 °C
continuous with that pad. The real requirement is that it is **flat** and that all six devices are
coplanar under it: the pad thickness tolerance is 14 °C/W per half-millimetre, so a
rocking sink is worse than a small one.

---

## 5. Things I noticed while doing this

1. **The shunts are your second-largest loss** — 0.5 mΩ × 3 phases burns 7.5 W at
   100 A and 1.9 W continuous, more than the six GaNs at 50 A. Each 2512 sees
   1.25 W at the 100 A peak and 0.31 W continuous; check the parts are ≥2 W, or use
   4 × 1 mΩ (0.25 mΩ) and double the INA240 gain headroom — you have plenty
   (50 mV × 20 = 1 V today).
2. **Dead time is set in firmware, not by the driver.** DHL and DLH (pins 5/6 of the
   LMG1210) are tied to +5 V, which selects independent-input mode; the MCU drives
   H_U/L_U separately. At 100 A/20 kHz every 100 ns of dead time costs 1.7 W
   (20 ns → 0.34 W, 50 ns → 0.84 W, 200 ns → 3.4 W). The LMG1210's delay matching
   allows 10–20 ns; do not run VESC-default dead times here.
3. **The snubbers work but cost 0.24 W** at 20 kHz (2.2 nF × 60 V² × f<sub>sw</sub> × 3).
   That is the price of the clean switching node; fine.
4. **PDN inductance is not in this model.** At 100 A with 2.2 nF snubbers the
   turn-off overshoot will set the real voltage margin, not the 100 V rating. Worth a
   double-pulse test before trusting 60 V + margin on a 100 V device.
5. The bus-cap ESR number (0.4 W) is an estimate — the MLCCs take most of the HF
   ripple. It is small enough not to change any conclusion.

---

## 6. Method and validation

* Copper: `plane_dc_loss_gn.py` (nitride-nano tool, 4-layer/0.3 mm-via retarget) +
  `run_pour.py`. Grid 0.05 mm (0.1 mm for Vdrive, 0.25 mm for GND), 3×3 subcell
  coverage, via barrels 0.3 mm drill / 25 µm plating, ρ = 1.72e-8 Ω·m.
  * Mesh convergence: Vdrive 0.3874 → 0.3869 mΩ (0.1 → 0.075 mm);
    phase U 0.0216 → 0.0215 mΩ; switch node W 0.1727 → 0.1724 mΩ.
  * Energy balance `Σg·ΔV² = I·ΔV` checked on every net.
* Semiconductors: `loss_model.py`. Rds(T) fitted to datasheet Fig. 9; Eoss from
  Qoss(50 V) = 90 nC scaled √(V/50); reverse conduction from Fig. 8.
* Thermal: `solve_tj` / `transient` in `loss_model.py`; 3-node RC with
  180 J/K sink, 20 J/K board, copper resistivity scaled with board temperature.

Reproduce:

```bash
cd .analysis/losses
V=/home/tj/nitride-nano/firmware/analysis/plane-dc-losses/.venv/bin/python
$V run_pour.py --nets "Net-(U3-+),Net-(U4-+),Net-(U5-+),/Phase/Phase,/Phase1/Phase,/Phase2/Phase" --pitch 0.05 --maps
$V run_pour.py --nets Vdrive --pitch 0.075
$V run_pour.py --nets GND --pitch 0.1
$V loss_model.py ; $V make_figs.py ; $V fig_copper.py
```

---

## 7. What I need from you to firm this up

1. **Switching frequency** — I used 20 kHz. 10–40 kHz moves the total by ~3 W.
2. **What "100 phase amps" means** — amplitude (assumed) or RMS?
3. **Thermal pad** — thickness and conductivity you can actually get, and whether the
   heatsink presses on all six devices with a flat face.
4. **Airflow** — is the PCB inside a sealed box, or will air move over it? That is
   what sets the board temperature during the 1-minute burst.
5. Motor inductance and Kv would let me replace the m = 1 assumption with the real
   speed/current locus, which mainly affects the 60 V copper and the bus caps.

---

## Addendum — halved shunts (0.25 mΩ) with 0.4 mm 6 W/mK pads

The main report analyses the board **as drawn**. This is the same model with the
shunt pair changed from 2 × 1 mΩ to 2 × 0.5 mΩ (0.5 → 0.25 mΩ per phase) and the
existing 0.5 mm / 6 W/mK pads compressed to 0.4 mm (6.9 → 5.6 °C/W per device):

| | as drawn, 0.4 mm pad | 0.25 mΩ shunts, 0.4 mm pad |
|---|---|---|
| Total, 100 A pk | 33.6 W | **29.2 W** |
| Total, 50 A sustained | 8.3 W | **7.3 W** |
| T<sub>j</sub>, 50 A | 45 °C | 43 °C |
| T<sub>j</sub>, 100 A steady | 89 °C | **82 °C** |
| T<sub>j</sub>, 1-min burst | 47 → 66 °C | **45 → 62 °C** |
| Board, end of burst | 68 °C | 63 °C |
| Shunt loss, 100 A pk | 7.5 W | 3.8 W |
| Per 2512 at 100 A pk | 1.25 W | 0.63 W |

Sensing at 0.25 mΩ: 100 A → 25 mV → 0.5 V at the INA240A1's gain of 20. On a
1.65 V-biased 3.3 V ADC that is ±1.65 V / 20 / 0.25 mΩ = **±330 A** of theoretical
range, and 0.16 A/LSB. Plenty.

The pad change is worth ~2–3 °C; the shunt change is worth ~7–8 °C and 4.4 W.
Neither changes any conclusion, both are free.
