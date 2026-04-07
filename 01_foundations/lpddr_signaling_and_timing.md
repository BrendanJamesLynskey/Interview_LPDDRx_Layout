# LPDDR Signaling and Timing

This section covers the signaling conventions and critical timing parameters in LPDDRx interfaces. These fundamentals directly determine the constraints that layout engineers must meet for length matching, impedance control, and signal integrity.

---

### Q1. What are the signaling types used in LPDDR5, and why are different types used for different signal groups?

**Answer:**

LPDDR5 employs three distinct signaling types, each chosen to optimise the particular requirements of the signal group.

Differential signaling is used for clocks: CK/CK_n (system clock) and WCK/WCK_n (write clock). Differential signaling provides superior noise rejection because any common-mode noise coupled onto both lines is cancelled at the differential receiver. It also provides a precise crossing point (where the two signals cross at VDDQ/2) that serves as the timing reference. Clock jitter must be minimised because it directly adds to timing uncertainty for all signals referenced to that clock. The differential pairs require controlled impedance routing (typically 100 ohm differential) with tight coupling between the positive and negative traces.

Single-ended signaling with a VREF (voltage reference) threshold is used for DQ (data), CA (command/address), and DQS/DQS_n (data strobe). While DQS is routed as a differential pair, each line also functions as a single-ended signal in certain modes. The DQ signals are point-to-point, single-ended, referenced to a trained VREF level (approximately VDDQ/2, adjusted during VREF training). Single-ended signaling is used for data because it maximises pin efficiency -- each pin carries one bit per UI, whereas differential signaling would require two pins per bit.

The CA bus signals are also single-ended, referenced to a CA-VREF. LPDDR5 uses double data rate on the CA bus (both CK edges), so the CA signals toggle at the CK frequency. The timing for CA signals is referenced to CK, and the setup/hold requirements are specified relative to the CK crossing point.

The choice of signaling type for each group balances pin count efficiency, noise immunity, timing precision, and power consumption. Layout engineers must apply the appropriate routing rules for each type: differential pairs for clocks with tight coupling and matched lengths, and single-ended controlled-impedance traces for DQ and CA with appropriate spacing for crosstalk control.

---

### Q2. What is the VDDQ voltage in each LPDDR generation, and how does it affect signal swing and noise margin?

**Answer:**

The VDDQ voltage has decreased with each LPDDR generation: LPDDR4 operates at 1.1V, LPDDR4X at 0.6V, and LPDDR5/5X at 0.5V (with some modes supporting 0.3V for ultra-low-power operation). This progressive voltage reduction is driven by the need to reduce IO power consumption, which scales with V^2.

The signal swing for single-ended signals is approximately VDDQ (rail-to-rail swing from VSS to VDDQ). The receiver uses a trained VREF at approximately VDDQ/2 as the decision threshold. The noise margin is the voltage difference between the signal level and the VREF threshold, minus any noise sources. For a perfect signal at LPDDR5 0.5V VDDQ, the ideal noise margin is 0.25V (VDDQ/2). However, real systems have noise from multiple sources: power supply ripple (IR drop and Ldi/dt noise on VDDQ), crosstalk from adjacent signals, inter-symbol interference (ISI) from reflections and lossy transmission lines, and receiver input offset.

At 0.5V VDDQ, even a 25mV VDDQ ripple consumes 10% of the noise budget. This is far more critical than at 1.1V VDDQ (LPDDR4), where the same 25mV ripple is only 4.5% of the margin. The reduced noise margin at lower VDDQ means that every aspect of the physical design must be tighter: the power grid must have lower impedance, crosstalk must be reduced through spacing or shielding, and impedance discontinuities must be minimised to reduce reflections.

For layout engineers, the practical impact is that LPDDR5/5X designs require more aggressive decoupling, tighter impedance control, wider spacing between signal groups, and more careful attention to return current paths. Simulation-driven design becomes essential rather than optional -- the margins are too thin to rely on rule-of-thumb guidelines alone.

---

### Q3. What are the critical timing parameters for LPDDR5, and which ones directly depend on layout?

**Answer:**

The critical timing parameters for LPDDR5 include several that are directly affected by the physical layout.

tDQSCK (DQS output access time) is the time from CK crossing to DQS transition at the DRAM during a read operation. This parameter is a property of the DRAM device and is typically 1.5-3.5ns. The SoC PHY must accommodate this range during read leveling training.

