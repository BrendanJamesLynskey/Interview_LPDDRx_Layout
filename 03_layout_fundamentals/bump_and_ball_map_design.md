# Bump and Ball Map Design

This section covers the design of bump maps on the SoC die and ball maps on the DRAM package for LPDDRx interfaces. The bump/ball map is a critical interface between the silicon and package design teams and directly constrains the PHY layout.

---

### Q1. What is the JEDEC ball map standard for LPDDR5, and how does it constrain the SoC bump map?

**Answer:**

The JEDEC JESD209-5 specification defines standardised ball maps for LPDDR5 memory devices across various package configurations. The ball map specifies the exact grid position of every signal, power, and ground ball on the bottom surface of the DRAM package. For a typical x16 LPDDR5 device (two 8-bit channels), the ball map uses a grid with 0.5 mm pitch, arranged in rows and columns labelled alphabetically and numerically.

The ball map organises signals by function: each channel's DQ byte (8 DQ + DQS/DQS_n) occupies a contiguous region of the grid, with power (VDDQ) and ground (VSS) balls interspersed for return current paths. The CA signals (7 CA + CK/CK_n + CS) for each channel are grouped separately. WCK/WCK_n pairs are positioned near their associated data byte.

The SoC bump map must be designed to create a clean routing path from the SoC die to the DRAM balls through the PoP stack. In the ideal case, the SoC bumps are positioned directly below (in the PoP assembly) the corresponding DRAM balls, minimising the package routing length. However, the SoC die is flip-chip mounted, which introduces a mirror transformation between die coordinates and package coordinates.

The constraint on the SoC is that the PHY IO cell positions must map to package bump positions that align with the DRAM ball positions through the TMV (Through-Mold Via) interconnect. The package substrate routing provides some flexibility to rearrange signals between the SoC bumps and the TMVs, but this routing adds parasitic delay and impedance discontinuity that should be minimised.

For layout engineers, the JEDEC ball map is a starting point. The SoC bump map is derived from the DRAM ball map by working backwards through the package routing, accounting for the TMV positions and the flip-chip transformation. This process requires close collaboration between the silicon layout team and the package design team.

---

### Q2. What is the signal-to-power bump ratio for LPDDR interfaces, and why is it important?

**Answer:**

The signal-to-power bump ratio is the number of signal bumps divided by the number of power and ground bumps in the LPDDR interface region. For LPDDR5, a typical ratio is approximately 1:1 to 1:1.5 (signal:power), meaning that for every signal bump, there are 1 to 1.5 power or ground bumps.

This ratio is important for several reasons. Return current path quality depends on having ground bumps close to every signal bump. High-speed signals require a nearby ground return path to maintain controlled impedance and minimise loop inductance. If ground bumps are too sparse, the return current must travel further, increasing the inductive loop area and degrading signal integrity.

Power delivery capability depends on the number of VDDQ and VSS bumps. Each bump has a finite current carrying capacity (typically 50-100 mA per bump for reliability) and a finite resistance (10-50 mohm per bump). The total current demand of the LPDDR PHY must be distributed across enough bumps to keep the per-bump current within limits.

Thermal considerations also apply because each bump serves as a thermal path from the die to the package substrate. More power/ground bumps provide better thermal dissipation from the hot IO region.

The JEDEC specification defines the minimum number of power and ground balls on the DRAM side. The SoC side should match or exceed this density. In practice, SoC designs often add extra VSS bumps (at the expense of additional die area) to improve power integrity, especially in the IO region where the switching current is high.

For layout, the signal-to-power ratio determines the bump map density and pitch. A higher ratio of power bumps means fewer signal bumps per unit area, which may require a larger total bump map area. The layout engineer must work with the package team to find the optimal balance between signal routing ease and power delivery adequacy.

---

### Q3. How are VDDQ, VDD2, and VDD1 power bumps distributed in the LPDDR bump map?

**Answer:**

The LPDDR bump map must accommodate multiple voltage domains, each with its own set of power and ground bumps. The distribution strategy affects the power integrity of each domain.

VDDQ bumps (0.5V for LPDDR5) are the most critical and most numerous. They provide the supply for all IO drivers and receivers, which draw the highest transient currents. VDDQ bumps are typically distributed uniformly across the IO region, interspersed between signal bumps. A common pattern places one VDDQ and one VSS bump for every 2-4 signal bumps. The VDDQ bumps connect to the VDDQ power grid on the die, which feeds the IO cells through wide metal straps.

