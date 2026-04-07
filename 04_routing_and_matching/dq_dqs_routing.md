# DQ DQS Routing

This section covers the routing methodology for DQ (data) and DQS (data strobe) signals in LPDDRx PHY layouts. DQ/DQS routing is the most timing-critical aspect of the layout because the intra-byte skew directly determines the available timing margin.

---

### Q1. What is the byte lane grouping concept, and why is it fundamental to DQ/DQS routing?

**Answer:**

A byte lane is a group of 8 DQ signals plus one DQS differential pair that form a single timing domain. All DQ bits within the byte lane are sampled using the same DQS strobe, which means the timing relationship between each DQ bit and DQS must be tightly controlled. The byte lane is the atomic unit of data transfer in LPDDR5, corresponding to one 8-bit channel.

The byte lane grouping concept means that all routing decisions (layer assignment, via transitions, spacing, shielding, length matching) are made at the byte lane level. The 8 DQ signals and the DQS pair within a byte must be treated as a group with matched characteristics. Cross-byte matching (between different byte lanes) is much less critical because each byte has its own DQS timing reference.

Physically, the byte lane routing should occupy a compact, contiguous region of the die. The 8 DQ traces and the DQS pair should run in parallel, on the same metal layer where possible, with the same routing topology (same number of vias, same layer transitions, same shielding). Any deviation from this matched topology introduces skew that must be compensated by per-bit deskew or that reduces timing margin.

The DQS pair is typically routed at the centre of the DQ group so that the maximum distance from DQS to any DQ is minimised. This central placement also provides a symmetric coupling environment: DQ bits on either side of DQS experience similar crosstalk from DQS transitions.

For LPDDR5 with 4 channels of 8 bits each, there are 4 independent byte lanes. Each byte lane is routed independently, with tight intra-byte matching but relaxed inter-byte matching requirements.

---

### Q2. What are the intra-byte length matching requirements for DQ-to-DQS, and how are they achieved?

**Answer:**

The intra-byte length matching requirement ensures that all 8 DQ signals and the DQS pair within a byte lane have the same routing delay. The JEDEC specification defines this through the tDQS2DQ parameter, which must be less than plus or minus 200 ps total (including DRAM, package, and SoC contributions). The SoC layout allocation is typically 30-60 ps, which at approximately 7 ps/mm propagation velocity on intermediate metal layers corresponds to plus or minus 4-8 mm of length matching tolerance.

However, for practical design, layout engineers target much tighter matching than the specification requires, typically plus or minus 50-100 um within the byte lane. This translates to approximately plus or minus 7-15 ps of delay matching, leaving substantial margin for the DRAM and package contributions.

Length matching is achieved through several techniques. Routing all signals on the same metal layer eliminates layer-to-layer velocity differences (different metal layers may have different effective propagation velocities due to different dielectric environments). If layer changes are necessary, all signals in the byte must make the same transitions.

Serpentine (meander) tuning adds length to shorter traces. The serpentine is added near the end of the trace (close to the IO cell or bump) where it has the least impact on signal integrity. The serpentine pitch should be at least 3 times the trace width to avoid self-coupling that would alter the effective impedance.

Via matching ensures all signals have the same number of via transitions. Each via adds approximately 5-15 ps of delay, so a single unmatched via can consume a significant portion of the matching budget.

The DQS pair is often used as the length reference. The DQS pair is routed first (as the longest or most constrained route), and then all DQ traces are matched to the DQS length.

---

### Q3. How is shielding implemented for DQ/DQS routing, and when is it necessary?

**Answer:**

Shielding involves placing grounded (VSS-connected) traces adjacent to or between signal traces to reduce capacitive and inductive coupling (crosstalk). For LPDDR5 DQ/DQS routing, shielding may be implemented in several ways.

Lateral shielding places VSS traces on both sides of a signal trace on the same metal layer. This reduces coupling to adjacent signals on the same layer. The shield traces should be connected to VSS through vias at regular intervals (every 50-200 um) to maintain them at ground potential at all frequencies of interest.

Vertical shielding uses a ground plane (solid or meshed VSS metal) on the layer above and/or below the signal routing layer. This reduces coupling to signals on adjacent layers and provides a well-defined return current path. In LPDDR PHY layouts, the power grid mesh on adjacent layers often serves as vertical shielding.

Coaxial shielding (lateral + vertical) provides the maximum isolation by surrounding the signal trace with ground on all sides. This is typically used only for the most sensitive signals (DQS, CK, WCK) where crosstalk must be minimised.

