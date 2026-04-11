# PCB Thermal Design for LPDDRx

This section covers thermal design for LPDDRx interfaces at the PCB level. Thermal management of LPDDR has become a first-class design concern because DRAM performance degrades at elevated temperature (refresh rate doubles, retention time falls, timing parameters relax) and because LPDDR5/5X now dissipates enough power in high-bandwidth workloads that junction temperature can limit sustained performance. The PCB is the primary heat-conduction path from the DRAM die to the ambient environment, so PCB copper distribution, stackup, and mechanical integration directly determine thermal performance.

---

### Q1. How much power does an LPDDRx device dissipate, and how does it break down?

**Answer:**

LPDDR power dissipation depends on the mode, bandwidth, and activity level. Typical dissipation numbers for a single x32 LPDDR channel (one DRAM package, two ranks) at peak activity:

- LPDDR4X at 4266 MT/s: 300-500 mW for the DRAM, 150-300 mW for the SoC PHY
- LPDDR5 at 6400 MT/s: 500-800 mW for the DRAM, 200-400 mW for the SoC PHY
- LPDDR5X at 8533 MT/s: 700-1200 mW for the DRAM, 300-600 mW for the SoC PHY
- LPDDR5X at 9600 MT/s: 900-1500 mW for the DRAM, 400-800 mW for the SoC PHY

The breakdown within the DRAM is roughly: IO drivers and receivers 30-40% (scales with bandwidth), sense amplifiers and row access 30-40% (scales with activation rate), refresh 10-15% (baseline cost), and leakage 10-20% (increases with temperature). The breakdown within the SoC PHY is IO drivers 40-50%, PLL/DLL 15-25%, clock distribution 15-20%, and training logic 5-10% (mostly idle after boot).

Background/idle power is much lower: 50-100 mW for DRAM in self-refresh, near-zero for the SoC PHY when the link is down. The thermal design must handle the peak sustained power, not just the idle power, because smartphones, SoCs and AI accelerators run bursts of peak activity for seconds to minutes.

Power density is the more important metric for thermal design. An LPDDR5X DRAM package is typically 10x10 to 12x12 mm, giving a power density of 10-15 mW/mm^2 at peak — comparable to a small CPU core. Without active cooling or a good PCB thermal path, the die temperature rises rapidly.

---

### Q2. What is the maximum operating junction temperature for LPDDRx, and what happens above it?

**Answer:**

JEDEC specifies LPDDR operating temperature ranges. Standard commercial LPDDR is rated to 85C junction (extended to 95C for some devices); industrial LPDDR is rated to 105C; automotive LPDDR is rated to 125C or 150C depending on the grade.

Behaviour above the rated range, in order of severity:

- At T_j below 85C (standard operating range), refresh period is the nominal 32 or 64 ms and timing is nominal.
- At T_j 85C to 95C, LPDDR enters "extended temperature mode", which typically doubles the refresh rate (from 32 ms to 16 ms) to preserve data retention. Bandwidth is slightly reduced because refresh operations steal bus cycles. This is recoverable and transparent to software.
- At T_j 95C to 105C, most LPDDR4 and 5 devices refuse new accesses and assert an overheat indicator. The controller must throttle or halt until temperature drops.
- Above 105C (commercial parts) or 125C (industrial), the DRAM may lose data due to insufficient refresh at the reduced retention time, and can suffer permanent damage if held at high temperature for extended periods.

LPDDR has on-die temperature sensors (TCSR and MPC commands) that the controller can read to monitor each device. When temperature rises, the controller can reduce the DRAM frequency (entering a lower-rate mode such as dropping from LPDDR5X 8533 MT/s to 6400 MT/s), reduce bus utilisation, or throttle the CPU/GPU load. This is the "thermal throttling" behaviour that users experience as reduced performance during prolonged heavy workloads.

The critical insight for the PCB thermal designer is that the thermal budget is not the absolute maximum rating (85C or 95C) but a lower number that leaves headroom for ambient rise and workload margin. For consumer products aiming for 40C ambient with 90C maximum, the PCB must limit the DRAM-to-ambient delta-T to 50C at peak dissipation.

---

### Q3. What is the dominant heat-conduction path from an LPDDR die to ambient?

**Answer:**

The heat path from an LPDDR die to ambient air has several parallel and series legs. For a BGA-mounted DRAM without a heat spreader, the dominant path is usually through the PCB, not through the top of the package.