VDD2 bumps (1.05V for LPDDR5) power the PHY analog circuits (PLL, DLL, VREF generators) and some internal logic. VDD2 bumps are fewer in number (typically 4-8 per PHY instance) because the VDD2 current is lower (no high-speed switching). These bumps are placed near the PLL and analog blocks, away from the noisy IO region.

VDD1 bumps (1.8V) power the DRAM core logic interface and are typically minimal on the SoC side (the 1.8V domain on the SoC is usually limited to a few level shifters). Most VDD1 bumps are on the DRAM side for the DRAM core array.

VSS (ground) bumps serve as the return path for all voltage domains. They are shared between domains (all domains share the same ground reference) and are distributed throughout the bump map. Ground bumps are often the most numerous category, as every signal and power bump benefits from a nearby ground reference.

The bump distribution must ensure that the inductive loop between any VDDQ bump and the nearest VSS bump is small. The self-inductance of a bump is approximately 30-100 pH, and the mutual inductance between adjacent bumps creates the return loop. A smaller loop (closer VDDQ-VSS spacing) means lower loop inductance and less Ldi/dt noise.

---

### Q4. How does the bump pitch affect routing in the PHY and package?

**Answer:**

The bump pitch (centre-to-centre distance between adjacent bumps) directly affects the routing density and signal integrity in both the PHY die layout and the package substrate.

For PoP applications, common bump pitches are 0.4 mm (400 um) and 0.35 mm (350 um). Smaller pitches allow more bumps in a given area (higher density) but impose tighter routing constraints.

On the SoC die, the bump pitch determines the IO cell pitch. If bumps are spaced at 400 um, each IO cell must fit within approximately 400 um of die edge length (or a multiple thereof, if signal bumps are interleaved with power bumps). The routing from the IO cell to its associated bump must navigate through the bump field, which becomes more congested at smaller pitches.

The under-bump metallisation (UBM) and solder bump diameter are related to the pitch. At 400 um pitch, the bump diameter is typically 200-250 um, leaving 150-200 um between bumps for routing. At 350 um pitch, the bump diameter may be 180-200 um, leaving only 150-170 um. This space must accommodate at least one routing track (for escape routing from inner bumps) plus keepout margins.

In the package substrate, the bump pitch determines the via and trace dimensions. Package substrate technology for PoP typically uses 2/2 um (line/space) or 3/3 um rules for the redistribution layers. At 400 um pitch, there is ample space for routing in the package. At 350 um pitch, the routing density increases and may require additional package layers.

For signal integrity, the bump pitch affects crosstalk between adjacent signal bumps. At 400 um pitch with 250 um bump diameter, the edge-to-edge spacing is 150 um. The capacitive and inductive coupling between adjacent bumps depends on this spacing and the bump height (typically 50-80 um). Signal integrity simulation should model the bump-to-bump coupling to ensure adequate isolation.

---

### Q5. What is escape routing in the bump map context, and how is it planned?

**Answer:**

Escape routing refers to the routing required to connect each bump to the die-internal circuitry. In a full-area bump map (where bumps cover the entire die surface), the inner bumps are surrounded by other bumps and cannot be directly connected to the underlying circuit without routing through the bump field.

For LPDDR PHY, the bumps are typically arranged in a region near the die edge, forming a partial-area bump map. The bumps closest to the die edge (outermost row) can connect directly downward (via a simple via stack from the bump pad to the underlying metal layers). The bumps in inner rows must route outward to escape the bump field, then navigate to their destination circuits.

The escape routing strategy uses the metal layers below the bump pad layer. For each row of bumps, the escape routes fan out in a specific pattern. Outermost bumps escape directly (no lateral routing). The second row escapes between the outermost bumps. The third row routes between both outer rows. Each successive row requires longer lateral routes and more metal layers.

For LPDDR, the escape routing planning involves assigning signal bumps to positions that minimise escape length. Performance-critical signals (DQ, DQS, WCK) should be in the outermost rows for the shortest escape routes. Power and ground bumps, which are less sensitive to parasitic routing, can be placed in inner rows where escape routes are longer.

The number of metal layers available for escape routing determines the maximum number of bump rows. With 2 metal layers for escape, typically 2-3 rows of bumps can be accommodated. With 3-4 layers, 4-6 rows are possible.

For layout engineers, escape routing must be planned during the bump map definition phase. The bump assignment must consider the routing resources available on each metal layer and ensure that no layer is over-congested. EDA tools (such as Cadence Allegro or Synopsys IC Validator) can automate escape routing analysis.

