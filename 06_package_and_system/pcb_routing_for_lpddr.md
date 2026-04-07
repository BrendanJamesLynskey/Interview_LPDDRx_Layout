# PCB Routing for LPDDR

This section covers PCB-level routing for LPDDR interfaces, applicable to non-PoP configurations and to the PoP-to-board interface.

---

### Q1. When is PCB routing required for LPDDR, and what are the key challenges?

**Answer:**

PCB routing for LPDDR is required in non-PoP configurations (discrete DRAM packages on the PCB), commonly found in automotive, embedded, and large-form-factor applications. Even in PoP configurations, some PCB-level routing is needed for the VDDQ power supply from the PMIC to the SoC package, decoupling capacitors placed near the BGA, and reference resistor routing (ZQ calibration resistor). In non-PoP, the full LPDDR signal path runs on the PCB from the SoC BGA to the DRAM BGA, typically 15-40 mm of controlled-impedance routing. Key challenges include maintaining 50-ohm impedance across via transitions and trace routing, managing insertion loss at high frequencies (loss increases with trace length and frequency), length matching within byte lanes on PCB (larger absolute differences require more careful routing), BGA fanout routing (escaping signals from the dense BGA ball field), and via transitions that create impedance discontinuities and stubs.

---

### Q2. How is controlled impedance achieved on PCB traces for LPDDR?

**Answer:**

Controlled impedance on PCB traces is achieved through careful stackup design and trace geometry. The PCB stackup defines the dielectric material, layer thickness, and copper weight for each layer. For LPDDR signals, the target impedance is typically 50 ohm single-ended and 100 ohm differential. The trace width required for 50-ohm impedance depends on the dielectric thickness and dielectric constant. For a typical FR-4 PCB (Er = 4.2) with 100 um (4 mil) dielectric thickness, the trace width for 50 ohm microstrip is approximately 100-125 um (4-5 mil). For stripline (trace between two ground planes), the width is narrower, typically 75-100 um. Impedance calculators or 2D field solvers determine the exact width for the specific stackup. The impedance tolerance is typically plus or minus 10% for LPDDR (45-55 ohm), achieved through PCB manufacturing process control. Critical impedance control requires impedance testing coupons on the PCB panel. For differential pairs (CK, WCK, DQS), the coupling between the two traces affects the differential impedance. Tightly coupled pairs (small spacing) have lower differential impedance; loosely coupled pairs approach twice the single-ended impedance. The target 100-ohm differential impedance requires the spacing to be chosen based on the trace width and dielectric.

---

### Q3. What is BGA fanout routing, and how does it affect LPDDR signal quality?

**Answer:**

BGA fanout routing is the process of routing signals from the BGA ball pads on the PCB to the wider-spaced routing channels between packages. For a BGA with 0.5-0.8 mm ball pitch, the balls in the inner rows cannot be routed directly to the surface layer and must escape through vias to inner layers. The fanout strategy determines how many routing layers are needed, the via type and placement, the trace lengths in the fanout region, and the signal integrity of the transition from ball to trace.

Common fanout strategies include dog-bone fanout (each ball has a short trace to a via placed between balls), which is simple but creates stubs and consumes routing space. Via-in-pad (the via is placed directly in the ball pad) eliminates the stub but requires via filling and planarisation, adding cost. Staggered fanout routes alternate rows on different layers, reducing congestion.

For LPDDR, the fanout region is critical for signal integrity because the via transition from the ball pad to the inner routing layer creates an impedance discontinuity. The via stub (the portion of the via extending beyond the routing layer) acts as an open-ended transmission line that resonates at a frequency determined by the stub length. For a 1 mm via stub in FR-4, the resonance frequency is approximately 30-40 GHz, well above LPDDR5 frequencies, but shorter stubs (0.3-0.5 mm) can resonate at 10-15 GHz, which may affect the highest harmonics.

Back-drilling removes the via stub by drilling out the unused portion of the via. This is common for high-speed designs and should be used for LPDDR5X and beyond.

---

### Q4. How does the PCB stackup design affect LPDDR signal integrity?

**Answer:**

The PCB stackup determines the layer arrangement, dielectric properties, and copper distribution that affect impedance, coupling, and loss for LPDDR signals. A typical PCB for LPDDR routing has 6-10 layers. A good stackup for LPDDR provides dedicated routing layers for LPDDR signals (separate from other high-speed interfaces), continuous ground planes adjacent to every signal layer (for return current and shielding), controlled dielectric thickness for impedance targets, and power planes for VDDQ distribution with decoupling.