Shielding is necessary when signal spacing is tight (less than 2-3 trace widths), when aggressive coupling between adjacent bytes or signal groups would exceed the crosstalk budget, when the DQS pair runs near high-speed aggressor signals (such as DQ bits from an adjacent byte), and when SI simulation shows that crosstalk-induced jitter exceeds the allocated budget.

For layout, shielding consumes routing resources: lateral shields occupy one trace width plus spacing on each side, effectively tripling the routing pitch. Vertical shielding on dedicated layers reduces the available layers for signal routing. The layout engineer must balance shielding effectiveness against routing density, applying shielding selectively to the most sensitive signal pairs and regions with the highest coupling risk.

---

### Q4. How does via minimisation contribute to DQ/DQS signal integrity?

**Answer:**

Vias (vertical connections between metal layers) introduce parasitic effects that degrade signal integrity. Each via adds parasitic resistance (5-50 mohm depending on via size and type), parasitic capacitance (0.5-2 fF, creating an impedance discontinuity), parasitic inductance (10-50 pH, resonating with the via capacitance at high frequencies), and delay (5-15 ps per via transition). These parasitics cause impedance discontinuities that create reflections and ISI.

At LPDDR5 data rates (6400 MT/s, Nyquist at 3.2 GHz), via discontinuities are significant. A single via transition creates a reflection coefficient of approximately 2-5%, which may seem small but becomes problematic when multiple vias accumulate. For a route with 4 via transitions, the total reflection energy can reduce the eye opening by 10-20% of the ideal value.

Via minimisation strategies include routing DQ/DQS on a single metal layer from the IO cell to the bump pad, avoiding layer transitions entirely. When layer changes are required (due to routing congestion or crossing other signal groups), all signals in the byte lane should transition together at the same location, maintaining matched via count and matched impedance environment.

Via stubs (unused portions of metal on the departure layer) should be minimised. When a trace transitions from M8 to M9, the short stub remaining on M8 acts as an open-ended transmission line that creates resonance at the frequency where the stub length equals one quarter of the wavelength. For a 100 um stub, this resonance occurs at approximately 400 GHz (well above the LPDDR5 frequency range and generally not a concern for on-die routing), but for longer stubs in the package, this can be problematic.

Anti-pads (holes in the reference plane around via locations) also contribute to impedance discontinuity. The layout should minimise anti-pad size by using the smallest via and pad dimensions allowed by DRC rules.

---

### Q5. What are the crosstalk concerns between DQ signals within a byte, and between bytes?

**Answer:**

Crosstalk is the unwanted coupling of signal energy from an aggressor trace to a victim trace through capacitive (electric field) and inductive (magnetic field) coupling. In LPDDR5, crosstalk affects both timing (crosstalk-induced jitter) and voltage (crosstalk-induced noise) margins.

Intra-byte crosstalk (between DQ signals within the same byte) is present but partially compensated by the common DQS timing reference. Since all DQ bits and DQS share the same coupling environment, the crosstalk-induced timing shift on DQ is partially correlated with the shift on DQS. The differential sampling (DQ sampled by DQS) cancels the common-mode component of the crosstalk. However, the differential component (different coupling to different DQ bits due to position) remains uncancelled and contributes to tDQS2DQ skew.

Inter-byte crosstalk (between DQ/DQS of adjacent bytes) is more problematic because the timing domains are independent. Crosstalk from an adjacent byte's DQ signal onto the victim byte's DQS creates jitter that is not correlated with the victim byte's DQ data, directly reducing the eye opening.

The crosstalk magnitude depends on several factors. Coupling length is the distance over which the aggressor and victim run in parallel -- longer parallel runs create more coupling. Spacing is the centre-to-centre distance between traces -- capacitive coupling decreases approximately as 1/distance, and inductive coupling decreases more slowly. Rise time determines the bandwidth of the crosstalk: faster transitions couple more energy, especially at the near end. Data pattern determines whether the crosstalk is additive or subtractive: a worst-case pattern (aggressor transitions while victim is at its threshold) maximises the timing impact.

For layout, inter-byte spacing should be at least 3-5 times the intra-byte spacing, or a ground shield should separate adjacent bytes. The DQS pair should be positioned to maximise its distance from adjacent byte signals. SI simulation should be used to quantify crosstalk for the specific routing geometry and verify compliance with the crosstalk jitter budget.

---

### Q6. What metal layers are typically used for DQ/DQS routing in advanced process nodes?

**Answer:**

In advanced process nodes (5nm, 7nm), the metal stack typically has 12-15 metal layers with varying thickness and pitch. The choice of routing layer for DQ/DQS depends on the signal integrity requirements, routing density, and power grid layer assignment.

