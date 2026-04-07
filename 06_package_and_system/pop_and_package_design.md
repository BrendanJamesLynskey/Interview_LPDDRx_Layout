# PoP and Package Design

This section covers Package-on-Package (PoP) technology and package design considerations for LPDDRx interfaces. PoP is the dominant packaging approach for mobile LPDDR, with the DRAM package stacked directly on top of the SoC package.

---

### Q1. What is Package-on-Package (PoP) technology, and why is it used for LPDDR?

**Answer:**

Package-on-Package (PoP) is a 3D packaging technology where two packages are vertically stacked, with the bottom package (typically the SoC) and the top package (typically the DRAM) connected through Through-Mold Vias (TMVs) or other vertical interconnects. The bottom package is soldered to the PCB through BGA (Ball Grid Array) balls, and the top package is soldered to the top surface of the bottom package through a second set of solder balls that connect to the TMVs.

PoP is used for LPDDR because it minimises the interconnect length between the SoC and DRAM. The signal path through a PoP stack is typically 2-5 mm total (die bumps to package substrate to TMVs to DRAM substrate to DRAM die), compared to 20-50 mm for discrete packages on a PCB. The shorter path reduces signal degradation, enabling higher data rates with simpler signaling (no equalization needed for LPDDR5 at 6400 MT/s). PoP reduces board area because the DRAM is stacked vertically rather than placed beside the SoC, saving valuable PCB real estate in compact mobile devices. PoP simplifies PCB routing by eliminating the need for controlled-impedance LPDDR traces on the PCB. The LPDDR signals are entirely within the package stack. PoP allows flexible memory configurations because different DRAM packages (different densities, different vendors) can be combined with the same SoC package, enabling multiple product SKUs from a single SoC design.

---

### Q2. What are Through-Mold Vias (TMVs), and what are their electrical characteristics?

**Answer:**

Through-Mold Vias (TMVs) are vertical copper interconnects that pass through the mold compound (epoxy resin) that encapsulates the SoC die in the bottom package. TMVs provide the electrical connection between the top surface of the bottom package (where the DRAM solder balls land) and the routing layers within the bottom package substrate.

TMV fabrication involves drilling holes through the mold compound after it has been applied, then plating or filling the holes with copper. The TMV diameter is typically 100-200 um, with a pitch of 400-500 um. The TMV height depends on the mold compound thickness, typically 200-400 um.

Electrical characteristics of TMVs include resistance of 10-50 mohm per via (depending on diameter and height, low due to copper fill), inductance of 50-150 pH per via (the dominant parasitic, determined by the via height and the distance to the nearest ground via), and capacitance of 50-100 fF per via (from the via-to-mold interface and from coupling to adjacent TMVs).

The TMV inductance is significant for both SI and PI. For signal integrity, the TMV inductance creates an impedance discontinuity (TMV impedance is higher than the 50-ohm transmission line), causing reflections that contribute to ISI. For power integrity, the TMV inductance is in the current path from the VDDQ regulator (on the PCB) to the DRAM and SoC IO cells, contributing to Ldi/dt noise.

For layout engineers, the TMV positions are determined by the package design team but must be coordinated with the SoC bump map. The SoC bumps that connect to TMV-routed signals must be positioned to minimise the substrate routing length between the bump and the TMV. This is a key constraint in the bump map design process.

---

### Q3. How is the SoC bottom package substrate designed for LPDDR PoP?

**Answer:**

The SoC bottom package substrate is a multi-layer organic laminate that provides routing between the SoC die flip-chip bumps and the external connections (PCB BGA balls on the bottom, TMV pads on the top). For LPDDR, the substrate must route all LPDDR signals from the die bumps to the TMV positions.

The substrate typically has 4-8 metal layers with 2/2 um or 3/3 um (line/space) design rules. The layer stack includes a signal routing layer directly above the die (for escape routing from the bump field), one or more ground planes (for return current paths and shielding), one or more power planes (for VDDQ, VDD2 distribution), and a signal routing layer on the top surface (for routing to TMV pads).

