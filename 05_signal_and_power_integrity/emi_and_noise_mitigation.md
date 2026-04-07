# EMI and Noise Mitigation

This section covers electromagnetic interference (EMI), simultaneous switching output/receiver (SSO/SSR) noise, and noise mitigation techniques for LPDDRx interfaces.

---

### Q1. What is SSO/SSR noise, and why is it a critical concern for LPDDR5?

**Answer:**

SSO (Simultaneous Switching Output) noise occurs when multiple output drivers switch simultaneously, creating large transient currents through the shared power and ground connections. SSR (Simultaneous Switching Receiver) noise is the analogous effect when multiple receivers switch simultaneously, though this is typically smaller than SSO because receivers draw less current than drivers.

For LPDDR5, SSO noise is critical because the IO voltage is only 0.5V, the data rate is high (6400-8533 MT/s, creating fast di/dt), multiple pins switch simultaneously (8 DQ per byte, potentially 32 DQ across all channels), and the noise directly reduces the signal margin that is already tight.

The worst-case SSO scenario occurs when all 8 DQ pins in a byte switch in the same direction (all 0-to-1 or all 1-to-0) simultaneously. This creates the maximum instantaneous current through the VDDQ and VSS paths. The current spike causes VDDQ to droop and VSS to bounce, effectively reducing the available signal swing. For 8 pins switching 6.25 mA each in 100 ps, the total di/dt is 5 x 10^8 A/s. With 50 pH of power path inductance, the noise voltage is 25 mV (5% of VDDQ).

SSR noise occurs during read operations when the DRAM drives data and the SoC receivers sample it. The receiver switching current is typically 10-20% of the driver current, so SSR noise is smaller but still contributes to the total noise budget.

For layout, SSO/SSR mitigation focuses on reducing the power path inductance (more bumps, wider straps, shorter connections to decoupling), increasing local decoupling (MIM/MOM capacitors close to the IO cells), and controlling the switching pattern through slew rate control and staggered timing.

---

### Q2. What is ground bounce, and how does it differ from VDDQ droop?

**Answer:**

Ground bounce and VDDQ droop are two manifestations of the same fundamental phenomenon (Ldi/dt noise on the power supply), but they affect the circuit differently because they act on different reference voltages.

Ground bounce occurs on the VSS (ground) rail when NMOS pull-down transistors switch simultaneously, driving current from the output pad through the NMOS transistors to VSS. The transient current through the VSS path inductance creates a voltage spike on the local VSS, lifting it above the ideal 0V. This shifts the ground reference for all circuits connected to that local VSS node.

VDDQ droop occurs on the VDDQ rail when PMOS pull-up transistors switch simultaneously, drawing current from VDDQ through the PMOS transistors to the output pad. The transient current through the VDDQ path inductance creates a voltage drop on local VDDQ, pulling it below the nominal 0.5V.

Both effects reduce the effective signal swing. If VSS bounces up by 15 mV and VDDQ droops by 15 mV simultaneously (worst case when some pins drive high while others drive low), the effective VDDQ-VSS at the IO cell is only 0.5V - 15 mV - 15 mV = 470 mV. The signal swing is reduced to 470 mV, and VREF (at VDDQ/2) shifts because it references the noisy supplies.

Ground bounce is often more problematic than VDDQ droop because the VSS rail is shared between the IO domain and the core logic domain. Ground bounce in the IO region can propagate to the core logic, potentially causing timing violations in the FIFOs, training logic, or even the PLL.

For layout, ground bounce mitigation requires providing a low-inductance VSS path specifically for the IO cells, with dedicated VSS bumps close to the IO region. The IO VSS should be isolated from the core VSS through separate bump connections and power grid routing, with reconnection only at the package level or through dedicated tie points with filtering.

---

### Q3. How does spread-spectrum clocking (SSC) affect LPDDR interfaces?

**Answer:**

Spread-spectrum clocking (SSC) is a technique that intentionally modulates the clock frequency over a small range to spread the spectral energy of the clock and its harmonics across a wider bandwidth. This reduces the peak EMI at any single frequency, helping the system meet electromagnetic compatibility (EMC) regulatory requirements.