Lower metal layers (M1-M4) have the finest pitch (20-40 nm) and highest resistance. They are used primarily for standard cell internal routing and local connections. They are not suitable for DQ/DQS routing due to high loss and limited width.

Intermediate metal layers (M5-M9) have wider pitch (40-100 nm) and moderate resistance. They are the primary candidates for DQ/DQS routing. These layers offer sufficient width for controlled-impedance traces (typically 0.5-2 um wide) and have reasonable loss characteristics. The dielectric constant and metal thickness on these layers determine the achievable impedance and propagation velocity.

Upper metal layers (M10-M13) have the widest pitch (100-400 nm) and lowest resistance. They are typically reserved for power grid (VDDQ, VSS straps), clock distribution, and global routing. Using upper layers for DQ/DQS is possible but may conflict with power grid needs.

Top metal layers (M14-M15, if present) are very thick (2-3 um) and used for power, inductors, and bump pads. They are not used for signal routing.

The optimal layer for DQ/DQS is typically M7-M9, providing a balance between routing pitch, resistance, and separation from the power grid. The DQ traces within a byte should all be on the same layer to ensure matched propagation velocity. If the routing crosses other signal groups (CA, CK), it may need to transition to a different layer briefly, with all byte signals transitioning together.

The layer assignment should be coordinated with the power grid design. If M10-M12 are used for VDDQ/VSS power straps, they provide vertical shielding for DQ signals on M7-M9, which is beneficial for signal integrity.

---

### Q7. How is impedance control achieved for on-die DQ/DQS routing?

**Answer:**

On-die transmission line impedance is determined by the trace geometry (width, thickness, spacing from reference planes) and the dielectric properties of the interlayer dielectric (ILD). For LPDDR5, the target DQ impedance is typically 40-50 ohm for a single-ended trace referenced to a ground plane.

The impedance is calculated using 2D electromagnetic field solvers (such as Cadence QRC, Synopsys StarRC, or Mentor Calibre) that model the actual metal cross-section. For a microstrip configuration (trace on one metal layer, reference plane on the adjacent layer), the impedance depends on the trace width (W), trace thickness (T), height above reference plane (H, determined by the dielectric thickness), and dielectric constant (Er, typically 3.0-4.5 for SiO2-based ILD in advanced nodes).

A typical calculation for an intermediate metal layer: with W = 1.0 um, T = 0.2 um, H = 0.3 um, and Er = 3.5:

```
Z0 ~ (60 / sqrt(Er)) * ln(2H / (0.8W + T))
   ~ (60 / 1.87) * ln(0.6 / 1.0)
```

This simplified formula is approximate; actual impedance is computed by field solvers with the exact process stack geometry.

For layout, impedance control requires maintaining consistent trace width along the entire DQ route. Any width variation (tapering, widening at via pads, narrowing at congestion points) creates impedance discontinuities that generate reflections. The trace width should be constant within the manufacturing tolerance of the process.

The proximity to other metal (adjacent signal traces, power straps on the same or adjacent layers) affects the impedance through coupling. Traces that are tightly spaced have lower impedance (odd-mode impedance) than isolated traces. The layout must account for this loading effect, potentially adjusting trace width in coupled regions.

The reference plane quality affects impedance. If the ground plane on the adjacent layer has slots or gaps (from power grid routing), the impedance increases locally and the return current path is disrupted. The layout should ensure a continuous reference plane under the DQ routing.

---

### Q8. How are DQ/DQS signals routed in the package substrate for PoP?

**Answer:**

After the DQ/DQS signals exit the SoC die through flip-chip bumps, they are routed through the SoC package substrate to the TMVs (Through-Mold Vias), then through the DRAM package substrate to the DRAM die bumps. Each segment has different routing characteristics.

The SoC package substrate typically uses 2-4 metal layers with 2/2 um or 3/3 um (line/space) design rules. The substrate dielectric is a resin-based laminate with a dielectric constant of 3.2-3.8. The trace width for 50-ohm impedance is typically 15-25 um, depending on the dielectric thickness and stack configuration.

The package routing must manage several challenges. Routing from the bump field to the TMV locations requires traces to fan out from the closely spaced bumps (0.4 mm pitch) to the TMV locations at the package periphery. This fanout region is routing-dense and may require multiple layers. Layer transitions in the package substrate add parasitic discontinuities similar to on-die vias but with larger parasitics (package vias are 50-100 um diameter with 10-30 pH inductance). Length matching within the byte lane must be maintained through the package routing, which may require serpentine tuning on the package substrate.