---

### Q6. How do through-mold vias (TMVs) affect the bump map design for PoP?

**Answer:**

Through-Mold Vias (TMVs) are vertical interconnects that pass through the mold compound encapsulating the SoC die, connecting the top surface of the SoC package to the bottom surface of the DRAM package. TMVs are the critical link in the PoP signal path, and their characteristics directly affect the bump map design.

TMV dimensions are larger than on-die vias or package vias: typical TMV diameter is 100-200 um with a pitch of 400-500 um. The TMV pitch must match or be a multiple of the DRAM ball pitch, as the TMVs connect directly to the DRAM package balls (through solder joints on the DRAM package substrate).

TMV parasitics include resistance (10-50 mohm per via), inductance (50-150 pH per via), and capacitance (50-100 fF per via). These parasitics add to the signal path and must be included in SI simulation. The inductive component is particularly important for power integrity, as the TMV inductance contributes to the Ldi/dt noise on the VDDQ supply.

The TMV positions are constrained by the mold compound area available on the SoC package. TMVs cannot be placed directly over the SoC die (the die occupies the central area under the mold), so they must be at the periphery of the SoC package, around the die. This creates a routing challenge: signals from the SoC die bumps must route through the SoC package substrate to the TMV locations at the package periphery.

For bump map design, the TMV constraint means that the SoC bumps should be positioned near the die edge to minimise the package routing distance to the TMVs. The bump map must be designed in coordination with the TMV placement to create short, direct routing paths.

The number of TMVs available is limited by the peripheral area and the TMV pitch. For a typical PoP package, 100-200 TMVs may be available for the LPDDR interface. These must be allocated between signal, power, and ground connections. Signal TMVs carry one signal each, while power and ground TMVs carry supply current and must be numerous enough to meet the current and inductance requirements.

---

### Q7. How is the bump map designed for a dual-rank LPDDR5 configuration?

**Answer:**

A dual-rank LPDDR5 configuration has two independent sets of DRAM arrays sharing the same data bus. This means the DQ and DQS signals are shared between ranks (only one rank drives at a time), but each rank has its own CS (chip select), CK, and potentially CA signals.

The bump map for dual-rank must accommodate the additional control signals. Compared to single-rank, dual-rank adds one extra CS signal per channel (CS0 for rank 0, CS1 for rank 1). The CK and CA may be shared or duplicated, depending on the specific implementation. In the shared case, one CK and one CA bus serve both ranks, with CS differentiating the target.

On the DRAM side, dual-rank may use two stacked or side-by-side DRAM dies. In a PoP stack, the two DRAM dies might be stacked with wire bonds or TSVs (Through-Silicon Vias), or placed in separate packages. The ball map must provide connectivity to both dies.

For the SoC bump map, the dual-rank implications are modest: one additional CS bump per channel (4 extra bumps for a 4-channel x32 interface), and the routing for the additional CS signals. The data bus bumps are shared and do not increase. However, the signal integrity analysis must account for the additional loading of the second rank on the shared bus.

The layout engineer must add IO cells for the additional CS signals in the PHY. These cells are typically CA-type drivers (unidirectional, not requiring receivers). The additional bumps must be accommodated in the bump map, which may require slight rearrangement of the existing bumps.

If the CA bus is duplicated for each rank (to reduce the loading on each CA receiver), the bump count increases more significantly: 7 additional CA bumps plus CK pair per channel, totalling 36 extra bumps for a 4-channel interface. This would require expanding the bump map area or using a finer pitch.

---

### Q8. What is the role of dummy bumps and how are they used in the LPDDR bump map?

**Answer:**

Dummy bumps (also called non-functional bumps or mechanical bumps) are bumps that do not carry electrical signals. They serve mechanical and thermal purposes in the PoP assembly.

Mechanical support bumps are placed to ensure uniform pressure distribution across the die during PoP assembly and reflow. If the functional bumps are concentrated in one area (such as the LPDDR interface region along one die edge), the opposite side of the die may lack bumps, creating a mechanical imbalance that can cause tilting, cracking, or poor solder joint formation. Dummy bumps in the non-functional areas provide even support.

Thermal bumps (often connected to ground) provide additional thermal paths from the die to the package substrate. The LPDDR PHY region generates significant heat, and dummy thermal bumps can help spread this heat into the substrate and out to the board.

VSS-connected dummy bumps serve a dual purpose: they provide mechanical support and also improve the ground plane integrity in the package substrate. More ground connections mean lower impedance in the substrate ground plane, which benefits power integrity for the entire die.