An example 8-layer stackup for LPDDR includes: Layer 1 (signal, microstrip for LPDDR DQ/DQS), Layer 2 (ground plane), Layer 3 (signal, stripline for LPDDR CA/CK), Layer 4 (power plane for VDDQ), Layer 5 (ground plane), Layer 6 (signal, stripline for other interfaces), Layer 7 (ground plane), Layer 8 (signal/power).

The dielectric material affects loss. Standard FR-4 (Df = 0.02 at 1 GHz) is adequate for LPDDR4/4X but may be marginal for LPDDR5 at 6400 MT/s. Low-loss materials (Megtron 6, Df = 0.004) may be needed for LPDDR5X at 8533 MT/s on longer PCB traces. The dielectric constant (Dk) affects impedance and propagation velocity. Higher Dk requires narrower traces for the same impedance but increases capacitive loss.

For layout engineers designing the SoC, the PCB stackup is typically defined by the system/board design team. However, the SoC layout must provide a bump map and PHY design that is compatible with the target PCB technology.

---

### Q5. How are vias managed on the PCB for LPDDR routing?

**Answer:**

Vias on the PCB are vertical connections between layers that allow signal routing to transition between routing layers. For LPDDR, vias create impedance discontinuities that must be managed. Via types include through-hole vias (drill through the entire PCB, simplest and lowest cost but creates the longest stub), blind vias (drill from one surface to an inner layer, reducing stub length), buried vias (connect two inner layers without reaching the surface), and microvias (laser-drilled, small diameter 75-150 um, connecting adjacent layers only).

For LPDDR signal routing, the via transition adds parasitic capacitance (0.3-1.0 pF per via, depending on pad size and anti-pad dimensions) and parasitic inductance (50-200 pH per via, depending on via length and nearby ground vias). The capacitance creates a low-impedance point, and the inductance creates a high-impedance point, resulting in an impedance dip-then-peak at the via location.

Via optimization for LPDDR includes placing ground vias adjacent to every signal via (to provide a local return current path and reduce the inductive loop area), minimising the signal via pad size (to reduce capacitance), using anti-pad optimization (adjusting the clearance hole in the ground plane to tune the via impedance), and back-drilling or using blind vias to eliminate stubs.

For differential pairs (CK, WCK, DQS), both vias of the pair should be placed with the same geometry and spacing to maintain differential impedance through the transition.

---

### Q6. What length matching rules apply to PCB routing for LPDDR?

**Answer:**

PCB length matching follows the same principles as on-die matching but with different absolute tolerances due to the different propagation environment. The key matching groups are intra-byte DQ-to-DQS (all 8 DQ within plus or minus 0.5-1.0 mm of DQS, corresponding to approximately plus or minus 7-15 ps at PCB propagation velocities of approximately 6-7 ps/mm), CK differential pair matching (CK to CK_n within plus or minus 0.1 mm for duty cycle preservation), CA-to-CK (all CA signals within plus or minus 1-2 mm of CK), and inter-byte matching (byte lanes within plus or minus 5-10 mm, compensated by training).

PCB length matching uses serpentine routing (meanders) to add length to shorter traces, with minimum serpentine pitch of 3x trace width to avoid self-coupling. The serpentine should be placed at the trace endpoint (near the receiver) rather than in the middle.

The PCB routing environment introduces additional matching challenges not present on-die. Different trace layers have different propagation velocities (microstrip versus stripline), so traces that transition between layers may need length adjustment. Via transitions add delay that must be matched across the byte. The trace environment (nearby copper, ground plane proximity) affects the propagation velocity.

---

### Q7. How is stub minimisation achieved in PCB LPDDR routing?

**Answer:**

Stubs are unterminated trace segments that create resonances at frequencies where the stub length equals one-quarter wavelength. In PCB LPDDR routing, stubs arise from via transitions (the unused portion of a through-hole via below or above the routing layer), T-junctions (if a trace branches to two destinations, each branch sees the other as a stub), and test point connections (if test points are tapped off the main trace).

For LPDDR5 at 6400 MT/s, a stub of 5 mm in FR-4 resonates at approximately 7.5 GHz. While this is above the Nyquist frequency (3.2 GHz), the resonance still affects the third and higher harmonics of the data signal, reducing edge rates and closing the eye.