The LPDDR routing in the substrate must maintain controlled impedance (typically 50 ohm single-ended, 100 ohm differential), length matching within signal groups (with tolerances similar to the on-die matching), and minimum crosstalk between signal groups.

The substrate design is challenging because the routing must navigate between the closely spaced die bumps (0.1-0.15 mm pitch for flip-chip) and the wider TMV pitch (0.4-0.5 mm). This fanout requires multiple routing layers and creates bottlenecks at congestion points.

For layout engineers, the substrate design is handled by the package design team, but the SoC layout must provide a bump map that is routable within the substrate technology constraints. Close coordination between the silicon and package teams is essential. The SI simulation must include the substrate routing parasitics, which are provided by the package team as S-parameter models or SPICE subcircuits.

---

### Q4. What are the thermal challenges of PoP for LPDDR, and how are they addressed?

**Answer:**

PoP creates thermal challenges because the DRAM package sits on top of the SoC, trapping heat between the two packages. The SoC dissipates significant power (5-15W for a mobile SoC), and the DRAM adds 1-3W. The mold compound between the die and the TMVs has low thermal conductivity (0.5-1.0 W/mK), acting as a thermal insulator.

The thermal path for the SoC die goes downward through the flip-chip bumps and substrate to the PCB. The thermal path for the DRAM goes upward through the DRAM package to the ambient (or to a heat spreader if present). The TMVs provide some thermal conduction between the packages, but their total cross-sectional area is small relative to the package area.

Thermal challenges affect LPDDR performance because higher temperature increases the transistor on-resistance (degrading impedance calibration accuracy), increases leakage current (increasing static power), accelerates electromigration (reducing reliability), increases the DRAM refresh rate (reducing available bandwidth), and shifts timing parameters (requiring more frequent training).

Thermal mitigation techniques include thermal bumps in the SoC (extra bumps connected to ground, providing additional thermal paths), thermal vias in the package substrate (extra vias connected to ground planes that conduct heat to the PCB), exposed die pad on the bottom of the SoC package (a large thermal pad on the PCB directly under the die), heat spreaders on top of the DRAM package (a metal cap that spreads heat and conducts it to the board or enclosure), and intelligent power management that throttles the LPDDR data rate or the SoC performance when temperatures exceed safe limits.

For layout engineers, thermal considerations affect the power grid design (which must handle the increased leakage at high temperature) and the timing margin analysis (which must account for the temperature-dependent timing shifts).

---

### Q5. What is package warpage, and how does it affect LPDDR PoP assembly?

**Answer:**

Package warpage is the bending or bowing of the package substrate caused by CTE (Coefficient of Thermal Expansion) mismatch between the different materials in the package (silicon die, copper redistribution layers, organic substrate, mold compound, solder bumps). Warpage varies with temperature because the CTE mismatch changes with thermal expansion.

For PoP, warpage is critical because the bottom package's top surface must be flat enough for the DRAM solder balls to make reliable connections during the PoP assembly reflow process. If the warpage exceeds the solder ball collapse height, some balls may not connect (opens) or adjacent balls may bridge (shorts).

The warpage specification for PoP bottom packages is typically less than 100-150 um across the DRAM mounting area. This is challenging for large SoC packages (15-20 mm body size) because the CTE mismatch between the silicon die and the organic substrate creates significant bowing.

Warpage affects the LPDDR interface through several mechanisms. Solder joint reliability is impacted because warpage-induced stress can crack solder joints during thermal cycling, leading to intermittent or permanent opens. TMV reliability is impacted because differential expansion between the mold compound and the TMV copper can stress the TMV connections. Electrical performance is affected because warpage changes the solder ball height and thus the parasitic inductance and capacitance, potentially causing variation across the bump field.

For layout engineers, warpage does not directly affect the die-level layout, but it influences the bump map design. Bumps at the die edges (where warpage is typically greatest) may experience more stress than central bumps. Critical LPDDR signals should be routed through bumps in the lower-warpage central region when possible, while less critical power/ground bumps can be at the edges.

