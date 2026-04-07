# SI for LPDDR

This section covers signal integrity (SI) analysis for LPDDRx interfaces. SI analysis ensures that the data eye at the receiver meets the JEDEC specifications for voltage and timing margin after accounting for all degradation sources in the channel.

---

### Q1. What is an eye diagram, and how is it constructed for LPDDR5 signals?

**Answer:**

An eye diagram is a composite visualisation created by superimposing multiple unit intervals (UIs) of a signal waveform, triggered on the clock or strobe edge. The resulting pattern resembles an eye, where the vertical opening represents voltage margin and the horizontal opening represents timing margin. A wider, taller eye indicates better signal quality.

For LPDDR5, the eye diagram is constructed by simulating the channel response to a pseudo-random bit sequence (PRBS) that exercises all possible bit transition patterns. The simulation includes the transmitter model (driver impedance, slew rate, pre-emphasis if any), the channel model (on-die routing, package substrate, TMVs, PoP interconnect), the receiver model (input capacitance, ODT, VREF), and all parasitic effects (coupling, reflections, ISI, power supply noise).

The simulator (typically HSPICE, Spectre, or Keysight ADS) drives the PRBS pattern through the channel model and captures the voltage waveform at the receiver input. The waveform is then sliced into 1-UI segments, and all segments are overlaid to form the eye. At LPDDR5 6400 MT/s, each UI is 156.25 ps.

The eye diagram reveals the worst-case voltage margin (minimum eye height), worst-case timing margin (minimum eye width), data-dependent jitter (DDJ, visible as horizontal eye closure at certain bit patterns), ISI (visible as vertical eye closure from previous-bit interference), and crosstalk effects (visible as additional eye closure when aggressor signals are active).

The JEDEC specification defines minimum eye opening requirements through the tDS, tDH, and VDIVmin parameters. The simulated eye must satisfy these requirements across all PVT corners and all data patterns.

For layout engineers, the eye diagram is the ultimate validation of routing quality. Every layout decision (impedance control, via count, length matching, shielding, power grid quality) contributes to the eye opening. SI simulation should be performed early (during floorplanning, with estimated parasitics) and iteratively refined as the layout progresses.

---

### Q2. What causes inter-symbol interference (ISI) in LPDDR5 channels, and how does it affect the eye?

**Answer:**

Inter-symbol interference occurs when the signal from a previous bit has not fully settled before the next bit arrives. The residual energy from previous bits shifts the voltage level of the current bit, closing the eye both vertically and horizontally.

ISI in LPDDR5 channels is caused by several mechanisms. Frequency-dependent loss (skin effect and dielectric loss) attenuates the high-frequency components of the signal more than the low-frequency components. This creates a bandwidth limitation that prevents the signal from fully transitioning in one UI. At 6400 MT/s, the Nyquist frequency is 3.2 GHz, and the channel must pass frequencies up to 6-10 GHz for reasonable eye quality. The on-die routing, package traces, and PoP interconnect all contribute loss.

Reflections from impedance discontinuities create delayed copies of the signal that overlap with subsequent bits. Each via, width change, package transition, and solder bump creates a small reflection. The reflection arrives at the receiver after a delay equal to twice the distance to the discontinuity divided by the propagation velocity. If this delay is an integer multiple of the UI, the reflection adds directly to the data, creating constructive or destructive interference depending on the data pattern.

Dispersion occurs when different frequency components travel at different velocities, spreading the pulse in time. In on-die routing, dispersion is typically small, but in package substrates with lossy dielectrics, it can be noticeable.

For layout, ISI is minimised by maintaining impedance continuity (minimising via count and controlling trace width), using low-loss metal layers for signal routing, keeping the channel as short as possible, and ensuring a continuous reference plane under the signal routing.

The ISI penalty can be quantified from the eye diagram as the difference between the ideal eye height (VDDQ/2 for a double-terminated line) and the actual worst-case eye height. Typical ISI penalties for LPDDR5 are 30-60% of the ideal eye height, leaving 40-70% as the actual voltage margin.

---

### Q3. How are reflections from impedance discontinuities analysed and mitigated?

**Answer:**

Reflections occur at any point where the characteristic impedance of the signal path changes. The reflection coefficient (Gamma) at a discontinuity is given by Gamma = (Z2 - Z1) / (Z2 + Z1), where Z1 is the impedance before the discontinuity and Z2 is the impedance after. A positive Gamma means a portion of the signal is reflected back toward the source, and the transmitted signal is modified.

