# Termination and ODT

This section covers on-die termination (ODT) and impedance calibration in LPDDRx interfaces. Termination is critical for signal integrity at high data rates, and the layout must support the various termination modes and calibration requirements.

---

### Q1. Why is on-die termination (ODT) necessary for LPDDR interfaces, and how does it differ from board-level termination?

**Answer:**

On-die termination is necessary because LPDDR interfaces use point-to-point connections between the SoC and DRAM in a Package-on-Package (PoP) configuration, with very short interconnect lengths (typically 2-5 mm total path through the package stack). At these short distances, there is no practical location to place discrete termination resistors on a PCB -- the entire signal path is within the package. Any impedance mismatch at the receiver end causes reflections that travel back to the transmitter and return in a very short time, potentially overlapping with subsequent data bits and causing inter-symbol interference (ISI).

ODT integrates the termination resistor directly into the silicon die, at the receiver input. When activated, the ODT circuit presents a controlled impedance (typically 40, 48, 60, 80, 120, or 240 ohm) that matches the transmission line impedance, absorbing the incoming signal energy and preventing reflections.

Board-level termination, used in DDR DIMM systems, places discrete resistors on the PCB near the receiver end of the trace. This is effective for longer traces (50-150 mm on a motherboard) where the propagation delay is long enough to separate incident and reflected waves. Board termination allows precise resistance values and power dissipation in external components.

The advantage of ODT is that it eliminates the need for external components, saving board area and reducing the interconnect stub to essentially zero. The disadvantage is that the termination resistance is less precise (it depends on transistor process variation, which is compensated by ZQ calibration) and it dissipates power within the die (heating the silicon).

For layout engineers, ODT means the IO cell must include the termination transistors, their calibration circuitry, and the mode control logic. The layout must ensure that the termination path (from pad through the ODT transistors to the supply rail) has low parasitic inductance, as any inductance in this path reduces the effectiveness of the termination at high frequencies.

---

### Q2. What are the different ODT modes in LPDDR5, and when is each mode activated?

**Answer:**

LPDDR5 defines several ODT modes that are activated in different operating scenarios.

NT-ODT (Non-Target ODT) is activated on a DRAM rank that is not the target of the current command. In a dual-rank system, when data is being written to rank 0, rank 1 activates NT-ODT to terminate its DQ inputs and prevent reflections from the idle rank's high-impedance input. NT-ODT values are typically 60-240 ohm, set via mode registers (MR11).

WR-ODT (Write ODT) is activated on the target rank during write operations. When the SoC drives write data, the target DRAM activates WR-ODT to provide proper termination for the incoming signal. The WR-ODT value is set via mode registers and is typically 40-120 ohm.

CA-ODT (Command/Address ODT) is new in LPDDR5 and provides termination for the CA bus at the DRAM receiver. Since the CA bus operates at double data rate in LPDDR5 (higher frequency than LPDDR4), CA-ODT improves signal quality by terminating reflections on the command bus. CA-ODT is activated during normal operation whenever the DRAM is receiving commands.

Park termination refers to the ODT state when no specific ODT mode is commanded. In LPDDR5, the DQ termination can be configured to a "park" value that is always active, providing a baseline termination regardless of the command state. This simplifies the ODT control logic and ensures consistent impedance at the receiver.

SOC-side ODT is also present: during read operations, the SoC PHY activates its own ODT on the DQ pins to terminate the data driven by the DRAM. This SOC-side ODT is controlled by the PHY training logic and is independent of the DRAM ODT modes.

For layout, the different ODT modes mean the IO cell must support multiple impedance values that can be switched dynamically. The switching between modes must be fast (within a few clock cycles) and glitch-free to avoid signal integrity issues during transitions.

---

### Q3. How does ZQ calibration work in detail, and what are the critical layout requirements for the ZQ pad?

**Answer:**

ZQ calibration is a feedback-based impedance matching process. The ZQ pad on the DRAM (and sometimes on the SoC) is connected to an external precision resistor (typically 240 ohm, 1% tolerance) that provides an impedance reference.

The calibration process uses a replica of the output driver connected to the ZQ pad. The calibration controller adjusts the driver's impedance code (a digital value that controls how many parallel transistor legs are enabled) while monitoring the voltage at the ZQ pad. When the driver impedance matches the external resistor, the ZQ pad voltage settles to VDDQ/2 (for a pull-up driver calibrating against a pull-down resistor to ground, or vice versa). A comparator detects this condition and locks the calibration code.

The calibration is performed in two phases: pull-up calibration (adjusting the PMOS network to match the external resistor) and pull-down calibration (adjusting the NMOS network to match the calibrated PMOS). This two-step process ensures both networks are matched.