The thermal network from die to ambient:

1. Die to package substrate: through die-attach adhesive or flip-chip bumps. Junction-to-case top (psi-JT) is typically 5-10 C/W; junction-to-case bottom (psi-JB, through the balls) is typically 3-7 C/W.
2. Package to PCB: through the solder balls. Each ball has thermal resistance of approximately 200-400 K/W; with 100-200 ground and power balls in parallel, the aggregate is 1-3 K/W.
3. PCB spreading: the PCB copper layers spread heat laterally. A 4-layer PCB with 1 oz copper and good ground planes has a spreading resistance of 5-15 K/W for a 1 cm^2 heat source.
4. PCB to ambient (top): radiation and natural convection from the component and solder mask surface. Roughly 50-100 K/W for a small package in still air.
5. PCB to ambient (bottom or through PCB): if the PCB has thermal vias under the DRAM connecting to a bottom-side copper area, heat conducts through the vias (each via 5-15 K/W, in parallel) to the bottom and radiates from there.
6. PCB to case or frame: if the PCB is mounted against a metal chassis with thermal interface material, this becomes a dominant path, reducing board temperature by 10-20C compared to free air.

For a mobile phone DRAM (PoP configuration), the top of the stack is typically the hot path: DRAM -> SoC top -> thermal interface -> chassis or heat spreader. For non-PoP boards (automotive, embedded), the PCB bottom path dominates and is engineered with thermal vias, copper pours, and chassis coupling.

The total thermal resistance junction-to-ambient (theta-JA) for an LPDDR package on a typical PCB without special thermal features is 30-60 C/W in free air. At 1 W dissipation and 40C ambient, T_j equals 70-100C — very close to the throttle threshold. This is why PCB thermal features are essential for sustained LPDDR performance.

---

### Q4. How are thermal vias used under an LPDDR BGA, and how are they laid out?

**Answer:**

Thermal vias are dedicated vias placed under the LPDDR package (SoC or DRAM) to provide a low-resistance heat path from the BGA solder balls to the PCB internal copper layers and ultimately to a bottom-side copper pour or chassis contact. They do not carry electrical signals beyond their ground or power function; their purpose is heat conduction.

Layout guidelines:

- Place thermal vias directly under the package footprint, in the area not occupied by signal balls. For a typical LPDDR BGA, the corners and centre are ground or power balls and are good targets for thermal vias.
- Each thermal via should be 0.2-0.3 mm drill diameter with 0.4-0.5 mm pad. Smaller vias offer lower heat transfer per via; larger vias waste board area and may be unnecessary if many are used.
- Use a regular grid pattern: 0.5-0.8 mm pitch thermal via array under the package. A 10x10 mm package can accommodate 150-250 thermal vias at 0.65 mm pitch.
- Fill or plug thermal vias. Unfilled vias have lower thermal conductance because the inside of the via is air; filled vias (with conductive epoxy or solder mask plugging plus copper plating) have 50-100% higher thermal conductance. Via-in-pad with full filling is the best but most expensive option.
- Connect thermal vias to as many copper layers as possible. A thermal via that connects the top signal layer to the bottom signal layer via all internal planes carries heat efficiently. A thermal via that only connects two adjacent layers wastes copper.
- Tie thermal vias to both VSS and VDDQ where electrically appropriate, effectively doubling their thermal function by also serving as PDN vias.

A typical LPDDR DRAM with 200 thermal vias under the package, connected to solid ground planes on layers 2, 4, 6, 8, and a copper pour on the bottom layer, achieves a junction-to-board thermal resistance of 4-8 C/W — significantly better than the 10-20 C/W of a board without thermal features.

---

### Q5. How does copper pour area affect LPDDR thermal performance?

**Answer:**

Copper pour is the unrouted copper area on a PCB layer, usually connected to ground. Large copper pours act as heat spreaders, conducting heat laterally from the component footprint to a larger radiating area, reducing the local temperature rise.

The thermal spreading resistance of a copper pour depends on its area, thickness, and thermal conductivity (copper is 400 W/m.K). A 100 mm^2 copper pour (approximately 1 cm^2) on a single 1 oz layer (35 um thick) has a spreading resistance of approximately 10-15 C/W. Doubling the area halves the resistance (for small pours where the heat source is much smaller than the pour); for larger pours the improvement levels off as the pour approaches infinite extent.