Common impedance discontinuities in LPDDR5 include via transitions (each via has parasitic capacitance that locally reduces impedance), trace width changes (wider traces have lower impedance), solder bumps (capacitive discontinuity from the bump pad), package substrate transitions (different trace geometry and dielectric), TMVs (significant parasitic inductance and capacitance), and bond pads (large capacitive load at the DRAM die).

To analyse reflections, the layout engineer extracts the parasitic model of the complete signal path using EM simulation (ANSYS HFSS for 3D structures like bumps and TMVs, or 2D cross-section solvers for traces) or lumped-element extraction (Cadence QRC, Synopsys StarRC). The extracted model is then simulated with a time-domain solver (SPICE) or frequency-domain solver (S-parameter analysis).

TDR (Time-Domain Reflectometry) simulation sends a step signal through the extracted channel model and observes the reflected waveform. Each impedance discontinuity appears as a bump or dip in the TDR trace, with the magnitude proportional to the reflection coefficient and the position proportional to the distance.

Mitigation strategies include minimising the number of discontinuities (fewer vias, fewer width changes), optimising the geometry at each discontinuity (adjusting via pad size, adding anti-pad tuning, tapering trace width transitions), and using termination (ODT) to absorb reflections at the receiver end.

---

### Q4. How is channel modelling performed for LPDDR5 SI analysis?

**Answer:**

Channel modelling creates a mathematical or circuit model of the complete signal path from the transmitter output to the receiver input. The model must capture all significant parasitic effects to predict the eye diagram accurately.

The channel is typically divided into segments, each modelled separately and then cascaded. The SoC on-die segment includes the transmitter IO cell model (driver impedance, slew rate, output capacitance) and the on-die routing model (extracted RLC or transmission line model from Cadence QRC or Synopsys StarRC). The SoC package segment includes the bump model, redistribution layer traces, via stacks, and substrate routing. These are modelled using 2.5D EM extraction (Cadence Sigrity, ANSYS SIwave) or 3D EM simulation (ANSYS HFSS) for complex structures. The PoP interconnect segment includes the TMVs and solder joints, typically modelled using 3D EM simulation. The DRAM package segment includes the DRAM substrate routing, bond wires or flip-chip bumps, and the DRAM die pad. The DRAM receiver model includes input capacitance, ODT, and VREF threshold.

The cascade of segments can be represented as S-parameters (frequency domain) or SPICE subcircuits (time domain). S-parameter representation is preferred for frequency-domain analysis (impedance profile, insertion loss, return loss), while SPICE models are needed for time-domain simulation (eye diagrams, BER analysis).

The channel model is validated by comparing simulated results with measurements on test vehicles or silicon. Key metrics for validation include impedance profile (measured with TDR, compared with simulated TDR), insertion loss (S21 in dB versus frequency), return loss (S11 in dB versus frequency), and eye diagram (measured with oscilloscope, compared with simulated eye).

For layout engineers, the on-die segment is the portion they directly control. Providing accurate extracted parasitics to the SI team is essential for channel modelling. The extraction must capture the routing geometry, via parasitics, coupling to adjacent signals, and the reference plane quality.

---

### Q5. What is write leveling from an SI perspective, and how does channel quality affect it?

**Answer:**

Write leveling calibrates the timing of the write DQS strobe relative to the system clock (CK) at the DRAM. From an SI perspective, write leveling is finding the optimal DQS launch timing that centres the DQS edge within the CK sampling window at the DRAM after the signal has propagated through the channel.

The channel quality affects write leveling in several ways. Channel delay determines the nominal DQS-to-CK flight time difference. If the DQS path is longer than the CK path (which is common because DQS passes through the byte lane while CK goes through the CA lane), write leveling must compensate for this delay difference. A larger delay difference requires more of the training delay line's range.

Channel loss attenuates the DQS signal, reducing the edge rate at the DRAM receiver. A slower edge rate makes the sampling point less well-defined, increasing the uncertainty in the leveling result. For LPDDR5 at high data rates, the DQS edge at the DRAM may have a 10-80% rise time of 50-100 ps, which is a significant fraction of the UI.

Reflections in the DQS path create pre-cursors and post-cursors that shift the apparent DQS edge position. A reflection arriving just before the main DQS edge can advance or retard the apparent crossing point by several picoseconds, depending on the data pattern and reflection polarity.

Jitter on DQS (from PLL phase noise and VDDQ-induced jitter) adds uncertainty to the leveling result. The training algorithm must average over multiple measurements to find the true centre of the passing window.

For layout, the DQS path quality directly affects write leveling robustness. A clean DQS path with controlled impedance, minimal reflections, and low jitter produces a wider passing window during training, which means the operating point can be more precisely centred with more remaining margin for PVT variation during operation.

---

### Q6. How does crosstalk from adjacent signals affect the LPDDR5 eye diagram?

