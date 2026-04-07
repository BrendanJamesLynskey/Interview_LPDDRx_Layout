# IO Cell Design

This section covers the design of IO cells used in LPDDRx PHY interfaces. IO cells are the analog/custom blocks at the die edge that drive and receive signals to and from the DRAM. Understanding IO cell architecture is essential for layout engineers because these cells define the die-edge interface and impose strict placement and routing constraints.

---

### Q1. What is the basic architecture of an LPDDR5 DQ IO cell?

**Answer:**

An LPDDR5 DQ IO cell is a bidirectional structure that functions as both a transmitter (during writes) and a receiver (during reads). The cell contains several sub-circuits arranged in a specific topology.

The output driver is a push-pull (CMOS) structure consisting of a PMOS pull-up network connected to VDDQ and an NMOS pull-down network connected to VSS. Each network consists of multiple parallel transistor legs that can be individually enabled or disabled to set the output impedance. For a 40-ohm target impedance, the total enabled transistor width is sized so that the on-resistance equals 40 ohm. The number of legs and the ZQ calibration code determine which legs are active.

The pre-driver is a buffer stage between the serialiser output and the main driver transistors. It provides the drive strength needed to switch the large output transistors at the data rate. The pre-driver also incorporates slew rate control, which adjusts the rise and fall times of the output to control signal bandwidth and reduce EMI.

The input receiver is a differential comparator that compares the incoming DQ signal against the VREF threshold. The receiver must have sufficient bandwidth to resolve the data eye at 6400 MT/s (requiring a bandwidth of approximately 5-6 GHz). The receiver input is connected to the pad through ESD protection devices.

The ODT (on-die termination) circuit is a resistive termination that can be activated during read operations (when the IO cell is in receive mode) to terminate the transmission line and reduce reflections. The ODT uses the same parallel transistor leg architecture as the driver, but configured as a resistor to VDDQ or VSS.

Level shifters translate signals between the VDDQ domain (0.5V) and the core logic domain (0.5-0.8V). These are needed on both the data path (serialiser to pre-driver, and receiver to deserialiser) and the control path (enable signals, calibration codes).

ESD protection devices (diode clamps or dedicated ESD structures) protect the IO cell from electrostatic discharge events.

---

### Q2. How does the push-pull driver achieve programmable impedance, and what are the layout considerations?

**Answer:**

The push-pull driver achieves programmable impedance through a segmented architecture. The pull-up and pull-down networks each consist of N parallel transistor legs (typically 5-7 binary-weighted legs), where each leg has a fixed on-resistance. By enabling different combinations of legs, the total parallel resistance can be adjusted in fine steps.

For example, a driver with 6 binary-weighted legs (1x, 2x, 4x, 8x, 16x, 32x) provides 63 possible impedance settings. The ZQ calibration process determines the optimal code by comparing against the external precision resistor, and this code is loaded into the driver's enable register.

The layout of the segmented driver is critical for matching. Each transistor leg must be designed and laid out as a unit cell, with all unit cells having identical geometry, orientation, and parasitic environment. This ensures that enabling different combinations of legs produces predictable impedance values. Common layout techniques include using a common-centroid arrangement for the unit cells, interleaving PMOS pull-up and NMOS pull-down cells, using dummy cells at the edges to maintain a uniform neighbourhood, and connecting all cells to the output pad through a balanced metal distribution (H-tree or fishbone pattern) to ensure equal resistance from each cell to the pad.

The total driver width for a 40-ohm LPDDR5 driver at 0.5V VDDQ can be substantial: for an NMOS pull-down with a typical on-resistance of 200 ohm-um (varies by process), achieving 40 ohm requires approximately 5 um of total enabled width. With the segmented architecture and calibration overhead, the total transistor area may be 3-5x larger.

The driver layout must also manage the large transient currents during switching. At 0.5V swing into a 40-ohm load, the peak current is 12.5 mA per pin. For 8 DQ pins switching simultaneously, the total current is 100 mA, which creates significant di/dt noise on the VDDQ supply. The power grid feeding the drivers must have very low impedance to handle this current without excessive voltage drop.

---

### Q3. How does the VREF generator work, and what are the requirements for its distribution in the layout?

**Answer:**

The VREF (voltage reference) generator provides the threshold voltage used by DQ and CA receivers to distinguish between logic high and logic low levels. For LPDDR5, the nominal VREF is approximately VDDQ/2 (0.25V for 0.5V VDDQ), but it is adjustable through VREF training to optimise the receiver margin for each bit.

The VREF generator is typically implemented as a resistor divider from VDDQ to VSS, with digital control to adjust the divider ratio in fine steps. A typical VREF DAC (digital-to-analog converter) provides 6-8 bits of resolution, giving 64-256 steps across the VREF range (typically 10-40% of VDDQ). The step size is approximately 1-3 mV, which corresponds to roughly 1% of the total signal swing.