Critical layout requirements for the ZQ pad include a dedicated bump and routing path. The ZQ pad must have its own package bump with a clean, short connection to the external resistor. The routing on the die from the replica driver to the ZQ bump must have minimal and predictable parasitic resistance and capacitance, as these parasitics affect the calibration accuracy. Noise isolation is essential because the ZQ pad voltage must be stable during calibration. Any noise coupling onto the ZQ pad (from switching DQ signals, power supply ripple, or substrate noise) will corrupt the comparison and produce an incorrect calibration code. The routing should be shielded and separated from high-speed signals by adequate spacing. The comparator placement must be close to the ZQ pad to minimise the routing length on the sensitive analog node. The comparator must have input offset less than a few millivolts to achieve the required calibration accuracy (typically plus or minus 5% of the target impedance). The replica driver matching requires that the replica driver connected to ZQ be laid out identically to the actual IO drivers, using the same unit cell dimensions, metal routing, and orientation. Any systematic difference between the replica and actual drivers creates a persistent impedance error.

---

### Q4. What impedance values are typically supported for LPDDR5 ODT, and how do they affect signal integrity?

**Answer:**

LPDDR5 supports a range of programmable ODT values, typically including 40, 48, 60, 80, 120, and 240 ohm for both the DQ and CA interfaces. The values are set independently for each ODT mode (WR-ODT, NT-ODT, CA-ODT, Park) through mode register writes. The SoC-side ODT also supports a similar range of values.

The choice of ODT value affects signal integrity through several mechanisms. Impedance matching is the primary consideration: when the ODT value matches the characteristic impedance of the transmission line (typically 40-50 ohm for LPDDR5), reflections are minimised. However, perfect matching is not always optimal because the PoP interconnect has distributed parasitics that change the effective impedance.

Voltage swing is affected because the ODT forms a voltage divider with the driver impedance. For a 40-ohm driver driving into a 40-ohm ODT, the received voltage swing is VDDQ/2 (250 mV for 0.5V VDDQ). Choosing a higher ODT value increases the voltage swing (less division) but at the cost of increased reflections.

Power dissipation scales inversely with ODT value. A lower ODT value draws more current from VDDQ when terminating. At 40 ohm with 0.5V VDDQ, the DC termination current is 12.5 mA per pin, or 100 mA for 8 DQ pins. This is a significant power contribution and also causes IR drop on the VDDQ supply.

For layout, the ODT value affects the power grid requirements. Lower ODT values draw more current, requiring a more robust VDDQ power grid with lower IR drop. The layout engineer must simulate the power grid under worst-case ODT activation (all DQ pins terminated at the lowest ODT value) to ensure adequate voltage margin.

The practical approach in LPDDR5 design is to use simulation to sweep ODT values and find the combination that maximises the eye opening at the receiver, considering the specific channel characteristics (trace length, via count, coupling, package parasitics). The selected ODT values are then programmed into the mode registers during initialisation.

---

### Q5. What is impedance calibration drift, and how does the layout support periodic recalibration?

**Answer:**

Impedance calibration drift occurs because the on-resistance of the driver and ODT transistors changes with temperature. As the die heats up during operation, the transistor mobility decreases, increasing the on-resistance. Conversely, cooling decreases the on-resistance. The temperature coefficient of MOSFET on-resistance is typically 0.3-0.5% per degree Celsius, meaning a 50-degree temperature change can shift the impedance by 15-25%.

To compensate, LPDDR5 supports periodic ZQ calibration using the ZQCS (ZQ Calibration Short) command. ZQCS is issued by the memory controller at regular intervals (typically every 1-10 ms, depending on the expected temperature change rate) and takes a shorter time to complete (approximately 30 ns for ZQCS versus 1 us for the initial ZQCL). During ZQCS, the calibration circuit performs a quick adjustment to track temperature-induced drift.

Some LPDDR5 PHYs also implement background ZQ calibration on the SoC side, continuously adjusting the SoC driver impedance without interrupting data traffic. This is done by running the ZQ calibration on a separate time-multiplexed basis, using the replica driver during idle cycles.

For layout, periodic recalibration requires that the ZQ calibration block be always powered and accessible (it cannot be in a power-gated domain that might be shut down during normal operation). The calibration code distribution buses from the ZQ block to all IO cells must support dynamic updates -- the new code is loaded into the IO cells while they continue normal operation, with the update occurring between data bursts.

The layout must also consider thermal gradients across the die. The ZQ calibration produces a single impedance code based on the temperature at the ZQ pad location. If the IO cells at different positions on the die experience different temperatures (due to hot spots from nearby logic blocks), the calibrated impedance will not be optimal for all cells. The layout engineer should place the ZQ block near the centre of the IO cell array and design the floorplan to minimise thermal gradients across the PHY region.

---

### Q6. How does CA-ODT work in LPDDR5, and why was it introduced?

**Answer:**