**Answer:**

Crosstalk from adjacent signals degrades the eye diagram by adding unwanted voltage and timing perturbations to the victim signal. The crosstalk magnitude depends on the coupling mechanism (capacitive and inductive), the coupling length, the signal-to-victim spacing, the aggressor slew rate, and the data pattern correlation.

For LPDDR5, the most significant crosstalk scenarios include intra-byte DQ-to-DQ crosstalk (where adjacent DQ bits within the same byte couple to each other), DQ-to-DQS crosstalk (where DQ transitions couple onto the DQS strobe, creating jitter), inter-byte DQ-to-DQ crosstalk (where DQ bits from an adjacent byte couple across the byte boundary), and DQ/DQS-to-CK/WCK crosstalk (where data signals couple onto the clock signals, creating jitter).

The crosstalk is classified as near-end (NEXT) and far-end (FEXT). NEXT occurs at the same end as the aggressor driver and is significant for short coupling lengths (as in on-die routing). FEXT occurs at the opposite end and is significant for longer coupling (as in package routing). For on-die routing in LPDDR5, both NEXT and FEXT contribute because the routing lengths are comparable to the signal wavelength.

The eye diagram impact of crosstalk is data-pattern-dependent. Worst case occurs when the aggressor transitions simultaneously with the victim's sampling edge, and the coupling creates the maximum voltage or timing shift at that instant. SI simulation captures this by running a multi-aggressor analysis where all possible data patterns on the aggressor signals are evaluated.

For layout, crosstalk is controlled by increasing spacing between signal groups, inserting ground shields, using different metal layers for different signal groups, and minimising parallel coupling length (crossing at 90 degrees rather than running parallel).

---

### Q7. What simulation tools are commonly used for LPDDR5 SI analysis?

**Answer:**

LPDDR5 SI analysis uses a suite of tools for different aspects of the analysis. For on-die parasitic extraction, Cadence Quantus (QRC) or Synopsys StarRC extract the RLC parasitics of the on-die routing from the layout database. These tools use 2D field solvers applied to the metal cross-sections and produce SPICE-compatible netlists or reduced-order models.

For package and board electromagnetic simulation, ANSYS SIwave provides 2.5D EM analysis of package substrates and PCBs, extracting S-parameter models that capture the broadband frequency response. ANSYS HFSS provides full 3D EM simulation for structures that cannot be adequately modelled in 2.5D (such as TMVs, solder bumps, wire bonds, and complex via transitions). Cadence Sigrity provides similar 2.5D and 3D EM capabilities.

For time-domain circuit simulation, Synopsys HSPICE or Cadence Spectre simulate the complete channel (driver, extracted routing, package model, receiver) in the time domain to generate eye diagrams and measure timing/voltage margins. These simulators handle the nonlinear behaviour of the driver and receiver transistors along with the linear passive channel model.

For statistical and channel analysis, Keysight ADS (Advanced Design System) provides channel simulation with statistical eye analysis, BER contour generation, and equalization optimisation. It can efficiently analyse the impact of jitter, noise, and crosstalk on the eye without running exhaustive time-domain simulations.

For power integrity analysis, Cadence Voltus or Synopsys RedHawk analyse the power distribution network, determining IR drop and dynamic voltage noise. ANSYS RedHawk-SC provides additional capability for transient analysis and PDN impedance extraction.

The typical SI analysis flow involves extracting on-die parasitics (QRC/StarRC), extracting package parasitics (SIwave/HFSS), building the complete channel model by cascading segments, running time-domain simulation (HSPICE/Spectre) to generate eye diagrams, and performing statistical analysis (ADS) for BER estimation.

---

### Q8. How does the LPDDR5 receiver compensate for channel impairments?

**Answer:**

The LPDDR5 receiver incorporates several mechanisms to compensate for channel impairments and maximize the effective eye opening. VREF training adjusts the receiver's voltage threshold (VREF) to the optimal level for each DQ bit. The ideal VREF is at the midpoint of the eye, which may differ from the nominal VDDQ/2 due to asymmetric ISI, offset in the receiver comparator, and supply noise. Per-bit VREF training finds the optimal threshold for each bit individually.

Per-bit deskew adjusts the timing of each DQ bit's sampling point relative to DQS. The deskew compensates for routing skew (DQ-to-DQS length mismatch), via count differences, and coupling-induced delay variations. The resolution is typically 1-5 ps per step, with 32-128 steps available.

Read leveling (DQS gate training) determines when to enable the DQS receiver to capture the incoming strobe from the DRAM. The gate timing accounts for the total flight time from the DRAM to the SoC and ensures the receiver is active during the DQS preamble.

