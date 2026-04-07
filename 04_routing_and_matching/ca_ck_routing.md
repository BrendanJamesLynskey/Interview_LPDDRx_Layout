# CA CK Routing

This section covers the routing methodology for Command/Address (CA) signals and Clock (CK/WCK) signals in LPDDRx PHY layouts. While CA/CK signals operate at lower data rates than DQ/DQS, they have their own stringent matching and integrity requirements.

---

### Q1. How does the CA bus routing differ from DQ routing in LPDDR5?

**Answer:**

The CA bus in LPDDR5 carries command and address information at double data rate (DDR), toggling on both edges of the CK clock. At 6400 MT/s data rate with WCK:CK ratio of 4:1, the CK frequency is 800 MHz, so the CA bus transitions at 1600 MT/s (effectively). This is 4x slower than the DQ data rate, which relaxes some SI requirements but introduces different challenges.

The CA bus has 7 signals per channel (CA[6:0]) plus CK/CK_n (differential clock), WCK/WCK_n (differential write clock), and CS_n (chip select). All CA signals within a channel are referenced to CK and must be length-matched to CK within the setup/hold budget (tIS/tIH, typically 150-250 ps).

Key differences from DQ routing include unidirectional signaling (CA is driven from SoC to DRAM, no bidirectional operation), a wider matching tolerance (since the CA UI is 4x longer than the DQ UI, the absolute matching tolerance in picoseconds is 4x larger), and no per-bit deskew training on the SoC side (the CA delay is adjusted as a group, not individually).

However, CA routing has its own challenges. All 7 CA bits must arrive at the DRAM within the setup/hold window of CK, so the matching between CA bits must be tight enough that a single delay setting works for all. Unlike DQ, where per-bit deskew can compensate for individual mismatches, CA relies on layout matching. The CK differential pair must maintain excellent duty cycle (45-55%) and low jitter, requiring symmetric routing of the CK/CK_n pair.

The CA routing is typically grouped in a dedicated CA lane within the PHY, separate from the DQ byte lanes. The CA lane contains the 7 CA IO cells, the CK IO cell, the WCK IO cell, and the CS IO cell.

---

### Q2. What are the length matching requirements for CA-to-CK, and how do they compare to DQ-to-DQS?

**Answer:**

The CA-to-CK length matching ensures that all CA signals arrive at the DRAM within the sampling window defined by the CK edge. The JEDEC specification defines setup time (tIS) and hold time (tIH) for the CA bus, typically 150-250 ps each at the DRAM receiver.

The total CA-to-CK skew budget includes the DRAM package contribution, the PoP interconnect contribution, and the SoC contribution (on-die + SoC package). The SoC on-die allocation is typically 30-100 ps, depending on how the budget is partitioned.

At 150 ps/mm propagation velocity, a 100 ps skew allocation corresponds to approximately plus or minus 650 um of length matching tolerance. This is significantly more relaxed than the DQ-to-DQS matching (typically plus or minus 50-100 um).

In practice, achieving plus or minus 200-300 um matching between CA signals and CK is straightforward with standard routing techniques. The wider tolerance allows CA routing to use different metal layers from CK if necessary (with appropriate delay compensation), to cross over other signal groups with less concern about matching impact, and to use simpler serpentine tuning with larger pitch.

However, the matching between individual CA bits should be tighter than the CA-to-CK matching. Since the Command Bus Training (CBT) adjusts all CA bits together (single delay for the CA group), any inter-CA skew directly consumes margin. The inter-CA matching should be within plus or minus 50-100 um, similar to intra-byte DQ matching.

The CK-to-WCK matching is another important constraint. WCK must be synchronised to CK at the DRAM through WCK2CK training, but the initial alignment (before training) must be within the trainable range. The CK and WCK routes should be reasonably matched (within plus or minus 500 um) to ensure training convergence.

---

### Q3. How is the differential CK pair routed to maintain duty cycle accuracy?

**Answer:**

The CK differential pair (CK and CK_n) requires symmetric routing to maintain duty cycle accuracy at the DRAM. Any asymmetry between the two lines (length mismatch, impedance mismatch, coupling asymmetry) creates duty cycle distortion (DCD) that reduces the timing margin for both the rising and falling CK edges.

