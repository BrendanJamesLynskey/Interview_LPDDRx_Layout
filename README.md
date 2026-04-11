![LPDDRx Layout](https://img.shields.io/badge/topic-LPDDRx%20layout-blue)

# Interview Preparation: LPDDRx Layout

A comprehensive interview preparation repository covering the physical design and layout of LPDDRx memory interfaces. This resource spans LPDDR4/4X through LPDDR5/5X and upcoming LPDDR6, with emphasis on PHY layout, routing, signal integrity, power integrity, and package-level design considerations relevant to silicon physical design engineers.

## Table of Contents

### 01 Foundations
Core knowledge on LPDDR standards, memory architecture, and signaling fundamentals.

- [LPDDR Evolution and Standards](01_foundations/lpddr_evolution_and_standards.md) -- Generational progression from LPDDR4 through LPDDR6, data rates, voltage scaling, and JEDEC standardisation milestones.
- [Memory Architecture Basics](01_foundations/memory_architecture_basics.md) -- Channels, ranks, banks, bank groups, prefetch and burst length, and how architecture maps to physical implementation.
- [LPDDR Signaling and Timing](01_foundations/lpddr_signaling_and_timing.md) -- Differential clocking, single-ended data/strobe/CA signaling, VDDQ levels, and critical timing parameters.
- [Worked Problems](01_foundations/worked_problems/) -- Bandwidth calculation, generation comparison, and timing margin analysis.

### 02 Physical Interface
PHY architecture, IO cell design, and termination strategies.

- [PHY Architecture](02_physical_interface/phy_architecture.md) -- DQ byte lanes, CA training blocks, ZQ calibration, DLL/PLL structures, and read/write FIFO design.
- [IO Cell Design](02_physical_interface/io_cell_design.md) -- Push-pull drivers, programmable impedance, VREF generators, slew rate control, and level shifters.
- [Termination and ODT](02_physical_interface/termination_and_odt.md) -- On-die termination modes (NT-ODT, WR-ODT, CA-ODT), park termination, and ZQ impedance calibration.
- [Worked Problems](02_physical_interface/worked_problems/) -- PHY floorplan, driver sizing, and ODT value selection.

### 03 Layout Fundamentals
Floorplanning, bump map design, and power grid construction for LPDDR PHY.

- [Floorplanning for LPDDR](03_layout_fundamentals/floorplanning_for_lpddr.md) -- PHY placement relative to the memory controller, channel orientation, abutment, and macro planning.
- [Bump and Ball Map Design](03_layout_fundamentals/bump_and_ball_map_design.md) -- JEDEC ball map standards, signal-to-power bump ratios, VDDQ/VDD2 power bump allocation.
- [Power Grid for Memory IO](03_layout_fundamentals/power_grid_for_memory_io.md) -- Dedicated VDDQ domain, VDD1/VDD2 supplies, ESD structures, and guard ring placement.
- [Worked Problems](03_layout_fundamentals/worked_problems/) -- PHY placement strategy, bump assignment, and IO power grid design.

### 04 Routing and Matching
DQ/DQS routing, CA/CK distribution, and length matching methodology.

- [DQ DQS Routing](04_routing_and_matching/dq_dqs_routing.md) -- Byte lane grouping, intra-byte matching, shielding, and via minimisation strategies.
- [CA CK Routing](04_routing_and_matching/ca_ck_routing.md) -- Command/address bus routing, differential clock pairs, symmetry, and tree structures.
- [Length Matching and Skew](04_routing_and_matching/length_matching_and_skew.md) -- Intra-byte DQ-to-DQS skew budgets, inter-byte matching, CA-to-CK skew, and serpentine tuning.
- [Worked Problems](04_routing_and_matching/worked_problems/) -- Byte lane routing, clock distribution, and skew budget analysis.

### 05 Signal and Power Integrity
SI analysis, power integrity, and noise mitigation for high-speed memory interfaces.

- [SI for LPDDR](05_signal_and_power_integrity/si_for_lpddr.md) -- Eye diagrams, ISI, reflections, impedance discontinuities, write leveling, and channel modelling.
- [Power Integrity for LPDDR](05_signal_and_power_integrity/power_integrity_for_lpddr.md) -- VDDQ ripple budgets, decoupling with MIM/MOM capacitors, IR drop analysis, and PDN design.
- [EMI and Noise Mitigation](05_signal_and_power_integrity/emi_and_noise_mitigation.md) -- SSO/SSR noise, ground bounce, timing impact, spread-spectrum clocking, and shielding.
- [Worked Problems](05_signal_and_power_integrity/worked_problems/) -- Eye diagram analysis, decoupling strategy, and SSR noise analysis.

### 06 Package and System
Package-on-Package design, PCB routing, and next-generation LPDDR.

- [PoP and Package Design](06_package_and_system/pop_and_package_design.md) -- SoC bottom package, DRAM top package, through-mold vias, substrate design, and warpage.
- [PCB Routing for LPDDR](06_package_and_system/pcb_routing_for_lpddr.md) -- Controlled impedance, via transitions, BGA fanout, stub minimisation, and stackup design.
- [LPDDR5X and Beyond](06_package_and_system/lpddr5x_and_beyond.md) -- Higher data rates, WCK:CK ratios, enhanced training, lower voltage, and LPDDR6 outlook.
- [Worked Problems](06_package_and_system/worked_problems/) -- PoP stackup design, PCB fanout routing, and LPDDR5X migration.

### 07 Quizzes
Multiple-choice quizzes covering all major topics.

- [Quiz: Foundations](07_quizzes/quiz_foundations.md)
- [Quiz: Layout](07_quizzes/quiz_layout.md)
- [Quiz: Signal Integrity](07_quizzes/quiz_signal_integrity.md)
- [Quiz: System](07_quizzes/quiz_system.md)

### 08 PCB Level Layout
Detailed PCB-level design considerations for LPDDRx interfaces, covering the full SoC-to-DRAM channel on the board.

- [PCB Stackup Choice](08_pcb_level_layout/pcb_stackup_choice.md) -- Layer count, dielectric materials, impedance targets, microstrip versus stripline, back-drill feasibility, reference-plane strategy.
- [PCB Signal Integrity](08_pcb_level_layout/pcb_signal_integrity.md) -- Channel loss budget, insertion loss, reflections, crosstalk, via stubs, fibre-weave effect, SSO noise, full-channel simulation.
- [PCB Power Integrity](08_pcb_level_layout/pcb_power_integrity.md) -- LPDDR supply rails, target impedance, decoupling hierarchy, embedded capacitance, IR drop, PMIC placement, cap de-rating, PDN integration.
- [PCB Timing](08_pcb_level_layout/pcb_timing.md) -- UI and flight-time budgets, intra-byte/inter-byte/CA-CK matching, flight-time versus physical-length matching, jitter accumulation, training algorithm interaction.
- [PCB Thermal](08_pcb_level_layout/pcb_thermal.md) -- LPDDR power dissipation, junction temperature limits, heat-conduction paths, thermal vias, copper pour, placement, thermal-electrical coupling, verification.
- [PCB Testing and Compliance](08_pcb_level_layout/pcb_testing_and_compliance.md) -- Test equipment, JEDEC compliance, training margin measurement, shmoo plots, manufacturing variation, bringup flow, DFT features, EMC.

## How to Use

1. **Sequential study** -- Work through sections 01 through 06 in order to build knowledge from fundamentals to system-level concerns.
2. **Worked problems** -- After each concept section, attempt the worked problems before reading the solutions.
3. **Quizzes** -- Use the quizzes in section 07 for self-assessment. Cover the answer key and attempt all questions first.
4. **Cross-references** -- Follow the relative links between files to reinforce connections across topics.
5. **Interview prep** -- Focus on sections most relevant to your target role: layout engineers should prioritise sections 03 and 04, SI engineers should focus on section 05, and system/package engineers should study section 06.

## Contributing

Contributions are welcome. Please open an issue or submit a pull request if you would like to add content, correct errors, or improve explanations. Follow the existing file format and naming conventions.

## Related Repositories

- Interview preparation repositories for related physical design topics (coming soon).

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.

Last updated: 2026-04-11
