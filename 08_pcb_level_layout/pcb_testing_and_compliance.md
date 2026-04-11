# PCB Testing and Compliance for LPDDRx

This section covers the testing, measurement, and compliance verification of LPDDRx PCB designs. Unlike SerDes interfaces where the channel is characterised with BER and eye measurements directly on the signal, LPDDR is harder to probe at speed because of PoP stacking, dense BGA routing, and the lack of test-access features on DRAM packages. Most LPDDR verification happens indirectly through controller debug registers, training margin measurement, and system-level stress testing.

---

### Q1. What test and measurement equipment is used for LPDDRx PCB verification?

**Answer:**

LPDDR PCB verification uses a mix of general-purpose high-speed test equipment and LPDDR-specific tools. The equipment required depends on whether the goal is impedance characterisation, SI verification, timing margin measurement, or stress testing.

Time-domain reflectometry (TDR) tools such as the Tektronix DSA8300 with 80E10B TDR modules or Keysight N1055A are used for impedance profile measurement of PCB test coupons. They produce plots of impedance versus distance along a trace, identifying discontinuities at BGA pads, vias, and layer transitions. TDR is typically run on test coupons embedded in the PCB panel rather than on the production channel, because production channels are not accessible to probes.

Vector network analysers (VNAs) such as the Keysight PNA-X or R&S ZVA are used for S-parameter measurement of PCB test coupons, extracting insertion loss, return loss, and crosstalk curves from DC to 20-40 GHz. The extracted S-parameters feed back into correlation of simulation models against measured hardware.

Real-time oscilloscopes (Tektronix DPO70000SX, Keysight Infiniium UXR, LeCroy WaveMaster) with 20-50 GHz bandwidth and high-impedance probes are used where direct signal probing is possible. For LPDDR, this is rare: probing requires a custom interposer socket or a purpose-designed debug board because BGA-mounted DRAMs offer no probe points. Signal probing is more common during PHY silicon bringup than production PCB verification.

BERT (bit error rate tester) equipment is not commonly used for LPDDR. Unlike PCIe or 10GbE where BERT loopback testing is a standard compliance step, LPDDR has no concept of a BER test mode — the interface is a memory protocol, not a serial link, and the DRAM cannot be put into a pattern-generator loopback.

Purpose-built LPDDR debug tools such as JEDEC-compliant memory protocol analysers (FuturePlus, Keysight U4164A) capture bus traffic on a dedicated probe socket that replaces the DRAM with a pass-through interposer. These tools measure command timing, data timing, and protocol compliance. They are expensive and used for bringup and debug rather than routine production test.

The most important verification tool for LPDDR is the SoC memory controller's own debug and training registers, which are accessed through JTAG or a software debugger. These registers expose trained per-bit delays, DQS gate positions, VREF settings, and training margin windows — effectively giving an internal view of the eye that external probing cannot provide.

---

### Q2. What compliance standards and specifications apply to LPDDRx PCB designs?

**Answer:**

LPDDRx compliance is governed primarily by JEDEC standards, with additional requirements imposed by silicon vendors and end-system certification bodies.

JEDEC JESD209-4 (LPDDR4), JESD209-4-1 (LPDDR4X), JESD209-5 (LPDDR5), JESD209-5A/B/C (LPDDR5/5X updates) define the electrical, timing, and protocol specifications for each LPDDR generation. These documents specify AC and DC parameters: VDDQ levels and tolerances, VIH/VIL thresholds, input/output impedance ranges, tDQSCK and tAC timing parameters, refresh requirements, and temperature operating ranges. A PCB design is "LPDDR5 compliant" if the measured signals at the DRAM pins meet the JEDEC input specifications under the controller's output conditions.

The compliance is typically assessed indirectly. There is no formal JEDEC compliance program for LPDDR PCBs (unlike PCIe-SIG compliance for PCIe or Ethernet Alliance compliance for 10GbE). Instead, the SoC vendor provides a layout design guide (LDG) that specifies PCB impedance, length matching, decoupling, and layout rules derived from their internal characterisation of the PHY. Following the LDG is the de-facto compliance for most designs.

Silicon vendor layout design guides are documents such as Qualcomm's LPDDR layout guide, MediaTek's DRAM layout rules, or NVIDIA's memory interface design guide. They typically contain:

- Stackup requirements (minimum layer count, target impedance, dielectric class).
- Length matching rules for each matching group.
- Decoupling capacitor count, values, and placement.
- BGA fanout pattern recommendations or required patterns.
- Recommended thermal vias, ground stitching, and reference plane rules.
- Prohibited constructs (split reference planes, high-impedance sections, excessive via stubs).

System-level compliance may add further requirements: FCC and CE for EMI, AEC-Q100 for automotive, Mil-Std-810 for defence and aerospace. These affect the PCB thermal and EMC design but not the LPDDR electrical specification itself.

For a new PCB, compliance verification consists of: confirming the LDG rules are followed (design-time review), running simulation against the silicon vendor's channel budget, and measuring training margin on prototype hardware. There is no "golden eye mask" test against which an oscilloscope capture is graded in the way PCIe has compliance masks.

---

### Q3. How is LPDDR training margin measured, and why is it the best proxy for PCB SI quality?

**Answer:**

Training margin measurement reads back the results of the SoC memory controller's training algorithms and analyses how much headroom exists before errors appear. It is the most accurate measurement of real-world LPDDR PCB performance because it uses the actual PHY and DRAM in the actual system, measured at the actual operating conditions.

The measurement flow:

1. Boot the system and let the memory controller complete all training phases (CA training, write leveling, read DQ training, write DQ training, VREF training).
2. Read the trained parameter values from the controller debug registers. These include per-bit DQ delays (usually in units of delay-line steps, 5-10 ps each), DQS gate position, read and write VREF settings, and in LPDDR5X, DFE tap values.
3. For each trained parameter, sweep the parameter value above and below the trained point while running a memory test and observing whether errors occur.
4. Record the minimum and maximum values that still produce error-free operation — these define the passing window for that parameter.
5. The trained value should ideally be at the centre of the passing window; the "margin" is the distance from the trained value to the nearest failing edge.

Typical memory tests for margin measurement: march patterns (e.g. mbwtest, march C- from the academic literature), stress patterns that exercise worst-case switching (walking 1s, alternating 0101), and real-workload stress (running a full OS with applications, or synthetic benchmarks such as stress-ng).

A healthy LPDDR5 design will show training margins of 15-30% of UI in each direction for DQ delay, 10-20% of VDDQ for VREF, and 20-40 ps of DQS gate margin. Marginal designs show less than 10% of UI; failing designs show asymmetric windows (training pushed to one edge) or near-zero margin at PVT corners.

The power of training-margin measurement is that it captures everything: PCB SI, package, on-die PHY, training algorithm quality, voltage and temperature effects, and aging. It is PVT-coverable by repeating the measurement at the corners. It is quantitative — giving a number of picoseconds or millivolts of margin rather than a qualitative pass/fail. And it is repeatable, so the same board can be re-tested after any ECO.

The main limitation is that it requires SoC controller support. Not all controllers expose training margin sweep through the debug interface; on some, the user must rely on the trained values alone (which says the system works at nominal but gives no margin data). For production test of a well-understood PCB, training margin on a sample of units is usually sufficient.

---

### Q4. How does a shmoo plot work for LPDDR verification?

**Answer:**

A shmoo plot is a two-dimensional visualisation of system passing/failing as a function of two swept parameters. For LPDDR verification, the typical shmoo axes are VDDQ voltage versus temperature, data rate versus temperature, or DQ delay versus VREF. Each point on the plot represents a (parameter 1, parameter 2) combination, and the point is shaded pass or fail based on whether a memory test runs without errors at that setting.

The passing region forms a "shmoo" — an irregular closed shape whose boundary traces the failure modes of the memory interface. A healthy design has a large, contiguous passing region centred on the nominal operating point. A marginal design has a small passing region or a passing region that does not include the full specification range.

Typical shmoo axes for LPDDR:

- VDDQ voltage (x axis) versus temperature (y axis): traces the voltage-temperature operating envelope. Failing at high temperature with low voltage indicates retention or timing margin loss at hot slow corner. Failing at low temperature with high voltage indicates overshoot or crosstalk-induced errors.
- Data rate versus VDDQ: steps down the data rate and shows the minimum VDDQ at which each rate passes. Extracts the voltage-frequency curve of the interface.
- DQ delay versus VREF: traces the effective eye at the sampling point. Produces a two-dimensional eye diagram where the passing region is the open eye. Eye width and height are the two dimensions of the shmoo.
- Read DQS gate versus DQ delay: verifies DQS gating margin independently of DQ sampling.

