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

---

## Repo Structure

```
.
├── README.md
├── docs/
│   └── ECE411_SRAM_Report.pdf        # Full project report (theory, derivations, all figures)
├── schematics/
│   ├── 6T_SRAM/                      # Schematic + testbench netlists/exports
│   └── 8T_SRAM/
├── layout/
│   ├── 6T_SRAM/                      # Layout exports (GDS/screenshots)
│   └── 8T_SRAM/
├── verification/
│   ├── drc_reports/                  # Calibre DRC logs
│   ├── lvs_reports/                  # Calibre LVS logs
│   └── pex_results/                  # Extracted netlists / parasitic reports
└── figures/                          # Simulation waveform screenshots
```

> Note: this repo does not include TSMC 65nm PDK files (technology/rule files), which are under foundry NDA. Only original schematic/layout work and simulation results are shared here.

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
