# Length Matching and Skew

This section covers the methodology for length matching and skew control in LPDDRx layouts. Length matching is one of the most important layout tasks because it directly determines timing margin at the receiver.

---

### Q1. What are the different categories of length matching in an LPDDR5 interface?

**Answer:**

Length matching in LPDDR5 falls into several categories with different tolerances and priorities. Intra-byte DQ-to-DQS matching is the tightest requirement, ensuring all 8 DQ signals match the DQS length within the byte lane. The target is typically plus or minus 50-100 um (7-15 ps). This is the highest priority because per-bit deskew has limited range and DQS is the sampling clock for all DQ bits in the byte.

DQS differential pair matching (DQS to DQS_n) ensures the two lines of the strobe pair are equal in length for duty cycle accuracy. The target is plus or minus 5-10 um (less than 1-2 ps) because any mismatch directly creates DCD on the strobe.

CK differential pair matching (CK to CK_n) has the same tight tolerance as DQS matching for duty cycle preservation: plus or minus 5-10 um.

WCK differential pair matching (WCK to WCK_n) requires plus or minus 5-10 um for the same duty cycle reasons, but is more critical because WCK runs at a higher frequency than CK.

CA-to-CK matching ensures all 7 CA signals match CK within the setup/hold budget. The target is plus or minus 200-400 um (30-60 ps), which is more relaxed than DQ-to-DQS.

Inter-CA matching ensures all CA bits within a channel are matched to each other (since they share a common delay adjustment). The target is plus or minus 50-100 um.

Inter-byte matching (between byte lanes) is the most relaxed because each byte has independent training. The target is plus or minus 1-2 mm, sufficient to ensure training convergence.

WCK-to-DQS matching (within the same channel) ensures the forwarded write clock and the data strobe have a predictable relationship. The target is plus or minus 500 um, compensated by WCK2CK training.

---

### Q2. How is serpentine (meander) routing used for length tuning, and what are the design rules?

**Answer:**

Serpentine routing adds length to a shorter trace by introducing a series of parallel routing segments connected by 90-degree or 45-degree bends. The serpentine pattern creates a controlled detour that increases the total route length without changing the start and end points.

The design rules for serpentine routing ensure that the meander does not introduce signal integrity issues. The serpentine pitch (distance between adjacent parallel segments) should be at least 3x the trace width. Smaller pitch causes capacitive self-coupling between parallel segments, which alters the effective impedance and propagation velocity, making the delay not proportional to the physical length.

The serpentine amplitude (depth of each segment) determines how much length is added per cycle. Larger amplitude adds more length but occupies more routing area. Typical amplitudes are 10-50 um per segment.

The serpentine should use 45-degree bends rather than 90-degree bends, as sharp corners create impedance discontinuities and current crowding at the inner corner. The 45-degree bend has a more gradual impedance transition.

The serpentine location is important: it should be placed at the end of the route (near the IO cell or bump) rather than in the middle. Placing the serpentine at the end means the majority of the route is a clean, straight trace, with the impedance perturbation confined to a small region.

For differential pairs, the serpentine must be applied symmetrically to both lines. If one line needs length added, the serpentine should extend from both lines (one inward, one outward) to maintain consistent intra-pair spacing. Alternatively, only the shorter line is serpentined while the longer line is held straight, but this creates a region where the pair spacing changes.

The total delay added by a serpentine is approximately equal to the physical length added multiplied by the propagation velocity, minus a correction factor for self-coupling. For well-designed serpentines (pitch greater than 3x width), the correction is less than 5%.

---

### Q3. What is the skew budget methodology, and how is it applied to LPDDR5?

**Answer:**

The skew budget methodology is a systematic approach to allocating the total timing budget across all sources of skew and uncertainty. For LPDDR5, the total timing budget is defined by the UI (156.25 ps at 6400 MT/s) minus the receiver setup and hold times.

The skew budget is partitioned among contributors: DRAM-side skew (tDQS2DQ from DRAM process variation and internal routing), package-side skew (from package substrate and TMV routing), SoC on-die skew (from PHY layout routing), PLL/DLL jitter (clock generation uncertainty), power supply noise (VDDQ-induced timing shift), crosstalk (coupling-induced jitter), temperature variation (delay changes with temperature), and voltage variation (delay changes with supply voltage).