Shmoo plots are produced by automated test: the test software loops over the parameter grid, sets the test parameters via controller registers or system configuration, runs a memory stress test at each point (typically 1-10 seconds long, enough to catch soft failures), and records the result. A full shmoo with 20 points on each axis takes 400 tests, or roughly 10-60 minutes depending on test length.

The shmoo plot is often used to compare designs or to track margin across manufacturing lots. A design with a "fat" shmoo is robust; a "thin" shmoo indicates a marginal design that may fail in production. Comparing shmoos across lots reveals whether process variation is pushing the population towards the failing edge.

Shmoos are also useful for diagnosing failures. A failure at high temperature and low VDDQ points to refresh or slow-corner timing; a failure at low temperature and high VDDQ points to overshoot or crosstalk; a failure on one DQ bit but not others points to a specific per-bit problem (routing, package, or silicon).

---

### Q5. How are PCB manufacturing variations managed for LPDDRx designs?

**Answer:**

PCB manufacturing variation affects LPDDR designs through trace geometry, dielectric properties, via structures, and assembly placement. The variations are statistical — most boards meet the nominal design but some outliers are close to or beyond the specification limits. Managing variation requires design-for-manufacturing (DFM) techniques and statistical verification.

Principal sources of variation and their typical magnitudes:

- Trace width: plus/minus 0.5-1.0 mil (12-25 um) for standard processes, plus/minus 0.3-0.5 mil for premium. A 5 mil trace with plus/minus 1 mil variation shifts impedance by about 10%.
- Dielectric thickness: plus/minus 0.5-1.0 mil. Affects impedance by 5% per mil.
- Dielectric constant: plus/minus 3-5% across the panel and across manufacturers. Affects impedance and propagation velocity.
- Copper weight: plus/minus 10% for 1 oz layers. Affects resistance (PDN IR drop) more than impedance.
- Via drill position: plus/minus 2-3 mil. Affects via symmetry and differential pair matching.
- Via barrel thickness: plus/minus 20% for plating. Affects resistance and thermal conductivity of thermal vias.
- BGA solder ball height after reflow: plus/minus 25-50 um. Affects signal via stub length slightly.
- Component placement: plus/minus 50-100 um. Affects decoupling capacitor parasitic inductance.

DFM techniques:

- Specify tighter tolerances only where needed. Impedance class 6 (plus/minus 10%) is standard; class 7 (plus/minus 5%) is premium. The LPDDR5X DQ layer may need class 7 while other signals run class 6.
- Use impedance test coupons on every panel. The fab measures impedance on the coupon and scraps panels that miss the spec.
- Allow etch compensation in the CAD data. The fab requests 0.5-1 mil wider tracks than drawn to compensate for etch; CAD data is generated with compensation baked in.
- Specify reference-plane copper density to keep the plane consistent and predictable. Isolated copper islands change the impedance of nearby traces.
- Use spread-glass laminate for LPDDR5X to reduce Dk uniformity variation.
- Require IPC-A-600 Class 3 inspection for critical designs (automotive, medical), Class 2 for consumer.

Statistical verification:

- Run Monte Carlo simulation on the PCB channel model with each parameter varied within its tolerance. Compute the eye width at each sample and plot the distribution. The design passes if the 3-sigma point (99.7%) is above the specification.
- Measure a sample of production boards (10-30 units) for impedance and training margin. Verify that the measured distribution matches the simulated distribution.
- At ramp, perform stress testing on a larger sample (100-500 units) spanning multiple lots, to validate the statistical model against reality.

Designs that are borderline at nominal should be reworked before manufacture. Relying on "we might be in spec across variation" is a recipe for yield problems. The cost of a PCB respin is much lower than the cost of a yield excursion at production.

---

### Q6. What protocol-level tests are run on LPDDR PCBs, and what do they catch?

**Answer:**

Protocol-level tests verify that the LPDDR interface operates correctly at the protocol layer: commands are decoded, data is delivered to and from the correct addresses, bank and row operations obey timing, and refresh is honoured. These tests catch issues that are electrical at root but manifest as protocol errors.