---

### Q6. How do PoP standards and form factors affect the PHY design?

**Answer:**

JEDEC defines standard PoP form factors that specify the package dimensions, ball map, and TMV positions for LPDDR memory. These standards ensure compatibility between SoC and DRAM packages from different vendors.

The primary PoP standards for LPDDR include the JEDEC JC-11 standards for package dimensions, the JEDEC ball map standards for LPDDR5 (within JESD209-5), and de facto industry standards for TMV positions and pitch established by major SoC vendors.

Common PoP form factors include 12mm x 12mm (for mid-range mobile SoCs), 14mm x 14mm (for high-end mobile SoCs), and 17mm x 17mm (for premium SoCs with large die). The DRAM package form factor is typically the same as the SoC package or slightly smaller.

These form factors constrain the PHY design in several ways. The bump field area is fixed by the package body size, limiting the number of bumps available for LPDDR. The TMV positions are at the package periphery, constraining the SoC bump-to-TMV routing length. The package layer count and routing capacity limit the complexity of the substrate routing.

For PHY design, the standard form factors mean that the SoC bump map and PHY placement must be designed within the constraints of the target PoP form factor. A PHY designed for a 12mm package may not fit in a 10mm package without redesigning the bump map and adjusting the PHY placement.

---

### Q7. What is the role of the redistribution layer (RDL) in LPDDR PoP?

**Answer:**

The redistribution layer (RDL) is a set of thin metal layers fabricated on the top of the SoC die (after the standard BEOL process) or on the package substrate, used to reroute the die bump positions to match the package requirements. For LPDDR PoP, the RDL provides flexibility to connect the die-level bump positions (optimised for the PHY layout) to the package-level bump positions (optimised for the PoP assembly and TMV routing).

The RDL typically uses 1-3 metal layers with 2-5 um line/space rules, fabricated using a wafer-level or panel-level process. The RDL metal is thicker than standard BEOL metal (2-5 um versus 0.1-0.5 um), providing lower resistance and higher current capacity.

For LPDDR signals, the RDL routing adds parasitic resistance (0.1-0.5 ohm per mm of RDL trace) and capacitance (50-100 fF per mm). At LPDDR5 data rates, these parasitics contribute to signal degradation and must be included in the SI model.

The RDL design must maintain controlled impedance for LPDDR signals. The trace width and spacing on the RDL layers are chosen to achieve the target impedance (50 ohm for single-ended signals, 100 ohm for differential pairs). The RDL dielectric properties (dielectric constant and loss tangent) affect the impedance and propagation velocity.

For layout engineers, the RDL provides a degree of freedom that can relax the die-level bump placement constraints. If the PHY layout cannot achieve exact bump alignment with the package TMV positions, the RDL can reroute the bumps. However, this adds parasitic cost, so minimising the RDL routing length is preferred.

---

### Q8. How does the PoP interconnect affect signal integrity compared to discrete packaging?

**Answer:**

The PoP interconnect has distinct SI characteristics compared to discrete (non-PoP) packaging where the DRAM is a separate package on the PCB.

The PoP interconnect is shorter (2-5 mm total path from SoC die to DRAM die, compared to 20-50 mm for discrete PCB routing). This shorter path means less frequency-dependent loss, fewer wavelengths at the Nyquist frequency (the channel is electrically short), and less ISI from channel loss.

However, the PoP path includes more discontinuities per unit length: the SoC flip-chip bumps, the SoC package substrate vias and traces, the TMVs, the solder joint between packages, the DRAM package substrate traces and vias, and the DRAM die connection (wire bond or flip-chip). Each discontinuity creates reflections that can overlap due to the short path length.

The short path also means that reflections return quickly. A reflection from the DRAM end of a 3 mm path returns to the SoC in approximately 40 ps (round trip at 1.5 x 10^8 m/s). At 6400 MT/s (156.25 ps UI), this reflection overlaps with the current bit or the immediately adjacent bit, creating constructive or destructive ISI.