The budget is typically managed in a spreadsheet or tracking document that assigns each contributor a maximum value. The sum of all contributors must be less than the available timing budget (UI - tDS - tDH).

For layout engineers, the key controllable contributors are SoC on-die skew (addressed by length matching and routing quality), crosstalk (addressed by spacing and shielding), and power supply noise (addressed by power grid design). These contributors are allocated specific budgets, and the layout must be designed and verified to meet them.

The skew budget is evaluated at worst-case conditions: maximum temperature, minimum voltage, worst-case process corner. The layout must meet the budget under these conditions, not just at nominal.

A typical skew budget allocation for LPDDR5 at 6400 MT/s might be:

| Contributor | Budget (ps) |
|---|---|
| UI | 156.25 |
| tDS (setup) | -55 |
| tDH (hold) | -55 |
| Available margin | 46.25 |
| DRAM tDQS2DQ | -15 |
| Package skew | -5 |
| SoC on-die skew | -8 |
| PLL jitter | -7 |
| VDDQ noise | -5 |
| Crosstalk | -3 |
| Guard band | -3.25 |

This leaves 3.25 ps of guard band, which is extremely tight and demonstrates the challenge of LPDDR5 timing closure.

---

### Q4. How does temperature affect routing delay and skew?

**Answer:**

Temperature affects routing delay through changes in metal resistivity and dielectric properties. As temperature increases, the metal resistivity increases (approximately +0.3-0.4% per degree Celsius for copper), which increases the RC delay of the interconnect. The dielectric constant may also change slightly with temperature (typically less than 0.1% per degree Celsius).

For a typical on-die metal route, the delay temperature coefficient is approximately +0.2-0.3% per degree Celsius. Over a 100-degree temperature range (from -40C to 85C for commercial, or 0C to 125C for automotive), the delay variation is approximately 20-30%.

For length matching, the temperature effect is largely common-mode: all traces in the same physical region experience the same temperature and therefore the same delay change. The intra-byte skew (which depends on the differential delay between DQ and DQS) is relatively insensitive to temperature because both DQ and DQS scale similarly. The residual differential temperature effect (from local thermal gradients within the byte lane) is typically less than 1-2 ps.

However, temperature does affect the overall timing budget through other mechanisms. The PLL jitter may increase at high temperature (due to increased thermal noise in the VCO). The driver and receiver characteristics change with temperature (threshold voltage shift, mobility degradation), affecting the setup and hold times. The DRAM internal timing (tDQSCK, tDQS2DQ) varies with temperature.

For layout, temperature effects are managed by designing for the worst-case temperature corner and including the temperature-dependent timing variation in the skew budget. The PHY training algorithms recalibrate periodically to track temperature drift, but between calibration events, the temperature-induced timing shift must be within the available margin.

---

### Q5. How are length matching constraints specified in EDA tools?

**Answer:**

Length matching constraints are specified in EDA place-and-route tools (Cadence Innovus, Synopsys ICC2) using group-based matching rules. The specification typically involves defining signal groups, setting the matching tolerance, and assigning a reference signal.

In Cadence Innovus, a typical constraint specification uses the setNanoRouteMode or create_route_rule commands to define matching groups. For example, a DQ byte lane matching constraint would specify all 8 DQ signals plus DQS as a group, with a matching tolerance of plus or minus 50 um relative to the DQS length.

In Synopsys ICC2, the set_routing_rule command with length matching options performs a similar function. The tool can also support differential pair constraints, which automatically enforce equal length between the two lines of a differential pair.

The matching constraints can be specified hierarchically: tight matching within bytes, moderate matching within the CA group, and loose matching between bytes. The tools process these constraints during routing, attempting to meet all tolerances simultaneously.

During routing, the tool performs automatic serpentine insertion to add length to shorter traces. The serpentine parameters (pitch, amplitude, corner style) can be configured to meet the design rules discussed earlier.

After routing, the tool generates a length matching report that lists each signal's total route length, the reference length, and the deviation. Signals that exceed the tolerance are flagged as violations. The layout engineer reviews these reports and manually adjusts the routing or serpentine tuning to resolve violations.