LPDDR5 supports per-bit VREF training (MR14/MR15 for DQ VREF), meaning each byte can have a different VREF setting. Some implementations go further with per-bit VREF generators, allowing each DQ receiver to have an individually optimised threshold.

For layout, the VREF distribution is critical. VREF is a DC reference that must be extremely stable -- any noise on VREF directly reduces the voltage margin at the receiver. The VREF generator should be placed in a quiet area of the PHY, away from switching signals and noisy power domains. The VREF output should be filtered with a decoupling capacitor (MIM or MOM cap) to suppress high-frequency noise.

The VREF distribution from the generator to each receiver must be routed on a dedicated metal layer, shielded by ground lines on both sides and on the layers above and below. The routing should avoid crossing over or running parallel to high-speed DQ or DQS signals. The resistance of the VREF distribution network should be low enough that the receiver input bias current does not create a significant voltage drop.

If per-byte VREF is used (one generator per byte lane), the generator should be placed centrally within the byte lane, with equal-length distribution to each DQ receiver.

---

### Q4. What is slew rate control, and why is it important for LPDDR IO cells?

**Answer:**

Slew rate control adjusts the rise and fall times of the output driver, controlling how quickly the output transitions between logic levels. The slew rate is specified in V/ns and determines the signal bandwidth -- faster slew rates contain higher-frequency spectral components.

In LPDDR5, slew rate control is important for several reasons. EMI reduction is achieved because faster slew rates generate more high-frequency energy that can radiate as electromagnetic interference. Mobile devices have strict EMI requirements, and controlling the slew rate is one of the primary tools for meeting these specifications. Overshoot and undershoot control is important because excessively fast transitions cause ringing at impedance discontinuities (vias, stubs, package transitions). Slew rate control reduces the amplitude of this ringing by limiting the high-frequency content. Crosstalk reduction results from slower transitions reducing the di/dt coupling between adjacent traces. Finally, there is a signal integrity trade-off: while faster transitions create a wider data eye (more time at the valid logic level), they also increase noise and reflections that can close the eye. The optimal slew rate balances these competing effects.

Slew rate control is typically implemented by adjusting the gate drive of the output transistors. A weaker gate drive turns the transistors on more slowly, producing a slower output transition. Common implementations use multiple pre-driver stages with selectable drive strength, or an RC filter on the gate drive signal.

For layout, slew rate control adds complexity to the pre-driver area. The slew rate control elements (selectable pre-drivers or RC filters) must be placed close to the main driver transistors to minimise parasitic coupling that could bypass the slew control. The control signals for slew rate selection must be routed from the training controller to each IO cell, adding to the routing density in the PHY.

---

### Q5. How are level shifters implemented between the VDDQ and core voltage domains in LPDDR5?

**Answer:**

Level shifters are essential in LPDDR5 IO cells because the VDDQ domain (0.5V) and the core logic domain (0.5-0.8V, depending on the process node) operate at different voltages. Every signal crossing between these domains must pass through a level shifter to ensure correct logic levels and avoid excessive leakage or short-circuit current.

The most common level shifter topology for VDDQ-to-core conversion is the cross-coupled PMOS level shifter. It consists of two cross-coupled PMOS transistors connected to the higher supply (core VDD), with NMOS pull-down transistors driven by the input signal (in the VDDQ domain). When the input switches, one NMOS pulls down one node while the cross-coupled PMOS regenerates the complementary node to the full core VDD level. For core-to-VDDQ conversion, a similar topology is used with the supplies reversed.

In LPDDR5, where VDDQ (0.5V) may be very close to or even equal to the core voltage, the level shifter design becomes challenging. The voltage difference between domains may be only 100-300 mV, requiring very sensitive level shifter circuits that can reliably convert signals with small voltage differences. Some designs use a multi-stage approach with an intermediate voltage level.

For layout, level shifters must be placed at the boundary between voltage domains. They must tap into both the VDDQ and core VDD power grids, which means they sit at the physical intersection of the two power domains. The placement should minimise the wire length on both the input and output sides.

Level shifters are on the critical timing path for both write data (core to VDDQ, from serialiser to driver) and read data (VDDQ to core, from receiver to deserialiser). Any delay added by the level shifter reduces timing margin. Therefore, the level shifters must be compact and placed to minimise parasitic loading. In some implementations, the level shifter is integrated into the pre-driver stage to eliminate an additional buffering stage.

---

### Q6. What ESD protection is required for LPDDR IO cells, and how does it affect the layout?

**Answer:**