tDQS2DQ (DQS-to-DQ skew) specifies the time offset between DQS and its associated DQ bits at the receiver. The JEDEC spec typically allows less than plus or minus 200ps for this parameter. In the layout, this translates directly to a length matching requirement between DQS and each DQ within a byte lane. For a signal propagation velocity of approximately 150ps/mm on-die (in intermediate metal layers), 200ps corresponds to approximately 1.3mm of length difference -- a tight but achievable constraint.

tDQSS (write DQS to CK timing) defines the alignment between write DQS and CK at the DRAM. This is calibrated during write leveling training, but the layout must ensure that the initial skew before training is within the trainable range (typically plus or minus 1 UI).

tCK (clock period) determines the data rate. At 6400 MT/s, tCK = 312.5ps (CK runs at 3200 MHz). The clock distribution must maintain jitter well below tCK/2 to ensure clean sampling.

tWCK2DQ (WCK-to-DQ timing) in LPDDR5 defines the relationship between WCK and write data. Since WCK is forwarded from the SoC to the DRAM, the layout must ensure that WCK and the associated DQ/DQS signals arrive at the DRAM package with appropriate timing alignment.

Setup time (tDS) and hold time (tDH) at the receiver define the window during which data must be stable relative to the strobe edge. These are typically 40-100ps each, defining the effective data eye requirement at the receiver. Any layout-induced skew directly reduces the available setup or hold margin.

---

### Q4. How does read leveling work, and what layout considerations support it?

**Answer:**

Read leveling (also called read training or read DQS gate training) is the process of calibrating the timing relationship between the read DQS returning from the DRAM and the SoC's internal clock. When the DRAM sends read data, the DQS strobe accompanies the DQ data with a known phase relationship (edge-aligned or centre-aligned, depending on the generation). However, the exact arrival time of DQS at the SoC's PHY depends on the flight time through the package, any on-die delays, and PVT variations. Read leveling determines the correct phase and delay setting to reliably capture the read data.

The process works as follows: the memory controller issues a read command, and the PHY sweeps a variable delay on the DQS receive path. At each delay setting, the PHY samples the incoming DQS and determines whether it correctly captured the expected preamble or data pattern. By sweeping across the full UI range, the PHY identifies the passing window (the range of delay settings that produce correct data) and sets the operating point at the centre of this window.

For layout, read leveling depends on the DQS receive path having a clean, monotonic delay adjustment. The delay cells (typically digitally controlled delay lines) must be placed close to the DQS receiver to minimise parasitic delay variation. The feedback path from the DQS sampler to the training controller must be matched for all byte lanes so that the training algorithm converges to a consistent solution across bytes.

The layout must also ensure that the DQS routing from the package bump to the PHY receiver has predictable delay that does not vary significantly with metal density or coupling from adjacent signals. Any unexpected delay variation would shift the optimal training point and reduce margin. Shielding the DQS route and maintaining consistent impedance along its length helps ensure a reliable training result.

---

### Q5. How does write leveling work, and what is its relationship to the physical routing?

**Answer:**

Write leveling calibrates the timing of the write DQS strobe relative to the system clock (CK) at the DRAM. Because the flight time from the SoC to the DRAM is not known precisely at design time (it depends on package parasitics, die-to-die variations, and temperature), the SoC must determine the correct DQS launch timing so that DQS arrives at the DRAM aligned with CK.

The write leveling procedure in LPDDR4/5 works as follows: the memory controller puts the DRAM into write leveling mode (via a mode register write). The SoC PHY then sends DQS edges while sweeping the DQS delay. The DRAM samples DQS using its internal CK reference and feeds back the result on DQ[0] (the DRAM drives DQ[0] to indicate whether DQS arrived before or after the CK edge). The SoC reads this feedback and adjusts the DQS delay until the optimal alignment is found.

The physical routing directly affects write leveling because the delay from the SoC's DQS driver to the DRAM's DQS receiver includes the on-die routing delay within the SoC, the bump/via delay in the SoC package, the interconnect delay through the PoP structure, the package routing within the DRAM package, and the on-die routing within the DRAM. The total flight time can be several hundred picoseconds to over a nanosecond.

The layout must ensure that the DQS delay adjustment range in the PHY is sufficient to cover the worst-case flight time variation across all PVT corners. Typically, the delay line must span at least 1-2 UI of adjustment range. The layout engineer should also ensure that the routing delay for DQS and DQ within a byte lane is closely matched, because write leveling only aligns DQS to CK -- the DQ-to-DQS alignment within the byte must be maintained by length matching in the layout.

---

### Q6. What is tDQS2DQ, and how do layout engineers ensure compliance with this specification?

**Answer:**