CA-ODT (Command/Address On-Die Termination) was introduced in LPDDR5 to improve signal integrity on the CA bus, which now operates at double data rate (DDR) compared to the single data rate (SDR) CA bus in LPDDR4. At DDR rates, the CA bus toggles at the CK frequency (up to 2133 MHz for LPDDR5X at 8533 MT/s), making it susceptible to the same reflection and ISI issues that affect the DQ bus.

CA-ODT is implemented on the DRAM side as a termination resistor at the CA input pins. The ODT value is programmable (typically 40, 60, 80, 120, or 240 ohm) and is configured via mode register MR11. Unlike DQ ODT, which is switched between different modes during read and write operations, CA-ODT is typically activated continuously during normal operation because the CA bus is always receiving commands from the SoC.

The introduction of CA-ODT was necessary because the PoP interconnect, while short, has non-trivial impedance discontinuities at the solder bump, through-mold via, and substrate routing transitions. Without termination, these discontinuities cause reflections that can overlap with the next CA bit (at DDR rates, the UI for CA is 1/CK, which is 2-4x the DQ UI but still only 470ps at 2133 MHz).

For layout, CA-ODT on the SoC side means the CA output drivers must be designed to drive into the terminated load. The driver impedance, combined with the CA-ODT value, determines the voltage swing at the DRAM receiver. For a 40-ohm driver and 40-ohm CA-ODT, the received swing is VDDQ/2. The CA driver must be sized appropriately for this loading condition, and the power grid must handle the additional DC current drawn by CA-ODT termination.

The SoC CA routing does not have ODT because the CA is unidirectional (SoC drives, DRAM receives). However, the characteristic impedance of the CA traces must be well controlled to minimise reflections from the unterminated SoC end.

---

### Q7. How does the ODT switching affect power supply noise, and what layout mitigation is needed?

**Answer:**

ODT switching creates transient current demands on the VDDQ supply that can cause voltage droop and ringing. When ODT is activated (e.g., transitioning from idle to a write operation where WR-ODT turns on), the termination resistors begin drawing DC current from VDDQ. Conversely, when ODT is deactivated, the current ceases. These step changes in current create Ldi/dt voltage noise on the VDDQ supply.

The magnitude of the current step depends on the ODT value and the number of pins being terminated. For 8 DQ pins with 40-ohm ODT to VDDQ (assuming DQ is at VSS level, worst case), the total current step is 8 x (0.5V / 40 ohm) = 100 mA. If the PDN (Power Distribution Network) has an inductance of 100 pH from the decoupling capacitor to the ODT circuit, the voltage droop is L x di/dt. For a current step of 100 mA in 1 ns (typical ODT turn-on time), the droop is 100 pH x 100 mA / 1 ns = 10 mV, which is 2% of VDDQ -- a significant portion of the noise budget.

Layout mitigation strategies include several approaches. Minimising PDN inductance requires placing decoupling capacitors (MIM or MOM) as close as possible to the IO cells, reducing the inductive loop area between the capacitor and the ODT transistors. Using multiple parallel vias for VDDQ and VSS connections also reduces inductance. Increasing decoupling capacitance is important because more capacitance near the IO cells acts as a local charge reservoir that can supply the ODT current without drawing it from the distant package bumps. Staggering ODT activation can help: if the system allows, activating ODT on different pins at slightly different times spreads the current step over a longer period, reducing the peak di/dt. Dedicated ODT power routing is sometimes used: some designs route a separate VDDQ supply to the ODT circuits with its own decoupling, isolating the ODT current transients from the signal driver supply.

The layout engineer should perform transient power integrity simulation with ODT switching events to verify that the VDDQ droop is within the noise budget.

---

### Q8. What is the relationship between driver impedance and ODT impedance for optimal signal integrity?

**Answer:**

The relationship between driver output impedance (Zo_driver) and receiver ODT impedance (Zo_ODT) determines the signal amplitude, reflection coefficients, and power consumption at the interface. For a transmission line with characteristic impedance Zo, the reflection coefficient at the receiver is:

```
Gamma_receiver = (Zo_ODT - Zo) / (Zo_ODT + Zo)
```

And the reflection coefficient at the driver is:

```
Gamma_driver = (Zo_driver - Zo) / (Zo_driver + Zo)
```

For zero reflections at both ends, both impedances should equal Zo. However, in practice, the driver impedance is set to Zo (40-50 ohm) and the ODT impedance is also set to Zo, creating a double-terminated line. The received voltage swing is then VDDQ x Zo_ODT / (Zo_driver + Zo_ODT) = VDDQ/2 = 250 mV for 0.5V VDDQ.