Common test categories:

- March tests (march C-, march B, march A): classical memory test patterns that write a sequence to every address and read it back. Catches stuck-at faults, addressing errors, and coupling faults. Fast and simple; typically the first bringup test.
- Pattern tests (walking 1s, walking 0s, checkerboard, alternating): exercise worst-case switching patterns that stress SI. A walking-1s test cycles through each DQ bit being the odd-bit-out while the others hold a background pattern, forcing per-bit crosstalk and per-bit SI margin to be tested individually.
- Linux memtester / mprime / stress-ng: user-space memory stress tools that run continuous allocation, write, read-verify, and free cycles. Typically run for minutes to hours. Catch intermittent errors that may not appear in a single-pass march test.
- Bandwidth stress tests such as stream benchmark or mbwtest: sustain peak memory bandwidth for an extended period, exercising thermal and PDN margin. A design that passes march tests at boot may fail stream under thermal stress.
- Refresh stress tests: disable auto-refresh and observe retention time. Used to verify refresh operation and to characterise retention margin.
- Temperature-sensor cross-check tests: read the DRAM TCSR and verify that refresh rate adapts correctly when the temperature crosses 85C. Catches thermal sensor wiring errors or controller firmware bugs.

Protocol-level tests are typically run on a prototype board with a real Linux (or other OS) and off-the-shelf memory test software. The test harness should expose controller error counters (CRC failures, ECC corrections, retraining events) so that the test can distinguish between different failure modes.

A failed march test usually indicates a gross problem: wrong wiring, stuck DQ, shorted pin, or severe SI failure. A failed stream test that passes march indicates a thermal, PDN, or aging margin problem. A failed stress-ng at high temperature points to refresh or thermal margin. Each failure signature maps to a different PCB fix.

The distinction from SerDes testing is important: LPDDR has no built-in self-test pattern generator and no loopback mode. All testing must run through the real memory controller and exercise real memory. This limits the speed of iteration but also means that passing tests directly prove the memory works end-to-end.

---

### Q7. What are the key EMC / EMI compliance considerations for LPDDRx PCB design?

**Answer:**

LPDDRx interfaces can create significant EMI because they run at GHz data rates with sharp edges and many parallel switching signals. EMC compliance requires the PCB to limit both radiated emissions (fields escaping the board and interfering with other devices) and conducted emissions (noise injected into cables and power supplies).

Radiated emission sources from LPDDR:

- Common-mode currents on data lines. A differential pair with mismatched routing creates common-mode current that radiates efficiently. LPDDR uses single-ended DQ (not differential), so every DQ line is already a common-mode source to some degree. The return current on the ground plane is the dominant radiator.
- Clock harmonics. The CK differential pair and the internal DLL/PLL harmonics extend well above the fundamental. At LPDDR5 3.2 GHz, harmonics reach 10-20 GHz.
- Power plane cavity resonance. The gap between VDDQ and ground planes forms a parallel-plate waveguide that resonates at frequencies determined by the plane dimensions. Energy from SSN excites these resonances and radiates from the board edge.
- PoP stack radiation: the gap between SoC and DRAM in a PoP package is an open cavity that can radiate at millimetre-wave frequencies.

Mitigation techniques for the PCB:

- Solid ground plane directly under and around the LPDDR BGA, extending beyond the package by at least 5 mm. This contains the return current close to the signal path and reduces radiation.
- Ground stitching vias at the board edge and around any plane cuts, at a pitch of lambda/20 at the highest frequency of concern. At 10 GHz this is 1.5 mm pitch.
- Decoupling capacitor placement to suppress plane-cavity resonance by providing a low-impedance current path at the resonant frequency.
- Spread-spectrum clocking on the internal clock generators (if supported by the PHY) spreads the radiated energy across a wider band, reducing peak emission at any one frequency.
- Slow edge rates on non-critical signals (resets, interrupts) to avoid creating unintended emitters.
- EMI shielding cans over the LPDDR BGA area in extreme cases, typically for automotive radar-adjacent PCBs.

Compliance testing uses anechoic chamber measurements per CISPR 22/32 (commercial), FCC Part 15 (US commercial), or CISPR 25 (automotive). The test measures radiated emission versus frequency from 30 MHz to 6 GHz (for Class B commercial) or higher (for automotive). LPDDR-induced emissions typically show up at the data rate fundamentals and their harmonics, sometimes spread across a wider band due to data randomisation.