Symmetric routing of the differential pair requires equal length for CK and CK_n, achieved by routing them as a tightly coupled pair with the same trace width, spacing, and routing path. The length matching between CK and CK_n should be within plus or minus 5-10 um (corresponding to less than 1 ps of skew, which translates to less than 0.1% duty cycle error at 800 MHz CK).

Impedance symmetry requires both lines to have the same characteristic impedance, which is ensured by identical trace width and identical distance to the reference plane. Any local perturbation (via, width change, proximity to other metal) must affect both lines equally.

Coupling symmetry means the capacitive and inductive coupling from external signals to CK and CK_n should be equal. This is naturally achieved when the pair is tightly coupled (small intra-pair spacing) and symmetrically positioned relative to external aggressors. If the pair must run near another signal, the pair should be oriented so that both lines are equidistant from the aggressor.

Common routing practices for CK include maintaining constant intra-pair spacing (typically 1-2x the trace width) along the entire route, routing both lines on the same metal layer without layer transitions (or with matched layer transitions), avoiding bends that change the relative position of the two lines (using 45-degree bends or matched-length bends), and placing ground shields on both sides of the pair to block external coupling.

The CK driver in the IO cell must also be designed for duty cycle accuracy, with matched PMOS and NMOS drive strengths and symmetric layout of the driver transistors. Any duty cycle error introduced by the driver or the routing accumulates and cannot be corrected at the DRAM (unlike DQ, where training can compensate).

---

### Q4. How is the WCK differential pair routed, and what special considerations apply?

**Answer:**

WCK (Write Clock) is a forwarded clock in LPDDR5 that runs at the data rate or half the data rate (WCK:CK ratio of 4:1 or 2:1). At 6400 MT/s with 4:1 ratio, WCK toggles at 3200 MHz, making it the fastest clock in the LPDDR5 interface. At 8533 MT/s (LPDDR5X), WCK runs at 4267 MHz.

The high frequency of WCK makes its routing more demanding than CK. The key considerations include impedance control being more critical because at 3200 MHz, the wavelength on-die is approximately 30 mm (for propagation velocity of 10^8 m/s), and even a 1 mm route is 3% of the wavelength, making transmission line effects significant. The impedance must be controlled to within plus or minus 5% of the 50-ohm target.

Crosstalk sensitivity is elevated because WCK is a clock that sets the timing for write data. Any crosstalk-induced jitter on WCK directly translates to timing uncertainty for the write operation. WCK should be shielded from DQ signals (which transition at the same frequency and can couple significant energy).

Duty cycle preservation is critical because WCK duty cycle distortion translates to asymmetric write timing for data captured on rising versus falling WCK edges. The routing must be symmetric for WCK and WCK_n with matching within plus or minus 5 um.

Length matching to the associated data byte is important because WCK is a forwarded clock that the DRAM uses to sample write data. The WCK arrival time at the DRAM relative to the DQ/DQS arrival time must be within the trainable range. The WCK route should be length-matched to the DQ/DQS routes within the same channel, with a tolerance of approximately plus or minus 500 um (the WCK2CK training compensates for the residual offset).

For layout, WCK is typically routed on the same metal layer as DQS (to match propagation velocity) and in close proximity to the data byte lane it serves. The routing should maintain consistent coupling environment and avoid running parallel to high-speed DQ signals over long distances without shielding.

---

### Q5. What is the clock tree structure for distributing CK within the PHY, and how is it balanced?

**Answer:**

The CK signal has two distribution paths in the PHY: the internal path (from PLL to the CA driver, for generating the CK output to the DRAM) and the return path (CK used as a timing reference for internal operations like write leveling).