If the ODT is set higher than Zo (e.g., 60 ohm ODT for 40 ohm line), the received swing increases to VDDQ x 60 / (40 + 60) = 300 mV, providing more voltage margin. However, the reflection coefficient at the receiver becomes (60 - 40)/(60 + 40) = 0.2, meaning 20% of the incident signal is reflected back. In the short PoP interconnect, this reflection returns quickly and can degrade signal quality.

The optimal driver-ODT combination depends on the specific channel. For short PoP interconnects with minimal loss, a matched termination (driver = ODT = Zo) is often best because the short round-trip time means reflections return quickly and overlap with the signal. For longer channels or channels with significant loss, a higher ODT value may be preferred because the channel loss attenuates the reflection and the increased voltage swing compensates for the loss.

For layout, the driver and ODT impedance choices affect the power grid design. Matched termination (40/40) draws the most current but produces the cleanest signal. The layout engineer should design the power grid for the worst-case current draw configuration and validate with the SI engineer's recommended impedance settings.

---

### Q9. How is ODT calibration coordinated between the SoC and DRAM during initialisation?

**Answer:**

ODT calibration involves both the SoC-side PHY and the DRAM-side circuits, and must be coordinated during the initialisation sequence. The process follows a defined sequence specified by JEDEC.

During power-up, the VDDQ, VDD1, and VDD2 supplies are ramped in the specified sequence. After supplies stabilise, the SoC issues a reset to the DRAM. Following reset, the SoC performs ZQ calibration on its own PHY (SoC-side ZQCL). The SoC then issues a ZQCL command to the DRAM (via the CA bus) to trigger the DRAM's ZQ calibration. Both sides are now calibrated to their target impedances.

After ZQ calibration, the SoC programs the DRAM mode registers with the desired ODT values for each mode (WR-ODT, NT-ODT, CA-ODT, Park termination). The SoC PHY is also configured with its own ODT values for the receive side. These values are typically determined during system characterisation (SI simulation) and stored in the SoC's firmware.

The initialisation sequence then proceeds to training: Command Bus Training (CBT), Write Leveling, Read Training, and VREF Training. During these training steps, the ODT values are active, so the training results reflect the actual terminated channel conditions.

For layout, the initialisation sequence has a layout implication: all the control signals needed for the initialisation (CA bus, CK, CS, and the DFI interface from the controller to the PHY) must be functional before any data transfer occurs. The layout must ensure that these control paths have adequate timing margin even before training, which means they cannot rely on per-bit deskew or VREF training for initial functionality. The CA routing and CK routing must be inherently well-matched (through layout length matching) to work reliably at the initial (slow) CK frequency used during initialisation.

---

### Q10. What are the power implications of different ODT configurations, and how does this affect the power grid design?

**Answer:**

The power dissipated by ODT circuits is a significant contributor to the total LPDDR interface power consumption. The DC power per pin for a terminated signal is:

```
P_ODT = V^2 / R_ODT (when signal is at opposite rail)
P_ODT_avg = V^2 / (4 x R_ODT) (average for random data)
```

For LPDDR5 at 0.5V VDDQ with 40-ohm ODT:

```
P_ODT_peak = 0.5^2 / 40 = 6.25 mW per pin
P_ODT_avg = 0.5^2 / (4 x 40) = 1.56 mW per pin
```

For 8 DQ pins per channel:

```
P_channel_ODT_avg = 8 x 1.56 = 12.5 mW per channel
P_total_ODT_avg (x32, 4 channels) = 50 mW
```

This is a significant power contribution in a mobile SoC where the total memory subsystem power budget may be 200-500 mW. Using higher ODT values (60 or 80 ohm) reduces this power by 33% or 50% respectively, at the cost of slightly less effective termination.

The ODT current flows through the VDDQ power grid, creating IR drop. The current distribution depends on which pins are terminated and the data pattern. Worst case occurs when all pins are driven low by the transmitter (maximum current through pull-up ODT to VDDQ) or all driven high (maximum current through pull-down ODT to VSS). This worst-case current must be supported by the power grid without exceeding the IR drop budget.

For layout, the power grid must be designed for the worst-case ODT scenario. This typically means the power grid must handle the sum of driver switching current and ODT DC current simultaneously. The IR drop analysis should include both the static ODT current and the dynamic switching current, modelled with appropriate activity factors.

The decoupling strategy must also account for ODT. When ODT switches on or off (during mode transitions between read and write), the current step creates a transient voltage disturbance. The decoupling capacitors must respond to this transient within the transition time (typically a few nanoseconds), requiring placement within a few hundred micrometres of the IO cells.

---

See also:
- [IO Cell Design](io_cell_design.md)
- [PHY Architecture](phy_architecture.md)
- [Power Integrity for LPDDR](../05_signal_and_power_integrity/power_integrity_for_lpddr.md)
- [Worked Problem: ODT Value Selection](worked_problems/problem_03_odt_value_selection.md)
