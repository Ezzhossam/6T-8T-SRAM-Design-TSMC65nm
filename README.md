# 6T vs 8T SRAM Cell Design & Verification — TSMC 65nm

Full-custom transistor-level design, layout, and post-layout verification of two SRAM bit-cell architectures (6T and 8T), built and verified end-to-end in Cadence Virtuoso and Mentor Graphics Calibre on the TSMC 65nm CMOS node.

This project implements the complete industry-standard IC design flow: **schematic capture → sizing analysis → transient verification → physical layout → DRC → LVS → parasitic extraction → post-layout re-simulation**, and uses that flow to quantify a real architectural trade-off: read stability vs. write speed.

> Course project — ECE 411, IC Technology, Ain Shams University (Faculty of Engineering, ICHEP)

---

## Why this project

The 6T SRAM cell is the standard building block of on-chip memory, but its shared read/write bitlines create a well-known reliability problem: reading a cell electrically disturbs the stored value. The 8T cell fixes this by adding a dedicated, isolated read port — at the cost of extra area and interconnect.

Rather than just building each cell, this project **measures** that trade-off directly, using simulation data from both the ideal schematic and the physically extracted (parasitic-included) layout.

## Key Results

| Metric | 6T SRAM | 8T SRAM |
|---|---|---|
| Read-disturb voltage bump | **~220 mV** on internal storage node | **0 mV** — node stays flat at 1.2V (isolated read port) |
| Write delay — ideal schematic | 50 ps | 45 ps |
| Write delay — post-layout (PEX) | 92 ps | 73 ps |
| Delay degradation from parasitics | +84% | +62% |
| DRC violations | 0 | 0 |
| LVS result | Clean match (CORRECT) | Clean match (CORRECT) |

**Takeaway:** the 8T architecture completely eliminates read-disturb by physically isolating the read path from the storage nodes, at a modest area/routing cost that shows up as extra parasitic delay — a textbook example of the stability-vs-performance trade-off in memory design.

---

## Architecture Overview

### 6T SRAM Cell
- Two cross-coupled CMOS inverters forming a bistable latch (Q / QB)
- Two NMOS access transistors gated by a shared Wordline (WL), connecting the cell to complementary bitlines (BL / BLB)
- Read and write share the same bitlines — this is what causes read-disturb

**Design constraints applied:**
- **Cell Ratio (CR)** — pull-down NMOS width / access NMOS width — sized to **1.2** to bound the read-disturb voltage bump below the opposing inverter's switching threshold
- **Pull-Up Ratio (PR)** — pull-up PMOS width / access NMOS width — sized to **0.8** to guarantee the access transistor can overpower the PMOS during a write

| Transistor | W | L |
|---|---|---|
| Access NMOS | 300 nm | 60 nm |
| Pull-down NMOS | 360 nm | 60 nm |
| Pull-up PMOS | 240 nm | 60 nm |

### 8T SRAM Cell
- Reuses the exact 6T core (identical transistor sizing — a controlled comparison, not a redesign)
- Adds a dedicated 2-transistor read stack (Read Driver NMOS + Read Access NMOS) with its own Read Wordline (RWL) and Read Bitline (RBL)
- Because the read stack never touches the internal storage nodes, the Cell Ratio constraint becomes irrelevant — read transistors were sized at the technology minimum (W = 200nm, L = 60nm)
- Configured as an **inverting read port**: storing a '1' pulls RBL to '0'

---

## Verification Flow

![Verification Pipeline](figures/verification_pipeline.svg)

Each cell went through the same rigorous pipeline:

1. **Schematic capture** in Cadence Virtuoso, sized per the CR/PR analysis above
2. **Transient testbench verification** in Cadence ADE — multi-cycle Write → Hold → Read sequences with custom analog stimuli, forced initial conditions, and (for the 6T cell) dummy 50fF bitline capacitors to model realistic bitline loading
3. **Physical layout** drawn in Virtuoso, with careful VDD/GND rail routing and minimized cross-coupled inverter spacing to reduce internal node capacitance
4. **Design Rule Check (DRC)** in Calibre — zero violations on both cells
5. **Layout Versus Schematic (LVS)** in Calibre — clean netlist match on both cells
6. **Parasitic Extraction (PEX)** — full RC extraction of the physical layout
7. **Post-layout re-simulation** using the PEX-extracted netlist to measure real-world timing degradation

## Figures & Design Rationale

> Filenames below assume `.png` — if you exported as `.jpg`/`.gif`, just find-and-replace the extension.