For post-route verification, parasitic extraction (rather than physical length) provides a more accurate measure of delay matching. The extracted RC or RLC network accounts for via delays, coupling effects, and geometry-dependent impedance that physical length alone does not capture. Some advanced flows use extracted delay matching as the final sign-off criterion rather than physical length.

---

### Q6. What is the difference between length matching and delay matching?

**Answer:**

Length matching ensures that all traces in a group have the same physical length (in micrometres or millimetres). Delay matching ensures that all traces have the same propagation delay (in picoseconds). While related, these are not identical because different traces may have different propagation velocities.

The propagation velocity of a trace depends on the effective dielectric constant of the surrounding materials, which varies with the metal layer (different layers have different dielectric thicknesses and compositions), the trace width (wider traces have slightly different effective dielectric than narrow traces), the proximity to other metal (coupling changes the effective capacitance and velocity), and the via transitions (vias add lumped delay that is not proportional to physical length).

For practical LPDDR layout, the difference between length matching and delay matching is small if all traces in the group are routed on the same metal layer, with the same width, in a similar coupling environment, and with the same number of vias. Under these conditions, the propagation velocity is consistent across all traces, and length matching is an excellent proxy for delay matching.

However, if any of these conditions are violated (such as one DQ signal using a different metal layer for part of its route, or having one extra via transition), length matching may not ensure delay matching. In such cases, the layout engineer should use delay matching based on parasitic extraction.

Modern EDA tools support both length-based and delay-based matching. Delay-based matching uses estimated or extracted parasitics to compute the signal delay and matches on delay rather than length. This is more accurate but also more computationally expensive.

For LPDDR5 at 6400 MT/s, the recommended approach is to use length matching during interactive routing (for fast feedback), with delay matching as a post-route verification step (for accuracy). Any discrepancy between length and delay matching results should be investigated and resolved.

---

### Q7. How is inter-byte matching handled, and what is its impact on system performance?

**Answer:**

Inter-byte matching refers to the length/delay matching between different byte lanes. In LPDDR5, each byte lane has its own DQS timing reference, and the PHY training independently calibrates each byte's read and write timing. This means inter-byte skew is compensated by training and does not directly affect the per-byte timing margin.

However, inter-byte matching still matters for several reasons. Training range consumption is the primary concern: each byte lane's training delay lines have a finite range (typically plus or minus 1-2 UI). If the inter-byte skew is large (approaching the training range), the training may not converge, or it may converge with little remaining range for PVT tracking.

System latency is affected because the memory controller must wait for all bytes to be ready before assembling a complete data word. If one byte has a significantly different read latency (due to routing delay differences), the controller must add wait states, increasing the effective read latency.

Power consumption is affected because the training delay lines consume power proportional to their delay setting. If one byte requires a large compensating delay (due to large routing skew), the associated delay line draws more power.

The inter-byte matching target is typically plus or minus 1-2 mm of physical length, which corresponds to approximately plus or minus 150-300 ps of delay. This is achievable with moderate routing effort: the byte lanes should have approximately the same routing path length from the IO cells to the bumps, without requiring precise serpentine tuning.

For layout, inter-byte matching is addressed during floorplanning by placing all byte lanes at approximately the same distance from the memory controller and from the bump field. If one byte lane has a significantly longer routing path (due to floorplan constraints), the layout engineer should be aware that this byte will consume more training range and should verify that the training algorithm can accommodate the skew.

---

### Q8. How is skew measured and verified in silicon after fabrication?

**Answer:**

After fabrication, the actual skew is measured through the PHY's built-in training and eye scanning capabilities. The primary methods include training result analysis, eye diagram scanning, and loopback testing.

Training result analysis examines the calibration codes determined by the read and write training algorithms. The read leveling code for each byte indicates the delay needed to align DQS with the internal clock, which includes the routing delay contribution. By comparing the training codes across bytes, the inter-byte skew can be inferred. Similarly, the per-bit deskew codes within a byte indicate the intra-byte skew.

Eye diagram scanning uses the PHY's variable delay and variable VREF capabilities to sweep the sampling point across the data eye. For each DQ bit, the scan produces a two-dimensional plot (timing vs voltage) showing the passing region. The horizontal opening of the eye indicates the timing margin, and the centre of the eye indicates the optimal sampling point. The difference between the DQS edge and the eye centre reveals the effective DQ-to-DQS skew after training.