For LPDDR interfaces, SSC is applied to the system clock (CK) by modulating the PLL frequency. Typical SSC parameters are a modulation depth of 0.5-1.0% (meaning the frequency varies by plus or minus 0.5-1.0% from the nominal), a modulation frequency of 30-50 kHz (the rate at which the frequency sweeps), and a modulation profile (typically triangular or Hershey-kiss shaped).

SSC affects LPDDR timing because the clock frequency is no longer constant. At any instant, the actual UI may be slightly longer or shorter than the nominal UI. The training algorithms must account for this variation: the read and write timing must be valid across the range of clock frequencies. The per-bit deskew and write leveling settings must have sufficient margin to cover the frequency modulation range.

For layout, SSC does not directly change the routing requirements, but it affects the timing analysis. The timing budget must account for the worst-case UI (shortest UI, at maximum frequency) rather than the nominal UI. For 1% SSC, the worst-case UI is 1% shorter than nominal, consuming approximately 1.56 ps of timing budget at 6400 MT/s.

SSC also complicates SI simulation because the channel response varies slightly with frequency. However, the 1% frequency modulation is small enough that the channel characteristics are nearly constant across the SSC range, and single-frequency simulation at the worst-case speed is typically sufficient.

---

### Q4. How does shielding mitigate EMI in the LPDDR PHY area?

**Answer:**

Shielding in the LPDDR PHY area takes several forms, each targeting different noise coupling mechanisms. On-die metal shielding uses grounded (VSS) metal traces or planes adjacent to signal traces. Lateral shields (VSS traces between signal groups) reduce capacitive crosstalk between adjacent signal groups. Vertical shields (VSS planes on layers above or below signal routing) reduce coupling between layers and provide a well-defined return current path.

The effectiveness of on-die shielding depends on the shield continuity (shields must be connected to VSS through vias at regular intervals, typically every 50-200 um, to remain at ground potential at the frequencies of interest), the shield coverage (the shield must extend along the full length of the protected signal, with no gaps), and the shield proximity (the closer the shield is to the signal, the more effective the shielding, but also the more capacitance it adds to the signal).

For LPDDR5 PHY, shielding is typically applied between byte lanes (to prevent inter-byte crosstalk), around the DQS and CK/WCK differential pairs (to protect the most timing-sensitive signals), between the DQ/DQS routing and the CA/CK routing (to prevent data-to-command coupling), and around the PLL block (to isolate it from IO switching noise).

Package-level shielding uses ground planes in the package substrate to isolate signal routing layers. The SoC package substrate typically has dedicated ground layers between signal routing layers. The PoP structure may also include ground planes that shield the TMV interconnect.

Board-level shielding (for non-PoP applications) uses ground planes on the PCB and may include physical shields (metal enclosures) over the LPDDR region to contain radiated EMI.

---

### Q5. What is the impact of SSO noise on timing, and how is it quantified?

**Answer:**

SSO noise impacts timing through several mechanisms that can be quantified through simulation. Supply-induced jitter occurs when VDDQ or VSS noise modulates the driver or receiver delay. The timing sensitivity is typically 1-5 ps per mV of supply noise. For 25 mV of SSO noise, this translates to 25-125 ps of timing uncertainty, which is extremely significant relative to the 156.25 ps UI.

Threshold shift occurs when the SSO noise shifts the receiver VREF. If VREF tracks VDDQ (as in a ratio-metric VREF generator), a VDDQ droop shifts VREF downward, effectively moving the sampling point within the eye. The timing impact depends on the eye slope at the VREF level.

Return current coupling occurs when the SSO current flowing through the shared ground creates voltage drops that couple between signal paths. If two signals share a ground return path with finite impedance, the switching current of one signal creates a noise voltage that appears on the other signal's ground reference.

Quantification of SSO timing impact is performed through PI-aware SI simulation. The simulation flow extracts the PDN model (power grid + decoupling + package), simulates the worst-case switching current pattern to determine the VDDQ and VSS noise waveforms, and then applies these noise waveforms as supply perturbations during the SI simulation. The eye diagram generated with the noisy supply shows the combined SI+PI impact, which is always worse than SI-only or PI-only analysis.

For layout, the SSO timing impact is minimised by reducing the power path impedance (so that the same switching current creates less voltage noise), filtering the VREF supply (so that VREF does not track the high-frequency VDDQ noise), and isolating the clock generation supply (so that SSO noise does not create jitter on CK/DQS).