Thicker copper (2 oz or 3 oz) improves spreading roughly proportionally. A 2 oz (70 um) pour of the same area has half the spreading resistance of a 1 oz pour. However, 2 oz copper adds cost and affects etch accuracy for fine signal traces on the same layer, so it is typically used on inner ground planes where there are no signals, not on signal layers.

Multiple copper layers in parallel provide lower total spreading resistance. Four 1 oz ground planes stacked (layers 2, 4, 6, 8 of an 8-layer board) give approximately the same spreading as one 4 oz layer, but with better distribution and easier manufacture.

Design guidance for LPDDR:

- Under the SoC and DRAM packages, provide solid ground planes on every possible layer, not just one. Reserve layer 2 as a continuous ground plane with no cuts under the BGA.
- Extend copper pour beyond the package footprint by at least 5-10 mm in every direction to provide a spreading region.
- Do not create thermal isolation gaps (insulated pads, thermal reliefs on vias) for thermal vias; use solid copper connections. Thermal reliefs are for manual soldering, not for reflow assembly of BGAs.
- Avoid large plane cut-outs near the LPDDR package. A routing channel that bisects the ground plane creates a thermal bottleneck and doubles the local temperature rise.
- Use blank copper fill in unused areas of every signal layer to contribute to spreading even on signal layers.

---

### Q6. What role do package-level thermal features (lids, heat spreaders, PoP TIM) play in LPDDR thermal design?

**Answer:**

Most LPDDR packages are bare-die epoxy molded compound (EMC) BGAs without metal lids. The top of the package is plastic with thermal conductivity of 0.5-1 W/m.K — poor. The bottom-side solder balls are the main heat path, which is why PCB thermal vias matter so much.

Some high-performance LPDDR applications use enhanced package thermal features:

- Metal lid on LPDDR DRAM: uncommon but seen on server and automotive LPDDR. Adds a copper or aluminium top cap with thermal conductivity 200-400 W/m.K, providing a top-side heat path to an external heat spreader or heat sink.
- Heat spreader above the package: a separate metal plate (copper or aluminium) placed over the LPDDR package with a thermal interface material (TIM) layer. This is common on automotive boards with sealed enclosures and on AI accelerator modules.
- PoP TIM between SoC and DRAM: in PoP configurations, the gap between the SoC top and the DRAM bottom is filled with underfill material or TIM. Standard underfill is a thermal insulator (0.5-1 W/m.K) and traps SoC heat against the DRAM, often causing the DRAM to run 10-20C hotter than it would in non-PoP. Higher-conductivity underfills (2-4 W/m.K) partially mitigate this.
- Thermal interface pad on the SoC top: for non-PoP SoCs where a heat sink or chassis contact is possible, a high-conductivity TIM (5-10 W/m.K gap pad) couples the SoC to a heat spreader. This is very common on mobile phones and tablets where the phone chassis doubles as the heat spreader.

For PoP specifically, the DRAM-on-top configuration means the SoC heat must pass through the DRAM before reaching any external heat path. The DRAM thus runs hot whenever the SoC is busy, which is exactly when the DRAM is also busy. This is one of the fundamental thermal limits of PoP and is a reason some high-performance designs have moved away from PoP in favour of package-side-by-side with shared heat spreader.

The PCB designer should understand which thermal features the package provides and design the PCB to complement them. If the package has a metal lid and top-side heat path, heavy PCB thermal vias are less critical. If the package is bare BGA, PCB thermal vias become essential.

---

### Q7. How does ambient temperature affect LPDDR PCB thermal design decisions?

**Answer:**

Ambient temperature is the starting point for the thermal budget: T_j equals T_ambient plus theta_JA times P_dissipation. Different applications have very different ambient assumptions, and this drives radically different PCB thermal designs.

Mobile phones and tablets: nominal ambient 25C, maximum 45C in pocket or direct sunlight. The PCB is small (20-100 cm^2), there is no airflow, and the phone chassis doubles as heat spreader. PoP is the norm. Peak sustained power is limited to what can be dissipated through the chassis without burning the user's hand (usually 3-4 W total for the SoC+DRAM+PMIC+camera). LPDDR thermal throttling kicks in during sustained gaming or 4K video recording.

Laptop and Chromebook: ambient 25C, internal air temperature 45-55C due to heat from CPU and display. Active cooling with a fan provides airflow over the PCB, dropping theta_JA by 2-4x compared to still air. LPDDR is usually non-PoP (soldered to the board or in SO-DIMM) with heat spreaders. Sustained power is 5-10 W total.