Loopback testing routes the PHY's output back to its input (either on-die or through the package) and measures the round-trip delay. By comparing the round-trip delays of different DQ bits, the relative skew can be determined.

For the layout engineer, silicon measurements provide validation of the layout skew predictions. A well-designed layout should show training codes that are close to the centre of the training range (indicating small initial skew) and eye diagrams with margins that match or exceed the simulation predictions. If the silicon results show larger-than-expected skew, the layout should be investigated for unmatched via transitions, unintended coupling effects, or length matching errors that were not caught during verification.

---

### Q9. What are the skew implications of corner routing and signal crossings?

**Answer:**

Corner routing (bends in the trace path) and signal crossings (where one trace crosses over another on a different layer) both introduce skew if not handled carefully.

Corner routing affects skew because a 90-degree bend changes the trace direction, and the inner and outer edges of the bend have different path lengths. For a trace width of 1 um with a 90-degree bend, the inner edge is approximately 0.5 x pi x 0.5 = 0.8 um shorter than the outer edge. This is negligible for a single-ended trace (the average path length is unchanged), but for a differential pair making a bend, the inner line becomes shorter than the outer line by approximately pi x spacing / 2. For a differential pair with 2 um spacing, this difference is approximately 3 um per 90-degree bend, which corresponds to approximately 0.5 ps of skew. Multiple bends in the same direction accumulate this skew.

For layout, differential pair bends should alternate direction (S-curves rather than C-curves) to cancel the skew from individual bends. If this is not possible, compensating length should be added to the shorter line at the bend location.

Signal crossings occur when a DQ trace must cross over another signal group (such as the CA bus or a different byte's DQ). The crossing requires a layer transition (at least 2 vias), which adds delay to the crossing trace. If only one DQ in the byte needs to cross, it acquires extra via delay that the other DQ bits do not have, creating skew.

For layout, crossings should be avoided within a byte lane. If a crossing is unavoidable, all DQ signals in the byte should cross together (even if some do not strictly need to) to maintain matched via counts. Alternatively, the extra via delay can be compensated by shortening the physical length of the crossing trace.

---

### Q10. How do length matching requirements evolve as LPDDR data rates increase toward LPDDR5X and LPDDR6?

**Answer:**

As data rates increase, the UI decreases proportionally, and the absolute timing budget shrinks. This directly tightens the length matching requirements because the same physical length mismatch represents a larger fraction of the smaller UI.

For LPDDR5X at 8533 MT/s: UI = 117.2 ps. With the same setup and hold times (approximately 50 ps each), the available margin is only approximately 17 ps. The SoC on-die skew allocation might shrink to 5-6 ps, requiring length matching within plus or minus 35-40 um.

For LPDDR6 (estimated 12800+ MT/s): UI = 78 ps. The timing margin becomes extremely small (potentially less than 10 ps total), requiring length matching within plus or minus 20-25 um. At this level, even the serpentine tuning granularity becomes a concern (the minimum serpentine segment may not provide fine enough length adjustment).

At these tighter tolerances, length matching alone may not be sufficient because via delay variation, coupling-induced delay variation, and process variation may exceed the matching tolerance. The design methodology will need to evolve.

Delay-based matching (using extracted parasitics rather than physical length) becomes essential. The layout tools must support extraction-driven routing, where the router uses a fast parasitic estimation to guide routing decisions in real time.

Per-bit deskew range must increase to compensate for larger residual skew after layout matching. This requires more delay cells in each delay line, which adds parasitic load and may degrade bandwidth.

Training algorithms must become more sophisticated, potentially using continuous background calibration to track temperature and voltage drift at a faster rate.

Equalization (decision feedback equalization, DFE) may be added to the receiver to compensate for ISI, relaxing the impedance matching requirement and allowing more routing flexibility.

For layout engineers, the trend toward tighter matching demands more automation, better extraction accuracy, more SI simulation, and closer collaboration between the layout, circuit design, and SI teams.

---

See also:
- [DQ DQS Routing](dq_dqs_routing.md)
- [CA CK Routing](ca_ck_routing.md)
- [LPDDR Signaling and Timing](../01_foundations/lpddr_signaling_and_timing.md)
- [Worked Problem: Skew Budget Analysis](worked_problems/problem_03_skew_budget_analysis.md)