### 6T SRAM — `figures/6T/`

![6T Schematic](figures/6T/schematic.png)
**Cadence schematic.** The CR/PR-derived widths (300nm access, 360nm pull-down, 240nm pull-up, all at minimum L=60nm) implemented and routed in Cadence Virtuoso.

![6T Testbench](figures/6T/TB.png)
**Testbench schematic.** A bare 6-transistor cell has no way to be driven or observed on its own — this adds write drivers to actually load data onto the bitlines, and a way to force known internal states.

![WL Stimulus](figures/6T/WL%20STIM.png)
![W_ENABLE Stimulus](figures/6T/W_ENABLE%20STIM.png)
![BL_DATA Stimulus](figures/6T/BL_DATA%20STIM.png)
**Stimulus configuration (WL, W_ENABLE, BL_DATA).** Included to explain a debugging finding: driving BL/BLB directly with **ideal voltage sources** initially broke the read simulation, since an ideal source instantly overpowers the cell's small transistors — completely unrealistic. The fix was routing write data through **NMOS pass-transistor write drivers** (gated by W_ENABLE) instead, so the bitlines behave like a real circuit — including a realistic "weak 1 / strong 0" pass-transistor threshold drop.

![Initial Conditions](figures/6T/IC.png)
**Initial conditions.** Why these were forced at all: without them the simulator has no defined starting state for Q/QB, and — combined with the ideal-source issue above — the first read would be meaningless. Forcing Q=0V / QB=1.2V (bitlines at 1.2V) gives the simulation a known, realistic starting point so the first read actually measures the CR-bounded disturb bump rather than simulator noise.

![Schematic-level transient](figures/6T/scematic_ver.png)
**Transient simulation — ideal schematic (no PEX).** The functional baseline: the read phase shows the CR-bounded disturb bump, the hold phase shows the latch staying stable, and the write phase shows the access transistor successfully flipping the cell.

![Schematic-level transient V4](figures/6T/TB_V4_NO_PEX.png)
**A second schematic-level transient run**, corroborating the same ideal-case behavior above.

![Post-layout transient with PEX](figures/6T/PEX_ver.png)
**Transient simulation — post-layout (with PEX).** The same testbench re-simulated using the PEX-extracted netlist. This is what actually caught the write delay degrading from 50ps (ideal) to 92ps (real routing R/C) — a result the ideal schematic alone could never reveal.

![Post-layout transient V4](figures/6T/TB_V4_PEX.png)
**A second post-layout (PEX) transient run**, corroborating the delay measurement above.

![6T Layout](figures/6T/Layout.png)
**Physical layout.** Drawn with deliberate attention to VDD/GND rail routing and minimizing distance between the cross-coupled inverters — shorter routing directly reduces the internal node capacitance that later shows up as post-layout delay.

![6T DRC](figures/6T/DRC.png)
**DRC — zero violations.** Confirms the layout is manufacturable under TSMC 65nm design rules (spacing, minimum widths, enclosure).

![6T LVS](figures/6T/LVS.png)
**LVS — clean match.** Confirms the physical layout is electrically identical to the verified schematic — the layout didn't silently change the circuit's behavior.

![6T PEX completion](figures/6T/PEX1.png)
**PEX completion.** Calibre's confirmation message that parasitic extraction finished with zero warnings/errors.

![6T PEX extracted view](figures/6T/PEX2.png)
**Extracted parasitics.** The full extracted view showing all parasitic resistors and capacitors overlaid on the layout — the actual RC data that feeds the post-layout simulation.

### 8T SRAM — `figures/8T/`

![8T Schematic](figures/8T/SCHEMATIC.png)
**Cadence schematic.** Reuses the *exact* 6T sizing for the core (a deliberate choice to keep the comparison controlled) and adds a 2-transistor read stack sized at the technology minimum (200nm/60nm), since the read path carries no risk of a read-upset and doesn't need to be sized against anything.

![8T Testbench](figures/8T/TB.png)
**Testbench schematic.** Differs from the 6T version because it has to drive the write port (WWL/WBL/WBLB) and read port (RWL/RBL) completely independently — unlike the 6T cell, they're now physically separate signals.

![WBL Stimulus](figures/8T/WBL_STIM.png)
![WBLB Stimulus](figures/8T/WBLB_STIM.png)
![WWL Stimulus](figures/8T/WWL_STIM.png)
![RWL Stimulus](figures/8T/RWL_STIM.png)
**Stimulus configuration (WBL, WBLB, WWL, RWL).** Documents exactly how the independent read/write timing was sequenced — getting the RWL and WWL pulses timed correctly relative to each other is what makes the Write → Read → Write proof valid.