The TMVs themselves have significant parasitics (100-200 pH inductance, 50-100 fF capacitance per via) and must be modelled in SI simulation. Each DQ signal passes through one TMV, so the parasitics are consistent across the byte (contributing common-mode delay rather than differential skew).

The DRAM package substrate routing is typically shorter (the DRAM die is small) but may include wire-bond pad connections if the DRAM is wire-bonded rather than flip-chip. Wire bonds add significant inductance (0.5-2 nH per bond) and are a major SI concern at high data rates.

For the layout engineer, the package routing is typically handled by the package design team, but the silicon layout engineer must provide constraints: the bump-to-TMV routing must maintain the length matching achieved on-die, and the parasitic budget must be shared between the die and package to ensure the total path meets the tDQS2DQ specification.

---

### Q9. What is the impact of process variation on DQ/DQS routing, and how is it managed?

**Answer:**

Process variation in semiconductor manufacturing causes the actual metal dimensions to differ from the drawn dimensions. For DQ/DQS routing, the key variations are metal width variation (typically plus or minus 5-10% of the drawn width), metal thickness variation (plus or minus 5-10%), dielectric thickness variation (plus or minus 5-10%), and resistivity variation (plus or minus 5-10%).

These variations affect the routing characteristics in several ways. Impedance variation occurs because a wider-than-drawn trace has lower impedance, and a thinner dielectric increases capacitance, also lowering impedance. The typical impedance variation due to process is plus or minus 10-15%, which means a 50-ohm design may range from 42 to 58 ohm across the process window.

Propagation delay variation occurs because the propagation velocity depends on the dielectric constant and the effective inductance/capacitance per unit length. Process variation causes plus or minus 5-10% variation in delay, which translates to timing uncertainty that consumes part of the timing budget.

Intra-byte skew from process variation is usually small because all traces within a byte lane are in close physical proximity and experience the same systematic process variation. The random variation (within-die, local mismatch) is typically 1-3% of the propagation delay, which for a 1 mm route is approximately 1-2 ps of random skew.

Inter-byte skew from process variation can be larger because byte lanes may be separated by millimetres on the die, and process gradients (systematic variation across the die) create different conditions for different byte lanes. This systematic skew is compensated by per-byte write leveling and read leveling training.

For layout, process variation is managed by designing with guard bands (matching length tolerance = specification minus process-induced skew), by performing Monte Carlo simulation during SI analysis (varying the metal geometry within the process window and checking that all corners meet the specification), and by relying on the PHY training algorithms to compensate for systematic variations.

---

### Q10. How are DQ/DQS routing quality checks performed during the layout review?

**Answer:**

DQ/DQS routing quality checks are performed throughout the layout process to ensure compliance with signal integrity requirements. These checks span multiple verification stages.

DRC (Design Rule Checking) verifies that the routing complies with the process-specific design rules: minimum width, minimum spacing, minimum via enclosure, and maximum via spacing. For DQ/DQS, additional custom DRC rules may be defined: minimum trace width for impedance control, minimum spacing between DQ and adjacent signals for crosstalk control, and maximum stub length at via transitions.

Length matching verification measures the total route length of each signal in the byte lane and reports the maximum deviation from the reference (DQS) length. This check can be performed using EDA tools (Cadence Innovus, Synopsys ICC2) that support group-based length matching constraints. The report should show each DQ bit's length, the DQS length, and the delta.

Via count matching verification checks that all signals in the byte have the same number of via transitions on each layer. This is a manual or script-based check, as standard DRC does not cover it.

Impedance verification uses parasitic extraction (Cadence QRC or Synopsys StarRC) to extract the routing parasitics and calculate the impedance profile along each trace. The impedance should be within plus or minus 10% of the target (40-50 ohm) along the entire length.

Crosstalk simulation uses extracted coupling capacitances and inductances to model the crosstalk between adjacent signals. This is performed using SI simulation tools (Cadence Sigrity, Synopsys HSPICE, Keysight ADS) with aggressor-victim analysis. The crosstalk-induced jitter and voltage noise are compared against the allocated budget.

Eye diagram simulation overlays multiple bit transitions to create the data eye at the receiver, including all extracted parasitics. The eye height and width must meet the JEDEC specification (tDS, tDH, VDIVmin). This is the most comprehensive check and should be performed for worst-case data patterns and PVT corners.

---

See also:
- [CA CK Routing](ca_ck_routing.md)
- [Length Matching and Skew](length_matching_and_skew.md)
- [SI for LPDDR](../05_signal_and_power_integrity/si_for_lpddr.md)
- [Worked Problem: Byte Lane Routing](worked_problems/problem_01_byte_lane_routing.md)