Electrostatic discharge (ESD) protection is mandatory for all IO cells to prevent damage from static electricity during handling, assembly, and operation. The JEDEC specification requires LPDDR devices to withstand at least 250V CDM (Charged Device Model) and 1000V HBM (Human Body Model) ESD events.

The primary ESD protection structure is a pair of diode clamps from the IO pad to VDDQ (forward diode) and from VSS to the IO pad (reverse diode). During a positive ESD event, the pad voltage rises above VDDQ, forward-biasing the upper diode and clamping the voltage. During a negative event, the lower diode clamps the pad to VSS. A secondary clamp (often a large NMOS device or an SCR - Silicon Controlled Rectifier) provides additional protection between VDDQ and VSS to handle the current shunted by the primary clamps.

For LPDDR5 at 0.5V VDDQ, the ESD design is particularly challenging. The ESD diodes add parasitic capacitance to the IO pad, typically 100-300 fF depending on the protection level. At 6400 MT/s, this capacitance degrades the signal bandwidth and increases the effective load on the driver. The trade-off is between ESD robustness (more diode area = more capacitance = better protection) and signal performance (less capacitance = faster transitions = wider eye).

For layout, the ESD structures are placed immediately adjacent to the IO pad, between the pad and the core circuit. They must be large enough to handle the ESD current without damage (the diodes must carry several amps for nanoseconds) but compact enough to minimise parasitic capacitance. The ESD diodes must connect to robust VDDQ and VSS rails that can carry the ESD current without melting. These rails are often dedicated metal straps separate from the normal power distribution.

The ESD protection ring around the die perimeter must be continuous and low-impedance. Any break in the ESD ring can create a vulnerable point. The layout engineer must ensure that the ESD structures are properly connected and that the guard rings around the IO cells are complete.

---

### Q7. How does the receiver in an LPDDR5 DQ IO cell achieve the required bandwidth?

**Answer:**

The DQ receiver in LPDDR5 must resolve the data eye at 6400 MT/s, which requires a bandwidth of approximately 5-6 GHz (roughly 1.5x to 2x the Nyquist frequency). The receiver is a sense amplifier or comparator that compares the incoming DQ signal against VREF and produces a full-swing digital output.

The receiver architecture typically consists of an input stage, a regeneration stage, and an output buffer. The input stage is a differential pair (or pseudo-differential pair) with the DQ signal on one input and VREF on the other. The tail current source sets the bias current and determines the gain-bandwidth product. The regeneration stage is a cross-coupled latch that amplifies the small differential voltage from the input stage to full digital levels. The output buffer drives the internal logic (deserialiser or delay line).

For LPDDR5 at 0.5V VDDQ, the input signal swing is only 500 mV rail-to-rail, with the receiver needing to resolve differences of approximately 250 mV (VDDQ - VREF) at the sampling instant. After accounting for ISI, crosstalk, and noise, the actual voltage margin at the receiver may be only 50-100 mV, requiring high receiver sensitivity.

The receiver bandwidth is primarily limited by the parasitic capacitance at the input node. This capacitance includes the ESD protection capacitance (100-300 fF), the pad and bump capacitance (50-100 fF), and the receiver input transistor capacitance (20-50 fF). The total input capacitance of 170-450 fF, combined with the source impedance of 40-50 ohm, gives an RC time constant of 7-22 ps, corresponding to a bandwidth of 7-22 GHz. While this appears sufficient, the actual bandwidth is reduced by parasitic inductance (from bond wires or bumps) and by the loading of the ODT circuit when it is active.

For layout, minimising the parasitic capacitance at the receiver input is critical. The interconnect from the IO pad to the receiver input transistors must be as short as possible, avoiding unnecessary metal stubs or wide routing that adds capacitance. The ESD structures should be placed to minimise the additional capacitive loading on the signal path.

---

### Q8. What is the role of the duty cycle corrector (DCC) in the IO cell, and where is it placed in the layout?

**Answer:**

The duty cycle corrector (DCC) adjusts the duty cycle of the DQS strobe (and sometimes CK and WCK) to ensure it is as close to 50% as possible. A perfect 50% duty cycle means that the high time and low time of the clock are equal, providing identical sampling windows for data captured on the rising and falling edges.

Duty cycle distortion (DCD) can arise from several sources: asymmetry in the PLL output buffers (different PMOS and NMOS drive strengths causing different rise and fall times), asymmetry in the routing (different capacitive loading on the true and complement paths of a differential pair), voltage-dependent threshold shifts in the receivers, and temperature-dependent transistor parameter drift.

The DCC typically operates by measuring the duty cycle (using an integrator that averages the high and low times) and adjusting a variable delay or variable drive strength to correct any imbalance. Digital implementations use a delay line that separately adjusts the rising and falling edge positions. Analog implementations use an offset current or variable load to adjust the crossing point of the differential signal.