Stub minimisation techniques include back-drilling of through-hole vias to remove the unused stub portion (reduces via stub to less than 0.2 mm), using blind or buried vias instead of through-hole vias, routing to the via directly without a T-junction (each signal has a single, point-to-point path), avoiding test point taps on critical LPDDR signals (or using high-impedance probing techniques that do not load the trace), and using laser-drilled microvias for layer transitions in the BGA fanout region.

For layout, the stub length should be analysed for every via transition in the LPDDR path. The maximum allowable stub length depends on the data rate: for LPDDR5, stubs should be less than 5 mm (preferably less than 2 mm). For LPDDR5X at 8533 MT/s, stubs should be less than 3 mm (preferably less than 1 mm).

---

### Q8. How does PCB routing for LPDDR differ between mobile and automotive applications?

**Answer:**

Mobile applications primarily use PoP, which eliminates most PCB routing for LPDDR signals. The PCB carries only the power supply routing and the ZQ reference resistor connection. The routing constraints are minimal, focusing on power integrity.

Automotive applications often use discrete LPDDR packages on the PCB (non-PoP) due to reliability requirements for the PoP solder joints under automotive temperature ranges (-40 to +150C) and vibration. The PCB must carry all LPDDR signals, creating significant routing challenges. Automotive PCBs may use different materials that can withstand higher temperatures, and the trace lengths may be longer due to larger board form factors. The extended temperature range widens the impedance variation due to CTE effects on trace geometry. The automotive reliability requirements (AEC-Q100) demand more conservative design margins.

High-performance computing (HPC) and server applications may use LPDDR in configurations with multiple DRAM packages per channel (not common for LPDDR but sometimes used for bandwidth). This creates a multi-drop topology with routing to multiple receivers, which is fundamentally different from the point-to-point PoP topology.

---

### Q9. How are LPDDR power supply traces routed on the PCB?

**Answer:**

The LPDDR power supply routing on the PCB connects the PMIC (Power Management IC) output to the SoC package VDDQ, VDD2, and VDD1 power balls. Even in PoP configurations, this routing is on the PCB and is critical for power integrity.

VDDQ routing should use a dedicated power plane or wide traces with low impedance. The trace width should be sufficient to carry the worst-case VDDQ current without excessive IR drop. For 500 mA total VDDQ current and a maximum IR drop of 10 mV, the resistance budget is 20 mohm, requiring wide traces (multiple millimetres) or a dedicated plane. Decoupling capacitors should be placed along the VDDQ path, as close to the SoC BGA as possible. Typical PCB decoupling includes bulk capacitors (10-22 uF ceramic) placed within 5 mm of the BGA for low-frequency decoupling and smaller capacitors (100 nF to 1 uF) placed within 1-2 mm of the BGA for mid-frequency decoupling.

The VDDQ power plane should have a clean ground return (a ground plane on an adjacent layer) to maintain low impedance at high frequencies. The plane should not be shared with other noisy power domains.

---

### Q10. What simulation and verification tools are used for PCB-level LPDDR design?

**Answer:**

PCB-level LPDDR design uses several simulation and verification tools. For impedance calculation, 2D field solvers (Polar Instruments Si9000, Cadence Sigrity PowerSI) calculate the trace impedance for a given stackup and geometry. These are used during stackup design to determine trace widths. For SI simulation, ANSYS SIwave or Cadence Sigrity extract S-parameters from the PCB layout, capturing the frequency-dependent response of the routing, vias, and planes. The extracted models are combined with the SoC PHY model and the DRAM model for time-domain simulation. For PI simulation, Cadence Sigrity PowerDC or ANSYS SIwave PDN analyse the power distribution network on the PCB, checking DC IR drop and AC impedance. For length matching, the PCB layout tool (Cadence Allegro, Mentor Xpedition, or Altium Designer) provides built-in length matching constraints and reports. For EMC analysis, ANSYS SIwave or CST Studio Suite can predict the radiated emissions from the PCB layout.

---

See also:
- [PoP and Package Design](pop_and_package_design.md)
- [LPDDR5X and Beyond](lpddr5x_and_beyond.md)
- [Worked Problem: PCB Fanout Routing](worked_problems/problem_02_pcb_fanout_routing.md)