---

### Q6. How does simultaneous switching affect the ZQ calibration accuracy?

**Answer:**

ZQ calibration relies on a precise voltage measurement at the ZQ pad. If SSO noise from the IO cells couples onto the ZQ pad or its power supply during calibration, the calibration result will be corrupted.

The coupling paths include substrate noise (SSO noise injects current into the substrate, which propagates to the ZQ block), power supply noise (SSO noise on VDDQ propagates through the power grid to the ZQ block's supply), and capacitive coupling (if the ZQ pad routing runs near DQ or DQS signal routes, capacitive crosstalk can inject noise directly onto the ZQ analog node).

The impact on calibration accuracy depends on the noise magnitude relative to the calibration comparator's resolution. A typical ZQ comparator has a resolution of 2-5 mV (corresponding to approximately 1% impedance accuracy). If SSO noise exceeds this resolution, the calibration code will fluctuate, leading to a noisier impedance setting that varies with data activity.

Mitigation strategies include performing ZQ calibration during quiet periods (when the IO is not actively switching data), which is the approach used by the ZQCL/ZQCS commands issued by the memory controller. Averaging multiple calibration measurements to filter out noise, using a dedicated quiet power supply for the ZQ block with filtering from the main VDDQ grid, and physically isolating the ZQ block from the IO cells in the layout (placing it at the end of the PHY, away from the byte lanes, with guard rings and shielded routing).

For layout, the ZQ pad routing should use a dedicated metal track with ground shields on both sides, avoiding any parallel run with DQ/DQS signals. The ZQ block's supply connections should tap the VDDQ grid at a quiet point (away from the byte lanes) and include local decoupling capacitors (MIM caps) to filter high-frequency noise.

---

### Q7. How do DQ data patterns affect noise levels, and which patterns are worst case?

**Answer:**

The data pattern on the DQ bus determines the switching activity and thus the magnitude of SSO noise, crosstalk, and ISI. Different patterns create different stress conditions.

All-same-direction switching (all 8 DQ bits transition 0-to-1 or all transition 1-to-0 simultaneously) creates the maximum SSO noise because all drivers draw current from the same supply rail at the same time. This is the worst case for VDDQ droop or VSS bounce.

Checkerboard pattern (alternating 0101 spatially across DQ bits, and alternating in time) creates the maximum crosstalk stress because every signal has an adjacent aggressor transitioning in the opposite direction.

Clock-like pattern (0101 repeating in time on each DQ bit) creates the maximum sustained switching power and the highest average current demand on VDDQ. This stresses the DC IR drop and thermal conditions.

Low-frequency pattern (long runs of same value followed by a transition) creates the worst ISI because the channel has time to charge up during the long run, and the subsequent transition must overcome this stored energy. The first bit after a long run has the worst voltage margin.

For SI simulation, the recommended approach is to use a PRBS (Pseudo-Random Bit Sequence) pattern that statistically covers all possible patterns. The PRBS length should be at least 2^7 - 1 = 127 bits for adequate coverage. For worst-case analysis, specific patterns (all-same-direction, checkerboard) should also be simulated explicitly.

For layout, the worst-case pattern for power grid design is the all-same-direction pattern (maximum current). The worst-case for SI is typically the low-frequency pattern (maximum ISI) or the checkerboard pattern (maximum crosstalk), depending on which mechanism dominates.

---

### Q8. How is EMI compliance tested for products with LPDDR interfaces?

**Answer:**

EMI compliance testing ensures that the electromagnetic emissions from the LPDDR interface do not exceed regulatory limits (such as FCC Part 15, CISPR 22/32, and EN 55032). The LPDDR interface is a significant source of emissions because it has high-speed signals (GHz-range fundamental frequencies), large signal counts (32+ DQ pins plus clocks), and the PoP or PCB interconnect can act as an antenna radiating the high-frequency energy.

Testing is performed at the system level (complete product) in an aneclosed chamber or semi-aneclosed room. The product is operated in worst-case EMI mode (maximum data activity on the LPDDR interface), and the radiated emissions are measured using broadband antennas and a spectrum analyser or EMI receiver. The measurements are compared against the applicable regulatory limits.

For LPDDR interfaces, the primary emission sources are clock harmonics (CK and WCK harmonics at multiples of the clock frequency can fall within the regulated frequency bands), data pattern emissions (the PRBS-like data creates a broadband emission spectrum centred at the data rate), and power supply noise that couples to the board ground plane and radiates.

Mitigation techniques applied during layout and system design include spread-spectrum clocking (as discussed earlier), controlled impedance routing (to minimise reflections that create standing waves on the interconnect, acting as antenna), power/ground plane continuity (minimising slots or gaps that could act as slot antennas), shield planes in the package substrate, and proper decoupling to reduce high-frequency current in the package and board loops.

For layout engineers, the primary EMI-related layout decisions are slew rate control (slower transitions reduce high-frequency content but narrow the eye), signal routing near the die centre (rather than at edges where radiation is more efficient), and continuous reference planes under all high-speed routing.

---

### Q9. How does the package substrate design affect EMI from the LPDDR interface?

**Answer:**

The package substrate is a critical element in the EMI performance of the LPDDR interface because it contains the high-frequency signal routing between the die and the external interconnect (TMVs or PCB). The substrate design affects EMI through several mechanisms.

Cavity resonance can occur within the package substrate if the ground and power planes form a parallel-plate cavity. At frequencies where the substrate dimensions equal a half-wavelength (typically 10-30 GHz for common package sizes), the cavity resonates and amplifies noise, creating emission peaks.

Via fencing (rows of ground vias around the substrate perimeter) acts as a Faraday cage that contains electromagnetic fields within the substrate and reduces radiation from the substrate edges. The via spacing must be less than one-quarter wavelength at the highest frequency of concern.

Power plane design in the substrate affects the return current path for high-speed signals. A continuous ground plane provides a low-impedance return path directly under the signal trace, minimising the current loop area and thus the radiation. Slots or splits in the ground plane force the return current to detour, increasing the loop area and radiation.

Decoupling capacitors embedded in the package substrate or placed on the package surface provide mid-frequency decoupling that reduces the high-frequency current flowing through the package, reducing both conducted and radiated emissions.

For layout, the die-level decisions that affect package EMI include the bump map design (which determines the package routing topology), the signal-to-ground bump ratio (which affects the quality of the return current path in the substrate), and the placement of power and ground bumps (which affects the substrate power plane integrity).

---

### Q10. What noise mitigation techniques are specific to the LPDDR PHY boundary with core logic?

**Answer:**

The boundary between the LPDDR PHY (operating in the VDDQ domain with high-speed IO switching) and the adjacent core logic (operating in the core voltage domain with digital switching) is a critical region for noise management.

Voltage domain isolation requires level shifters at every signal crossing between VDDQ and core VDD domains. The level shifters must be designed to not propagate noise from one domain to the other. The level shifter layout must include local decoupling on both supply rails to absorb transient current during switching.

Guard rings at the domain boundary prevent substrate noise coupling. The LPDDR IO cells inject noise into the substrate through their drain junctions during switching. Without guard rings, this noise propagates through the substrate to nearby core logic, potentially causing timing violations or functional errors. A continuous P+ guard ring (connected to VSS) and N+ guard ring (connected to the appropriate VDD) should surround the PHY boundary.

Power grid isolation ensures that the VDDQ grid noise does not corrupt the core VDD through resistive or inductive coupling. The VDDQ and core VDD power grids should be on separate metal layers where possible, with no direct connections (each domain has its own bumps and distribution). The VSS grid may be shared but should have low impedance to prevent ground bounce in one domain from affecting the other.

Clock isolation is essential for the PLL. The PLL generates clocks for the PHY but is sensitive to noise from both the IO domain and the core domain. The PLL should be placed at the boundary with its own isolated power supply and surrounded by guard rings that isolate it from both domains.

Decoupling at the boundary is important to prevent noise from propagating across the domain transition. MIM or MOM capacitors placed along the boundary, connected between each domain's VDD and the shared VSS, absorb high-frequency noise at the source.

---

See also:
- [SI for LPDDR](si_for_lpddr.md)
- [Power Integrity for LPDDR](power_integrity_for_lpddr.md)
- [Worked Problem: SSR Noise Analysis](worked_problems/problem_03_ssr_noise_analysis.md)