Server and HPC: ambient 25C, internal air 35-50C with active airflow. High-bandwidth LPDDR (in AI accelerators and GPUs) is often package-side-by-side with the logic die on a silicon interposer (HBM-style) rather than LPDDR, but some designs use LPDDR5X. Liquid cooling or heavy heat sinks with fan coupling are common. Sustained power is 10-30 W for the memory subsystem.

Automotive: ambient -40C to +85C cabin, -40C to +105C engine bay. Non-PoP discrete LPDDR is mandatory due to PoP reliability concerns. The PCB is often mounted to the chassis with thermal paste to use the vehicle chassis as a heat sink. AEC-Q100 Grade 2 parts (rated -40 to +105C) are standard, with Grade 1 (-40 to +125C) for under-hood ECUs. Heavy copper (2 oz or more) on inner planes and extensive thermal vias are typical.

Industrial and outdoor: ambient -40C to +70C. Sealed enclosures with no airflow, relying entirely on PCB and chassis conduction. Large copper pours, thick PCBs, metal cores, and aluminium backing plates are common. Thermal derating must be applied because even extended-temperature LPDDR runs close to its limit in these environments.

The PCB thermal designer must know the target ambient, airflow assumptions, and any chassis coupling, and design accordingly. A PCB that works well in a laptop with active cooling may overheat in a fanless industrial enclosure, even with the same SoC and DRAM.

---

### Q8. How are thermal-electrical coupling effects analysed for LPDDR?

**Answer:**

Thermal-electrical coupling arises because temperature affects electrical parameters and electrical activity affects temperature, forming a feedback loop. For LPDDR, several coupling mechanisms matter:

- Resistive losses increase with temperature. Copper resistivity has a temperature coefficient of approximately 0.4% per degree C. A 50C rise increases PDN resistance by 20%, worsening IR drop and noise margin. This is a mild positive feedback (more drop causes more reliability problems).
- DRAM refresh doubles above 85C, increasing the refresh current and raising local power dissipation by 10-15%. Once the DRAM is in extended temperature mode, the temperature-induced power cannot be reduced without dropping frequency.
- DRAM leakage increases exponentially with temperature. Retention time halves every 10-12C. Above 85C the refresh rate must double to compensate; above 95C some cells fail; above 105C retention is too short for refresh to cover.
- Driver strength decreases at high temperature (mobility decreases with temperature in MOSFETs), slowing edge rates and increasing ISI. Training must re-run if temperature changes significantly; otherwise timing margin erodes.
- Dielectric properties drift with temperature. Standard FR-4 Dk rises 1-2% per 50C, changing trace impedance and propagation velocity. This creates a temperature-dependent timing skew of a few ps per 25 mm of trace over the operating range.

Analysis flow combines thermal and electrical simulation:

1. Compute DC power dissipation at each component based on the worst-case workload.
2. Run thermal simulation (ANSYS Icepak, 6SigmaET, Flotherm, Siemens Simcenter) on the PCB plus enclosure model, finding steady-state temperature at each component.
3. Feed the temperature back into electrical simulation, updating copper resistance, driver strength, Dk, and leakage.
4. Re-compute power dissipation with the updated parameters, check if power has changed significantly.
5. Iterate until convergence — usually 2-3 iterations are enough.

For LPDDR, the electrical-to-thermal link is mild (temperature sensitivity of power is 10-15% over the operating range) but the thermal-to-electrical link can be significant (timing and PDN margins shift noticeably with temperature). The analysis identifies whether the system works at both cold (-40C) and hot (+95C) extremes, since different failure modes dominate at each.

A common failure seen in the lab is "works cold, fails hot": the DRAM refreshes fine at 25C but misses retention at 85C because the PCB thermal design did not account for sustained workload. Fix: either improve the PCB thermal path or add software throttling.

---

### Q9. How does placement of the DRAM relative to other hot components affect LPDDR thermal design?

**Answer:**

Component placement determines thermal coupling between neighbouring devices. Components that run hot (SoCs, GPUs, PMICs, power transistors) heat the PCB and radiate to nearby components. An LPDDR DRAM placed next to a hot SoC receives conducted heat through the PCB copper and radiated heat across the air gap, raising its baseline temperature before LPDDR-generated heat is added.

Design rules to manage placement:

- Keep the DRAM away from the hottest components (main SoC, GPU, power transistors) by at least 5-10 mm if possible. This reduces the thermal coupling and lets each device's thermal path work independently.
- For PoP, the DRAM is physically stacked on the SoC and thermally coupled through the PoP interface. The designer cannot change this, but can ensure the SoC top-side heat path is efficient (TIM, heat spreader, chassis contact) so the SoC heat is removed before it fully reaches the DRAM.
- Place the PMIC and VDDQ LDO on the opposite side of the SoC from the DRAM. The PMIC is a heat source; placing it on the DRAM side adds to the DRAM's thermal environment.
- Isolate hot components from the DRAM with a "thermal cut" — a gap in the copper pour that reduces lateral heat conduction. This is counter-intuitive (you normally want more copper for spreading) but it can be useful when a specific hot spot would otherwise dump its heat directly into the DRAM.
- Use heat-sink partitioning: separate heat sinks for the SoC and the DRAM, with an air gap between them, so the SoC heat exits through its own heat sink without heating the DRAM.
- Place DRAM on the side of the board with better airflow (for forced-air systems) or better chassis contact (for conduction-cooled systems).

For mobile and small form factor designs, placement is heavily constrained by signal integrity (keep the DRAM close to the SoC for short traces) which conflicts with thermal isolation. The design compromise is usually to place the DRAM as close as needed for signal integrity (less than 30 mm for non-PoP) and then mitigate thermal coupling with copper pours, thermal vias, and a shared heat spreader that treats the SoC and DRAM as a single thermal system.

The PCB designer should run a thermal simulation with all significant heat sources included, not just the LPDDR subsystem in isolation. Isolated simulation under-predicts the actual DRAM temperature by 10-30C in dense designs.

---

### Q10. What thermal verification is performed on an LPDDR PCB design before manufacture?

**Answer:**

Thermal verification combines simulation, physical measurement on prototypes, and accelerated reliability testing.

Simulation phase: a 3D thermal model of the PCB assembly is built in Icepak, Flotherm, 6SigmaET, or similar. Inputs include PCB stackup with copper density per layer, component footprints with power dissipation, package thermal models (two-resistor or detailed compact model from the component vendor), enclosure geometry, and boundary conditions (ambient temperature, airflow, chassis coupling). The solver computes steady-state and transient temperature fields. Outputs are colour-coded temperature maps and junction temperatures for each component. The design iterates until all junction temperatures are below their limits with margin.

Prototype measurement: the first PCB build is tested in a thermal chamber or at room temperature with a representative workload. Measurement techniques include:

- Thermocouples placed on the top of each component package (fast and cheap, but measures case temperature not junction temperature; junction is estimated from psi-JT and power).
- Thermal imaging camera (IR camera) pointing at the PCB, giving a full temperature map in real time. The camera must be calibrated for the component surface emissivity (plastic packages have emissivity of approximately 0.9, metal lids have emissivity of 0.1-0.3 unless painted).
- On-die temperature sensors read through the LPDDR MPC or the SoC thermal registers. These give the true junction temperature and are the most accurate measurement.
- Differential scanning or lock-in thermography for high-resolution hot spot localisation.

The measurement is performed under multiple workloads: idle, typical load, peak sustained load, and worst-case synthetic benchmark. For each workload, the junction temperature is recorded at steady state (after 10-30 minutes) and compared against the simulation prediction. A simulation-to-measurement discrepancy of more than 5-10C indicates a model error and should be investigated.

Reliability testing: accelerated temperature-cycling tests (for example JESD22-A104, -55C to +125C for 500-1000 cycles) verify that the solder joints and package do not fail under thermal stress. For automotive and industrial, longer cycling and higher temperature are required (AEC-Q100 Grade 1 requires 1000 cycles at -40 to +150C).

Qualification also includes dwell testing: the PCB is held at elevated temperature (85-105C) for 168 hours or 1000 hours while running memory tests. Any bit errors during dwell indicate insufficient thermal margin or insufficient refresh handling.

The final verification is the in-system stress test: the product is placed in its worst-case enclosure at the worst-case ambient, running a production workload, for an extended period. If the LPDDR throttles, the thermal design is insufficient and must be revised before production release.

---

See also:
- [PCB Stackup Choice](pcb_stackup_choice.md)
- [PCB Power Integrity](pcb_power_integrity.md)
- [PoP and Package Design](../06_package_and_system/pop_and_package_design.md)