LPDDR PCBs for radio devices (phones, WiFi routers, automotive radar) need particular attention because the LPDDR harmonics can fall within the radio band and desensitise the receiver. A phone with a 2.4 GHz WiFi radio and an LPDDR4 interface at 2.133 GHz has a fifth harmonic at 10.67 GHz — clear. But the second harmonic at 4.266 GHz is close to the WiFi 5 GHz band and may leak in. Coordination between the RF design team and the memory design team is required.

---

### Q8. How is a prototype LPDDR PCB brought up from first power-on to full operation?

**Answer:**

LPDDR PCB bringup follows a progression from gross functionality to fine margin verification. Each step catches a class of issues that the previous steps do not expose.

Step 1 — Power rail check. Before enabling the LPDDR, verify that all power rails come up to their nominal voltages (VDDQ, VDD2, VDD1, reference rails) with the correct sequencing. Probe the rails with a scope during boot and check for ringing, overshoot, or sequencing errors. Catches PMIC programming errors, decoupling problems, and gross PDN shorts.

Step 2 — ZQ calibration. After power-up, the DRAM performs ZQ calibration on the external reference resistor. If the ZQ resistor value is wrong or the trace is broken, calibration fails and the DRAM reports an error (or the SoC firmware detects it). A simple register read reveals the ZQ result. Catches ZQ trace issues and wrong resistor values.

Step 3 — Mode register read. Attempt to read the DRAM mode registers (MR0, MR1, etc.) at a low data rate (typically 400-533 MT/s, the reset default). If this works, the CA bus and lowest-rate DQ path are functional. If it fails, the CA routing, CK, CKE, or reset is broken.

Step 4 — Training at low rate. Run CA training, write leveling, and DQ training at the low data rate. Training should converge quickly (within seconds). If training fails at low rate, there is a gross SI problem — likely a wrong length, missing decoupling, or a swapped signal.

Step 5 — Low-rate memory test. Run a march test at the low rate. If this passes, the interface is functional end-to-end at low speed. Bandwidth is reduced but the system boots into the OS.

Step 6 — Step up to target rate. Retrain at the target rate (6400, 8533, or higher MT/s). Training must converge. If it fails at target rate but passes at low rate, the SI, PDN, or thermal margin is insufficient at the high rate.

Step 7 — Margin measurement at target rate. Read the training margins (per-bit DQ, VREF, DQS gate) and compare against expected values. Margin below specification indicates PCB issues.

Step 8 — Memory stress test. Run stress-ng or similar for extended periods (1-24 hours) at room temperature. Catches intermittent errors, PDN noise accumulation, and thermal creep.

Step 9 — PVT corner testing. Repeat the tests at hot (85C ambient) and cold (0C or -40C for automotive) temperatures and at voltage corners (nominal plus/minus 5%). This verifies margin across the environment.

Step 10 — EMC testing and system integration. Once LPDDR is stable, test the full system for EMC compliance and verify that LPDDR operation does not break other subsystems (radios, sensors, displays).

Failures at any step point to specific classes of problem. A bringup log that tracks which step each board reached is invaluable for triaging bad boards and for comparing PCB revisions.

---

### Q9. What DFT (Design For Test) features should a PCB provide for LPDDRx?

**Answer:**

Design-for-test features make a PCB testable in production and debuggable during bringup. For LPDDR, DFT is constrained because probing is hard and no protocol test exists, but several features help.

Test points on power rails: 0.5-1 mm pads or through-hole test points on VDDQ, VDD2, VDD1, and ground, ideally close to the BGA. Allows scope and DMM measurement during bringup. Every power rail that can be individually probed should have a test point.

Test points on reference signals: ZQ resistor (to verify the resistor value and trace continuity), VREF (for measurement during training), reset (to verify timing). These are static signals that can be probed without loading the high-speed trace.

JTAG or SWD access to the SoC: allows the debugger to read and write controller registers, including training results and error counters. Essential for any bringup or margin work. The connector should be accessible on the top side of the PCB.

UART for console output: lets early boot firmware print status messages (LPDDR training results, error codes). The bootloader typically prints "LPDDR init OK" or similar, which is the first indication of memory functioning.