tDQS2DQ is the timing skew between the DQS strobe and any individual DQ bit within the same byte lane. This parameter is critical because the receiver uses DQS edges to sample DQ data -- any skew between DQS and DQ directly reduces the timing margin (data eye opening) at the receiver.

The JEDEC specification for LPDDR5 typically requires tDQS2DQ to be less than plus or minus 200ps for the total system (including DRAM, package, and SoC contributions). The SoC layout contribution to this budget is usually allocated 50-100ps, with the remainder consumed by the DRAM and package.

To meet this specification, layout engineers employ several techniques. First, intra-byte length matching: all DQ traces and the DQS pair within a byte lane are routed to the same length, typically within plus or minus 50-100 micrometres. This corresponds to approximately plus or minus 7-15ps of delay matching at typical on-chip propagation velocities. Second, consistent routing topology: all signals within the byte should use the same metal layers, the same number of vias, and the same routing environment (similar neighbouring metal density and coupling conditions). Third, via matching: every DQ signal should have the same number of via transitions as DQS, since each via adds approximately 5-15ps of delay.

Serpentine (meander) routing is used to add length to shorter traces. The serpentine pitch must be large enough (typically greater than 3x the trace width) to avoid self-coupling that would alter the effective impedance and delay. The serpentine segments should be placed close to the trace endpoint (near the IO cell or bump) to minimise the region of impedance mismatch.

Layout engineers typically verify tDQS2DQ compliance using parasitic extraction (RC or RLC extraction) followed by timing analysis, or by using length-based rules with appropriate guard bands for via and coupling effects.

---

### Q7. What is the eye diagram, and what determines the eye opening for LPDDR5 signals?

**Answer:**

An eye diagram is a visualisation created by overlaying multiple UI (unit intervals) of a signal waveform to show the aggregate opening through which the receiver must sample data. The horizontal opening represents the timing margin, and the vertical opening represents the voltage margin. A wider, taller eye opening indicates better signal integrity and more margin for reliable operation.

For LPDDR5 DQ signals at 6400 MT/s, the UI is 312.5ps (1 / 6400 MT/s * 2, since DDR means data on both edges). The eye opening is reduced from the ideal by several factors. ISI (inter-symbol interference) is caused by frequency-dependent loss in the channel and reflections from impedance mismatches. At 6400 MT/s, the Nyquist frequency is 3.2 GHz, and the channel (on-die routing, package traces, PoP interconnect) attenuates higher-frequency components, causing the signal to not fully transition between bits, closing the eye. Crosstalk from adjacent signals couples noise onto the victim signal, reducing both voltage and timing margins. Power supply noise on VDDQ shifts the signal levels and the receiver threshold, reducing the voltage margin. Jitter on DQS (from PLL phase noise, power supply-induced jitter, and routing-induced skew) reduces the timing margin by shifting the sampling point within the eye.

The JEDEC specification defines minimum eye opening requirements as setup and hold times (tDS/tDH for the data eye and tDS_CA/tDH_CA for the command eye). These parameters define the minimum time window during which the signal must be at a valid level, after accounting for all degradation sources.

For layout engineers, maximising the eye opening requires minimising ISI (maintaining impedance continuity, reducing via transitions, using low-loss metal layers), minimising crosstalk (adequate spacing, shielding), maintaining power integrity (low VDDQ ripple), and ensuring matched routing (to minimise DQS-to-DQ skew that shifts the sampling point).

---

### Q8. How does the differential clock (CK) signaling work in LPDDR5, and what are the layout requirements?

**Answer:**

LPDDR5 uses a differential system clock (CK/CK_n) that provides the timing reference for command/address sampling at the DRAM and for overall system synchronisation. The CK signal is driven by the SoC PHY as a differential pair with a typical impedance of 50 ohm per line (100 ohm differential). The CK frequency is the data rate divided by the WCK:CK ratio divided by 2 (for DDR). At 6400 MT/s with WCK:CK = 4:1, the CK frequency is 800 MHz.

The differential CK pair requires careful layout attention. Impedance control demands that both lines of the pair maintain the target impedance along their entire length. This is achieved by using the appropriate trace width and spacing for the metal layer and dielectric stack, as determined by a 2D field solver. The coupling between CK and CK_n should be consistent -- they should be routed as a tightly coupled pair with fixed spacing.

Length matching between CK and CK_n is critical for maintaining duty cycle accuracy. Any length mismatch translates to a timing offset between the rising and falling edges, which becomes duty cycle distortion (DCD). The JEDEC spec requires duty cycle to be within 45-55% (tCH/tCL specifications). A 1% duty cycle error at 800 MHz CK corresponds to 6.25ps, so the length matching between CK and CK_n must be within a few micrometres.