For the bump map design, dummy bumps are added after the functional bumps (signal, power, ground) are placed. The dummy bump placement must not interfere with the routing in the RDL (redistribution layer) and package substrate. If a dummy bump is connected to VSS, it needs a via connection to the die's ground grid, which consumes routing resources.

The layout engineer should coordinate with the package team to determine the dummy bump requirements based on the PoP assembly process specifications. The minimum bump density (bumps per square millimetre) and the maximum gap between bumps are typically specified by the assembly house.

---

### Q9. How do bump map considerations differ between PoP and non-PoP LPDDR configurations?

**Answer:**

While PoP is the dominant packaging technology for mobile LPDDR, some applications (automotive, embedded, large-die SoCs) use non-PoP configurations where the DRAM is a separate package on the PCB. The bump map considerations differ significantly.

In PoP, the SoC bumps must align with the DRAM ball map through the TMV interconnect. The signal path is short (2-5 mm), the parasitic environment is dominated by bump and via parasitics, and the routing flexibility is limited by the TMV positions and the package substrate layer count. The bump pitch is fine (0.35-0.5 mm) to achieve high density in the compact PoP footprint.

In non-PoP (discrete) configurations, the SoC bumps connect through the SoC package to the PCB, and then through PCB traces to the DRAM package. The signal path is much longer (10-50 mm on the PCB), and the PCB routing provides significant flexibility. The bump pitch may be coarser (0.5-0.8 mm) because the package can be larger.

For the SoC bump map, non-PoP configurations allow more freedom in bump placement because there is no direct alignment constraint with the DRAM package. The PHY bumps can be placed to optimise the SoC package routing and the PHY die layout, without worrying about TMV positions. However, the longer PCB path introduces more signal integrity challenges (higher loss, more reflections, more crosstalk on the PCB), which may require stronger drivers, different termination schemes, and more careful impedance control.

The power delivery in non-PoP is different: the VDDQ supply comes from a voltage regulator on the PCB rather than from a shared package. The PCB routing for VDDQ must maintain low impedance, and the decoupling strategy includes PCB-level capacitors in addition to on-die decoupling.

For layout engineers, the key difference is that PoP designs are more constrained in bump placement but benefit from shorter signal paths, while non-PoP designs have more placement freedom but more demanding SI requirements on the longer interconnect.

---

### Q10. How is the bump map verified before tapeout?

**Answer:**

Bump map verification is a multi-disciplinary process that must be completed before the die layout is finalised (tapeout). Errors in the bump map are extremely costly because they affect both the die and the package, and cannot be fixed with a metal-only ECO.

Physical verification checks include verifying that all bump positions comply with the minimum pitch, minimum pad size, and minimum edge-to-die-edge spacing rules defined by the assembly process. The bump pattern must also satisfy the assembly house's requirements for uniformity and coplanarity.

Electrical connectivity verification ensures that every functional bump is connected to the correct net on the die (via the pad and via stack) and on the package (via the RDL and substrate routing). This is done by extracting the bump-to-die and bump-to-package netlists and comparing them against the schematic.

Signal integrity verification simulates the complete signal path (die pad to bump to RDL to substrate trace to TMV to DRAM ball) for every LPDDR signal. The simulation checks impedance continuity, reflection coefficients, crosstalk coupling, and timing. This verification uses extracted parasitic models from both the die (from Calibre or StarRC extraction) and the package (from ANSYS SIwave or Cadence Sigrity extraction).

Power integrity verification simulates the PDN from the die power grid through the bumps and package to the external supply. The verification checks DC IR drop, AC impedance (PDN resonance), and transient voltage droop. Every VDDQ, VDD2, VDD1, and VSS bump is included in the model.

Mechanical verification uses finite element analysis (FEA) to check for stress concentrations, warpage, and solder joint reliability under thermal cycling. The bump pattern must not create areas of excessive stress that could lead to solder cracking or die delamination.

The bump map verification is an iterative process. Issues found during verification (such as SI violations or PDN resonance) may require modifying the bump map, which in turn requires re-verification. The layout engineer must plan for multiple iterations and maintain a clear revision history of the bump map.

---

See also:
- [Floorplanning for LPDDR](floorplanning_for_lpddr.md)
- [Power Grid for Memory IO](power_grid_for_memory_io.md)
- [PoP and Package Design](../06_package_and_system/pop_and_package_design.md)
- [Worked Problem: Bump Assignment](worked_problems/problem_02_bump_assignment.md)