Current-measurement resistor in series with VDDQ: a 5-10 mohm precision resistor allows current measurement across the resistor for power analysis and debug. For production, the resistor is removed or replaced with a jumper. For bringup, it stays in.

LED indicators for power-good, training-complete, and error states, driven from SoC GPIOs. Give visual feedback during bringup without needing a debugger.

Debug connector for DRAM protocol analyser: advanced designs include a socket-compatible interposer footprint that lets the DRAM be replaced with a protocol analyser probe. This is invasive and rare but useful for bringup of a new DRAM vendor.

Boundary scan on SoC pins: standard JTAG boundary scan can verify that each SoC LPDDR pin is connected to the expected DRAM pin. Does not verify high-speed function but catches wiring errors, shorts, and opens. Essential if the design has any uncertainty about the BGA wiring.

Margining registers: many modern SoC controllers expose margining registers that let software sweep DQ delay or VREF while the memory is running. Used for automated margin measurement in production.

None of these features help signal integrity or timing directly, but they dramatically reduce the time to diagnose failures. A PCB without DFT typically takes 2-5x longer to bring up than one with full DFT, and production test cost is higher.

---

### Q10. How are design corrections applied when a PCB fails LPDDR testing?

**Answer:**

When a PCB fails LPDDR testing, the corrective action depends on the failure mode and the severity. Minor failures can be fixed with firmware changes; moderate failures may need component changes on the current PCB revision; severe failures require an ECO or full respin.

Firmware-level fixes (no PCB change):

- Training algorithm tuning: adjust training step sizes, starting points, or sequence ordering. Can recover marginal designs that barely miss the passing window.
- VREF manual override: if automatic VREF training converges to a sub-optimal point, a manual override in boot firmware can force a better setting.
- Data rate derating: drop from 6400 MT/s to 5333 MT/s to gain timing and SI margin. Costs bandwidth but ships the product.
- Refresh rate increase: force extended-temperature refresh (double rate) even at normal temperature. Costs bandwidth but improves retention margin.
- Thermal throttling tuning: adjust the temperature thresholds at which the controller throttles. Keeps the DRAM below its failure temperature at the cost of sustained performance.

Component-level fixes (rework the PCB):

- Add or replace decoupling capacitors: if PI analysis reveals insufficient decoupling at a specific frequency, adding small-value caps (100 nF, 1 nF) near the BGA may fix the issue. The PCB must have landing pads prepared.
- Replace a ferrite bead with a zero-ohm jumper: if a ferrite bead creates a resonance that violates PDN target impedance, removing the bead may be necessary. PDN simulation must verify no new issue is introduced.
- Re-terminate ZQ resistor: if the ZQ value is wrong, replace the resistor.
- Add shielding cans: if EMI is the failure, a shielding can over the LPDDR area may fix it without a respin.
- Thermal interface material upgrade: higher-conductivity TIM can lower junction temperature by 5-15C, recovering thermal margin without changing the PCB.

ECO (Engineering Change Order) level fixes (PCB respin):

- Stackup change: if the existing stackup cannot meet impedance or loss targets, a new stackup with different layer ordering or different materials is required. Always a respin.
- Length matching correction: if the matching rules were incorrect in the original design, serpentines are added or removed. Usually a respin unless the change is very small and can be done with via-swapping.
- Via pattern change for thermal or SI: moving vias, adding thermal vias, or changing via type (through-hole to blind) requires a respin.
- Component re-placement: moving decoupling caps closer to the BGA or relocating the PMIC requires a respin.
- Adding a retimer or redriver: LPDDR does not support retimers (there are no retimer products for LPDDR because the interface is too tightly timed). This is a fundamental limit that cannot be fixed in software or by rework.

The root-cause analysis flow is critical: measuring and classifying the failure accurately is more important than applying a fix. A firmware-level workaround applied to a PCB that really has a hardware problem will fail in the field. Every LPDDR failure should be characterised (which signal, which corner, which pattern) before the fix is designed.

---

See also:
- [PCB Stackup Choice](pcb_stackup_choice.md)
- [PCB Signal Integrity](pcb_signal_integrity.md)
- [PCB Timing](pcb_timing.md)
- [Quiz: System](../07_quizzes/quiz_system.md)