Duty cycle correction (DCC) on the DQS receive path compensates for DCD introduced by the DRAM driver, the channel, or the SoC receiver front end. DCC equalises the high and low phases of DQS to maximise the sampling window.

Some advanced LPDDR5 PHYs include DFE (Decision Feedback Equalization), which uses the previously decided bit values to cancel ISI from those bits. DFE is effective for channels with significant post-cursor ISI (reflections that arrive after the main signal). DFE can recover 20-50% of the ISI-closed eye, significantly improving the voltage margin.

For layout, these compensation mechanisms relax the requirements somewhat (the layout does not need to be perfect because training compensates for imperfections), but each mechanism has limited range. The layout must keep impairments within the trainable range to ensure convergence.

---

### Q9. What is the impact of impedance discontinuities on LPDDR5 eye quality, and how are they quantified?

**Answer:**

Impedance discontinuities create reflections that degrade the eye quality. The impact is quantified using return loss (S11) in the frequency domain and TDR impedance profile in the time domain.

Return loss (S11) measures the ratio of reflected power to incident power at each frequency. For a well-designed LPDDR5 channel, the return loss should be better than -15 dB (less than 18% reflected voltage) at the Nyquist frequency (3.2 GHz for 6400 MT/s) and better than -10 dB at 2x Nyquist (6.4 GHz). Poor return loss at frequencies within the signal bandwidth creates ISI and reduces the eye opening.

TDR impedance profile shows the impedance along the signal path as a function of position (proportional to time delay). An ideal channel has a flat impedance of 50 ohm throughout. Each deviation from 50 ohm indicates a discontinuity. The magnitude of the deviation and its distance from the receiver determine its impact on ISI.

Common discontinuities and their typical impact on LPDDR5 eye quality include via transitions with impedance deviation of plus or minus 10-20%, reducing eye height by 5-10% per via. Solder bumps with impedance deviation of plus or minus 15-25% reduce eye height by 10-15%. TMVs with impedance deviation of plus or minus 20-30% reduce eye height by 15-20%. Trace width changes with impedance deviation of plus or minus 5-15% reduce eye height by 3-8%. Bond wire (if present) with impedance deviation of plus or minus 30-50% reduces eye height by 20-30%.

The cumulative effect of multiple discontinuities is not simply additive because reflections interact with each other (multi-bounce effects). SI simulation with the complete extracted channel model captures these interactions.

For layout, the goal is to minimise the total number and magnitude of discontinuities. This is achieved by careful via design (optimised pad size, anti-pad tuning), consistent trace width, and proper termination. The SI simulation should identify the dominant discontinuities so that layout optimisation effort is focused on the highest-impact elements.

---

### Q10. How is BER (Bit Error Rate) estimated from SI simulation, and what BER targets apply to LPDDR5?

**Answer:**

BER (Bit Error Rate) is the probability that a received bit is incorrectly decoded. For LPDDR5, the target BER is typically 10^-16 or better (less than one error in 10^16 bits), which corresponds to less than one error per approximately 24 hours at 6400 MT/s per pin.

Estimating BER from SI simulation requires statistical techniques because running a time-domain simulation long enough to observe one error in 10^16 bits is computationally infeasible. Instead, two approaches are used.

Eye margin analysis measures the eye opening at the target BER by extrapolating from the simulated eye. The horizontal and vertical eye openings are measured at progressively tighter confidence levels (corresponding to lower BER), and the eye opening at BER = 10^-16 is estimated by fitting a statistical distribution (typically Gaussian) to the measured eye edges.

Statistical channel simulation tools like Keysight ADS compute the BER contour directly from the channel impulse response using convolution-based analysis. This approach evaluates all possible data patterns (or a statistically significant subset) and computes the probability distribution of the received voltage at the sampling instant. The BER at any sampling point is the probability that the received voltage is on the wrong side of VREF.

The BER contour plot shows lines of constant BER on the timing-versus-voltage plane. The area within the 10^-16 BER contour is the effective eye opening at the target BER. This area must be large enough to accommodate the receiver setup/hold times and VREF uncertainty.

For layout engineers, the BER analysis provides the most accurate measure of whether the layout meets the signal integrity requirements. A layout that produces a BER eye opening meeting the tDS/tDH requirements at 10^-16 BER is sign-off quality.

---

See also:
- [Power Integrity for LPDDR](power_integrity_for_lpddr.md)
- [EMI and Noise Mitigation](emi_and_noise_mitigation.md)
- [LPDDR Signaling and Timing](../01_foundations/lpddr_signaling_and_timing.md)
- [Worked Problem: Eye Diagram Analysis](worked_problems/problem_01_eye_diagram_analysis.md)