![8T transient without PEX](figures/8T/TB_WITH_OUT_PEX.png)
**Transient simulation — ideal schematic (no PEX).** The key comparison figure of the whole project: during the read phase, node Q stays **perfectly flat at 1.2V** — directly contrasting with the ~220mV bump seen in the 6T read. This is the core simulation evidence for the entire architectural argument (isolated read port = zero read-disturb).

![8T transient with PEX](figures/8T/TB_WITH_PEX.png)
**Transient simulation — post-layout (with PEX).** The same testbench re-simulated using the PEX-extracted netlist — shows write delay degrading from 45ps (ideal) to 73ps (real routing parasitics included).

![8T Layout](figures/8T/Layout.png)
**Physical layout** of the 8T cell.

![8T DRC](figures/8T/DRC.png)
**DRC — zero violations** on the larger 8-transistor cell.

![8T LVS](figures/8T/LVS.png)
**LVS — clean match** confirming layout-schematic equivalence.

![8T PEX completion](figures/8T/PEX1.png)
**PEX completion.** Calibre's confirmation message that extraction ran with zero warnings/errors.

![8T PEX extracted view](figures/8T/PEX2.png)
**Extracted parasitics.** The combined view of all extracted parasitic resistors and MOSFETs overlaid on the layout, in one image.

![8T PEX zoomed](figures/8T/PEX3.png)
**Extracted parasitics — zoomed detail.** A closer view of the same extracted network from `PEX2`, showing the RC detail more clearly at the transistor level.

A zoomed-in detail view of the same extracted parasitics in `PEX2`, showing the RC network more clearly at the transistor level.
---

## Repo Structure

```
.
├── README.md
├── LICENSE
├── docs/
│   ├── ECE411_SRAM_Report.pdf        # Full project report (theory, derivations, all figures)
│   └── Project_Description.pdf       # Original course assignment brief
├── cadence_project/
│   └── SRAM_PARTA.rar                # Cadence Virtuoso schematic/layout library (6T + 8T cells & testbenches)
└── figures/
    ├── verification_pipeline.svg     # Design flow diagram
    ├── 6T/                           # Cadence/Calibre screenshots for the 6T cell (see "Figures & Design Rationale")
    └── 8T/                           # Cadence/Calibre screenshots for the 8T cell
```

> **Opening the Cadence project:** `SRAM_PARTA.rar` contains the Virtuoso library used for this project (schematics, layouts, and testbenches for both cells). It requires **Cadence Virtuoso** with a **TSMC 65nm PDK** license set up locally to open — the PDK itself is not included here, as it's under foundry NDA. Everyone without local Cadence/PDK access can review the full design and results through the report and figures instead.

## Tools Used

- **Cadence Virtuoso** — schematic capture, physical layout, ADE transient simulation
- **Mentor Graphics Calibre** — DRC, LVS, PEX parasitic extraction
- **TSMC 65nm CMOS** process design kit

## Challenges & Debugging Notes

- Ideal voltage-source stimuli in simulation initially caused unrealistic instant bitline discharge (an ideal source overpowers the cell's small transistors immediately). This was resolved by forcing a **read as the first operation via initial conditions**, and later gating bitline pull-up through the write-driver NMOS pass transistors themselves rather than ideal sources.
- The isolated single-cell testbench had no realistic bitline capacitance, which masked the true read-disturb behavior. Adding **50fF dummy capacitors** on the bitlines correctly anchored the voltage and revealed the actual read-disturb bump.

## Conclusion

- **6T** is area-efficient but structurally vulnerable during reads — a ~220mV disturb bump was measured on the internal node every read cycle, safely below the flip threshold only because of careful CR sizing.
- **8T** eliminates read-disturb entirely via electrical isolation of the read path — internal nodes stayed perfectly flat during read in simulation.
- **Post-layout extraction mattered:** parasitic routing capacitance degraded write delay by 84% (6T) and 62% (8T) relative to the ideal schematic, underscoring why post-layout verification is non-negotiable at deep-submicron nodes.

---

## License

This project is licensed under the [MIT License](LICENSE). Note that TSMC 65nm PDK files are **not** included and remain subject to foundry NDA.

### Author
Eiz Eldeen Housam Elhusany Mohamed Abdel-Aal — ID 21P0170
ECE 411, IC Technology — Ain Shams University, Faculty of Engineering, ICHEP