For LPDDR5, the JEDEC specification requires the CK duty cycle to be within 45-55% (tCH and tCL parameters). At 6400 MT/s, a 5% duty cycle error corresponds to approximately 8 ps of timing asymmetry, which is a significant fraction of the total timing budget.

In the layout, the DCC is placed on the DQS receive path (for read data sampling) and on the CK/WCK output path (for transmit clocking). The DCC for DQS receive is placed between the DQS input pad and the sampling flip-flops, typically after the input buffer but before the per-bit deskew delay lines. The DCC for CK/WCK output is placed after the PLL output divider but before the output driver.

The DCC must be placed close to the circuit it corrects, as any routing between the DCC and the corrected signal introduces additional asymmetry that could partially undo the correction.

---

### Q9. How is the IO cell arranged physically at the die edge, and what determines the cell pitch?

**Answer:**

IO cells are arranged in a row along the die edge, with each cell occupying a fixed width (cell pitch) along the edge and a fixed depth into the die. The cell pitch is determined by several factors.

The bump or pad pitch is the primary constraint. For PoP applications with 0.4-0.5 mm ball pitch, the IO cell pitch must align with the bump pitch. Since multiple bumps (signal, power, ground) must be accommodated, the IO cell pitch is typically related to the bump pitch by an integer ratio. For example, with 0.5 mm bump pitch and two bumps per IO cell (one signal bump and one shared power/ground bump), the IO cell pitch is 0.5-1.0 mm.

The transistor width required for the driver determines the minimum cell width. A 40-ohm LPDDR5 driver requires significant transistor width, and the segmented architecture with calibration overhead increases this further. In advanced process nodes (5-7nm), the minimum IO cell width is typically 15-40 um, well below the bump-pitch-limited width.

The power grid requirements also affect cell pitch. Each IO cell needs connections to VDDQ, VSS, and possibly VDD1/VDD2 power rails. The metal straps for these rails must have sufficient width to carry the switching current without excessive IR drop. This sets a minimum metal pitch between cells.

The cell depth (from the die edge inward) is determined by the number of circuit elements stacked behind the pad. A typical IO cell depth is 60-120 um, accommodating the ESD structures, the driver and receiver transistors, the pre-driver, level shifters, and the first stage of the serialiser/deserialiser.

In the layout, IO cells are instantiated as hard macros (pre-designed and characterised blocks) that are abutted along the die edge. The abutment boundaries must be defined so that power rails connect cleanly between adjacent cells, well and substrate taps are shared to save area, and signal routing channels between cells are available for the serialiser and FIFO connections.

---

### Q10. What are the key differences between DQ and CA IO cells in terms of design and layout?

**Answer:**

While DQ and CA IO cells share the same fundamental architecture (push-pull driver, receiver, ODT, ESD), they differ in several important ways that affect their layout.

Data rate and bandwidth differ between the two types. DQ operates at the full data rate (e.g., 6400 MT/s for LPDDR5), while CA operates at the CK rate, which is 2x to 4x lower (e.g., 1600 MHz for CK at WCK:CK=4:1). This means CA IO cells can have lower bandwidth requirements, potentially allowing smaller transistors and more ESD capacitance tolerance.

Bidirectionality is another key difference. DQ cells are bidirectional (transmit during writes, receive during reads), requiring both a driver and a receiver with ODT. CA cells are unidirectional on the SoC side -- they only transmit. The DRAM's CA receiver does not need a driver for normal operation (though it may have a loopback mode for testing). This means the SoC CA IO cell may omit the receiver and ODT circuits, saving area.

Drive strength requirements may differ. CA signals may use different impedance targets than DQ signals. LPDDR5 allows separate programmability for CA and DQ drive strengths.

Termination for CA is handled differently. LPDDR5 introduces CA-ODT on the DRAM side, but the SoC CA driver does not have an ODT function since it only transmits. The CA driver impedance is set by ZQ calibration.

For layout, the CA IO cells are typically narrower than DQ cells because they lack the receiver and ODT circuits. The CA cells may be grouped together in a dedicated CA lane, separate from the DQ byte lanes, with their own clock distribution (CK) and training circuits.

The CA lane placement must ensure that all CA signals within a channel are closely matched in length to CK. The CK differential pair is often placed centrally within the CA group so that the longest CA route is minimised. WCK, while not part of the CA bus per se, is often grouped near the CA lane for routing convenience since it shares the same channel.

---

See also:
- [PHY Architecture](phy_architecture.md)
- [Termination and ODT](termination_and_odt.md)
- [Worked Problem: Driver Sizing](worked_problems/problem_02_driver_sizing.md)
- [Power Grid for Memory IO](../03_layout_fundamentals/power_grid_for_memory_io.md)