The CK pair must also be length-matched to the CA signals within the same channel, as CA is sampled on CK edges. The CK-to-CA skew budget is defined by the setup and hold times (tIS/tIH), typically 150-250ps. This allows somewhat more routing flexibility than intra-byte DQ-to-DQS matching but still requires intentional length management.

The CK routing should avoid running parallel to high-speed data signals (DQ, DQS) over long distances to prevent crosstalk coupling. If parallel routing is unavoidable, interposing a ground shield trace between CK and adjacent signals is recommended.

---

### Q9. What is the purpose of the DQS preamble and postamble, and how do they affect timing?

**Answer:**

The DQS preamble is a defined pattern that precedes the first data edge of a burst, allowing the receiver to detect the start of data and lock onto the DQS timing. In LPDDR4, the preamble is a static low level for 1 tCK (static preamble) or a toggling pattern for 2 tCK (toggling preamble). LPDDR5 uses a programmable preamble (static or toggling, 1 or 2 cycles) configured via mode registers.

The DQS postamble follows the last data edge of a burst and provides a clean termination of the strobe signal. It is typically 0.5 tCK of static level, ensuring the receiver does not generate spurious clock edges after the burst ends.

For timing, the preamble serves as the initial timing acquisition window. The receiver's DQS gate circuit must be enabled during the preamble to detect the first DQS edge and begin sampling data. If the DQS gate opens too early, it may detect noise as a false DQS edge. If it opens too late, it misses the first data beat. The read leveling training determines the optimal DQS gate timing.

The toggling preamble provides better timing acquisition because it gives the receiver multiple edges to lock onto before the actual data arrives. However, it adds latency (one extra clock cycle) to each read burst. The choice between static and toggling preamble is a trade-off between timing robustness and latency.

For layout, the preamble and postamble affect the switching pattern on DQS and thus the power supply current profile. The transition from tri-state (between bursts) to the preamble pattern creates a transient current demand as the driver turns on and begins toggling. The power grid must handle this transient without excessive droop that could corrupt the early data beats. The DQS routing must also be clean enough that the preamble pattern is undistorted, as a corrupted preamble can cause the gate circuit to misalign.

---

### Q10. How are setup and hold times specified for LPDDR5, and how do layout engineers translate these into physical constraints?

**Answer:**

Setup time (tDS) is the minimum time that data (DQ) must be stable before the sampling edge of DQS. Hold time (tDH) is the minimum time that data must remain stable after the DQS edge. For LPDDR5 at 6400 MT/s, typical tDS and tDH values are in the range of 40-80ps each. Similar parameters exist for the CA bus: tIS (input setup for CA relative to CK) and tIH (input hold for CA relative to CK).

The total timing budget for a single data eye is one UI minus the sum of all timing deductions. At 6400 MT/s, 1 UI = 156.25ps (the data rate is double data rate, so the UI is half the clock period). From this 156.25ps, we must subtract: tDS (setup time), tDH (hold time), DQS jitter, VDDQ-induced timing shift, crosstalk-induced jitter, and DQS-to-DQ skew from layout. What remains is the timing margin.

Layout engineers translate setup and hold requirements into physical constraints through several steps. First, the DQ-to-DQS length matching budget is derived by allocating a portion of the timing margin (typically 10-30ps) to layout-induced skew. At 6-7ps per millimetre of propagation delay, this corresponds to 1.5-4mm of length matching tolerance. Second, the impedance continuity requirement comes from the need to minimise reflections that create ISI and close the eye. Via count limits and stub length limits are derived from SI simulation. Third, crosstalk spacing rules are derived by simulating the crosstalk-induced jitter for given aggressor-victim spacing and coupling length, then ensuring this jitter is within the allocated budget.

In practice, layout engineers work with a skew budget spreadsheet that tracks each contributor to timing degradation. The layout-specific allocations (DQ-DQS skew, impedance discontinuity ISI, crosstalk) are derived from this budget and translated into physical design rules: length matching tolerance, via count limits, minimum spacing, and shielding requirements.

---

See also:
- [LPDDR Evolution and Standards](lpddr_evolution_and_standards.md)
- [Memory Architecture Basics](memory_architecture_basics.md)
- [SI for LPDDR](../05_signal_and_power_integrity/si_for_lpddr.md)
- [Length Matching and Skew](../04_routing_and_matching/length_matching_and_skew.md)
- [Worked Problem: Timing Margin Analysis](worked_problems/problem_03_timing_margin_analysis.md)