The PoP path has limited ability to absorb reflections through loss. Unlike a long PCB trace where the loss naturally attenuates reflections, the short PoP path has minimal loss, and reflections maintain their amplitude through multiple bounces.

For SI analysis, the PoP channel requires careful 3D EM modelling of the package structures (bumps, TMVs, substrate routing), full-path S-parameter extraction, and time-domain simulation with multi-bounce reflections captured. The on-die routing contribution is a smaller fraction of the total channel degradation in PoP (compared to discrete packaging where the PCB routing dominates), but it still matters.

---

### Q9. What quality and reliability concerns are specific to LPDDR PoP?

**Answer:**

PoP introduces specific quality and reliability concerns related to the multi-package assembly process and the vertical interconnect. Solder joint fatigue from thermal cycling causes cracks in the solder balls connecting the top and bottom packages. The CTE mismatch between the packages creates cyclic stress during temperature excursions (power cycling, ambient temperature changes). The JEDEC reliability standard requires packages to survive 500-1000 thermal cycles (-40 to +125C for automotive, 0 to +100C for consumer).

TMV reliability includes concerns about copper fatigue in the TMVs, mold compound delamination around the TMVs, and stress-induced cracking at the TMV-to-substrate interface. These are tested through accelerated life testing and monitored in production through electrical continuity tests.

Assembly yield is affected by the PoP stacking process. The alignment between top and bottom packages must be within plus or minus 50-100 um. Warpage, solder ball coplanarity, and reflow profile control all affect yield. For LPDDR, a failed solder joint on any signal pin renders the entire channel non-functional.

Moisture sensitivity is a concern because the mold compound and package materials absorb moisture, which can vaporise during reflow and cause delamination ("popcorn" effect). Packages must be handled and stored according to moisture sensitivity level (MSL) ratings.

For layout engineers, reliability concerns affect the bump map design (avoiding critical signals on bumps at high-stress locations), the power grid design (sufficient redundancy that a single bump failure does not cause catastrophic power loss), and the SI margin analysis (including a reliability derating factor that accounts for degradation over the product lifetime).

---

### Q10. How is the PoP assembly process validated for LPDDR signal integrity?

**Answer:**

PoP assembly validation ensures that the manufactured PoP stack meets the signal integrity requirements of the LPDDR interface. The validation process includes pre-silicon validation using extracted models, silicon bring-up validation on first samples, and production validation on mass-produced units.

Pre-silicon validation uses EM simulation models of the package structures (extracted from the package design database using ANSYS HFSS or Cadence Sigrity). The complete channel model (SoC die + package + PoP + DRAM) is simulated to predict the eye diagram and timing margins. This validation occurs before any hardware is built and is used to approve the package design for fabrication.

Silicon bring-up validation uses the first manufactured samples to measure the actual LPDDR performance. The PHY's eye scanning capability is used to measure the data eye at the SoC receiver, and the training results are analysed to verify that the training converges with adequate margin. TDR measurements on test structures can validate the impedance profile of the PoP interconnect. S-parameter measurements on dedicated test vehicles can validate the package model accuracy.

Production validation uses production test programs that run LPDDR functional tests (reading and writing memory patterns) and check for bit errors. The test coverage must be sufficient to catch assembly defects (open/shorted solder joints, TMV failures) that would affect LPDDR functionality.

Correlation between simulation and measurement is essential. If the silicon measurements show significantly worse performance than predicted by simulation, the models must be updated, and the root cause must be investigated (possible causes include incorrect material properties in the EM model, manufacturing variation not captured in the model, or unexpected coupling effects).

---

See also:
- [PCB Routing for LPDDR](pcb_routing_for_lpddr.md)
- [LPDDR5X and Beyond](lpddr5x_and_beyond.md)
- [Bump and Ball Map Design](../03_layout_fundamentals/bump_and_ball_map_design.md)
- [Worked Problem: PoP Stackup Design](worked_problems/problem_01_pop_stackup_design.md)