The internal CK distribution starts at the PLL output and feeds the CK output driver through a clock tree. This tree must be low-jitter (the PLL's output jitter must not be degraded) and low-skew (if CK serves multiple channels, each channel should receive CK at the same time).

The clock tree is balanced using an H-tree or symmetric buffer tree. In an H-tree, the PLL output drives a central buffer, which splits into two branches (left and right channel pairs). Each branch splits again into individual channel taps. The physical routing of each branch is matched in length and loading.

Buffer stages in the tree provide drive strength for the capacitive load of the downstream routing and buffers. Each buffer is designed to maintain duty cycle (matched PMOS/NMOS strengths) and add minimal jitter (clean power supply, low noise sensitivity). The buffer placement must be symmetric: the left and right branches should have buffers at the same distance from the root.

The tree balance is verified by extracting the complete clock tree (including buffer delays and wire delays) and checking the arrival time at each channel tap. The maximum skew across all taps should be less than 10-20 ps to avoid consuming training range.

For layout, the clock tree routing should be on a dedicated metal layer (or shared with only the lowest-noise signals) with shielding. The routing should avoid areas with high switching activity (such as the DQ byte lanes) to prevent coupling-induced jitter. The power supply to the clock buffers should be filtered (separate from the noisy IO VDDQ) to prevent supply-induced jitter.

---

### Q6. How does the CS (Chip Select) signal routing affect timing?

**Answer:**

The CS_n (Chip Select) signal is a single-ended control signal that enables the DRAM to accept commands. CS_n is sampled by the DRAM on the rising edge of CK: when CS_n is low during a CK rising edge, the DRAM interprets the values on the CA bus as a valid command.

CS_n timing is specified relative to CK, with setup (tIS_CS) and hold (tIH_CS) times that define the valid window. These are typically similar to or slightly more relaxed than the CA setup/hold times.

For routing, CS_n should be length-matched to CK (within the CA group matching tolerance) and routed with the CA group on the same metal layer. Since CS_n determines whether a command is accepted, a timing violation on CS_n can cause a missed command (if CS_n is not asserted when it should be) or a spurious command (if CS_n glitches during a CK edge).

In dual-rank configurations, there are two CS signals (CS0_n and CS1_n), one per rank. Both must be matched to CK within the specified tolerance. The routing of the two CS signals should be independent (not coupled to each other) because they are asserted at different times for different ranks.

For layout, CS_n is a relatively straightforward signal to route because it is single-ended, unidirectional, and operates at CK frequency. The main concern is ensuring it is included in the CA group length matching and that it does not couple excessively to other CA signals or CK.

---

### Q7. How are the CA signals organised within the channel for routing efficiency?

**Answer:**

The 7 CA signals per LPDDR5 channel carry command opcodes and address information. Their organisation within the channel affects routing efficiency and matching feasibility.

JEDEC defines the CA pin assignment to minimise routing complexity: CA[0] through CA[6] are assigned to specific positions in the ball map that allow straightforward routing from the SoC IO cells to the DRAM CA inputs. The assignment considers the physical proximity of each CA pin to the CK pin, ensuring that the inherent length differences between CA pins are minimised.

For routing, the CA signals are best organised as a bus running from the SoC CA IO cells to the channel's bump group. The bus should maintain consistent spacing between all CA traces, the same metal layer for all CA traces (to ensure matched propagation velocity), parallel routing with CK/CK_n running at the centre or edge of the CA group, and WCK/WCK_n routed nearby but with adequate spacing to prevent coupling.

The CA bus width (7 signals at typical routing pitch of 1-2 um per trace plus spacing) is approximately 15-30 um. This is compact enough to route as a group through most congestion points.

For routing efficiency, the CA group should take the most direct path from the IO cells to the bumps. Since the CA signals need less aggressive matching than DQ, there is more flexibility in the routing topology. The CA group can cross over other signal groups (using a different metal layer) if necessary, as long as the layer transition is applied to all CA signals equally.

The CK pair should be routed as close to the CA group as practical, as this minimises the absolute length of both CK and CA routes, making matching easier. If CK is routed far from the CA group, both need longer serpentine tuning to match, which wastes area and introduces unnecessary parasitic loading.

---

### Q8. What are the common CA/CK routing pitfalls, and how are they avoided?

**Answer:**

Several common pitfalls in CA/CK routing can lead to timing violations or signal integrity issues.

CK duty cycle degradation from asymmetric routing is a frequent issue. If the CK pair routes through a congested area where one line must detour around an obstacle while the other does not, the resulting length and coupling asymmetry creates DCD. This is avoided by routing CK on a relatively open layer with no obstacles, and by treating the pair as a rigid unit that moves together.

CA-to-CK matching violation from unplanned routing is another pitfall. When the CA group is routed first and CK is added later (or vice versa), the lengths may not match within the required tolerance. This is avoided by routing CK and CA together, using the CK pair as the reference length and matching all CA signals to it.

Crosstalk from DQ to CK/CA occurs when CK or CA routes run parallel to DQ byte lane signals without adequate spacing or shielding. At LPDDR5 data rates, the DQ signals have significant high-frequency energy that can couple onto the lower-frequency CK and CA signals, creating jitter and noise. This is avoided by maintaining separation (at least 3-5 trace widths) between CK/CA and DQ groups, or by interposing ground shields.

WCK-to-CK coupling can be problematic because WCK toggles at 4x the CK frequency. If WCK is routed too close to CK, the fast WCK transitions can couple onto CK, creating jitter that degrades CK quality. This is avoided by routing WCK with adequate spacing from CK (or with shielding), even though both signals are within the same channel.

Excessive via transitions on CK degrade the clock quality by introducing impedance discontinuities and potential duty cycle distortion (if the via adds slightly different parasitic to CK versus CK_n). This is avoided by routing CK on a single metal layer without transitions, or by ensuring any transitions are symmetric for both lines.

---

### Q9. How does CA-ODT affect the CA routing requirements?

**Answer:**

CA-ODT (Command/Address On-Die Termination) is activated at the DRAM to terminate the CA bus for improved signal integrity. The ODT provides a resistive termination at the DRAM CA input, absorbing the incoming signal and reducing reflections.

With CA-ODT active, the CA signal propagation changes from an unterminated scenario (where the DRAM input is high-impedance and fully reflects the signal) to a terminated scenario (where the ODT absorbs most of the signal energy). This affects the voltage swing at the DRAM: the voltage division between the SoC driver impedance and the DRAM CA-ODT reduces the received swing (similar to DQ with ODT).

For CA routing, the presence of CA-ODT means that the CA traces should be designed for controlled impedance matching. The target impedance for the CA traces (typically 50 ohm single-ended) should be close to the CA driver impedance and the CA-ODT value to minimise reflections. Impedance discontinuities in the CA path are more impactful with ODT because the terminated line produces shorter reflections that can overlap with the next CA bit (at DDR rate, the CA UI is 1/CK_freq).

The CA routing length becomes less critical in terms of reflection timing because the terminated line damps reflections more quickly. However, the length matching between CA bits remains important for the group setup/hold timing.

For the SoC-side layout, CA-ODT does not require changes to the CA IO cells (since the ODT is on the DRAM side). However, the CA driver impedance should be well-controlled (via ZQ calibration) to match the target impedance, and the power grid must handle the additional DC current flowing through the CA drivers into the DRAM ODT.

---

### Q10. How are CA/CK signals handled during power-down and self-refresh modes?

**Answer:**

During power-down and self-refresh modes, the LPDDR interface enters a low-power state where some signals are tri-stated (driven to high impedance) or held at static levels. The CA/CK behaviour in these modes has implications for the routing and termination design.

In self-refresh mode, the DRAM refreshes its contents autonomously without controller intervention. CK can be stopped (held static) or continue toggling at a reduced frequency. CA is not driven (no commands issued). The CA ODT may be deactivated to save power. The DQ/DQS signals are tri-stated.

When CK stops, the CK pair should be parked at a defined state (both high or both low, depending on the implementation) to avoid floating inputs at the DRAM. The routing does not need to change, but the static power dissipation (if CK is held at a state that turns on the CA-ODT) should be considered in the power analysis.

During power-down entry and exit, the CK signal resumes toggling, and the CA bus begins issuing commands. The transition from static to toggling state creates a transient that can couple to nearby signals. The layout should ensure adequate isolation between CK and sensitive analog circuits (such as the PLL) during these transitions.

For layout, the power-down modes do not change the routing requirements but affect the power grid design. During self-refresh, the VDDQ supply may be reduced (to save power) or maintained (to preserve IO cell state). The power grid must support both the active and self-refresh current profiles.

The training state must be preserved across power-down modes. If the PHY state is lost during deep power-down (where VDDQ is removed), full retraining is required on wake-up. The layout must support the worst-case training time (which can be 1-10 ms) by ensuring the initialisation sequence can complete without timing violations.

---

See also:
- [DQ DQS Routing](dq_dqs_routing.md)
- [Length Matching and Skew](length_matching_and_skew.md)
- [LPDDR Signaling and Timing](../01_foundations/lpddr_signaling_and_timing.md)
- [Worked Problem: Clock Distribution](worked_problems/problem_02_clock_distribution.md)
