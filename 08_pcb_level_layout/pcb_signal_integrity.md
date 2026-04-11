# PCB Signal Integrity for LPDDRx

This section covers signal integrity (SI) analysis and design techniques for LPDDRx interfaces at the PCB level. PCB SI focuses on the channel from SoC package ball to DRAM package ball, treating the on-die PHY and the package as fixed boundary conditions. The goal is to preserve an open eye at the DRAM receiver after accounting for loss, reflections, crosstalk, jitter, and ISI.

---

### Q1. What is the channel loss budget for an LPDDR5 PCB channel, and how is it allocated?

**Answer:**

The channel loss budget for an LPDDR5 PCB channel is typically 3-5 dB at the Nyquist frequency (3.2 GHz for 6400 MT/s). For LPDDR5X at 8533 MT/s, the budget is 4-6 dB at 4.27 GHz Nyquist. These numbers are smaller than for comparable SerDes interfaces (which often tolerate 20-30 dB) because LPDDR uses single-ended signalling, simple equalisation, and relies heavily on training rather than continuous-time adaptation.

Budget allocation for a typical 25-30 mm LPDDR5 PCB channel:

- PCB trace dielectric loss: 1.5-2.5 dB (dominant at Nyquist, scales with length and Df)
- PCB trace conductor loss: 0.3-0.6 dB (skin effect, scales with sqrt(f))
- BGA fanout and via transitions: 0.3-0.6 dB per end (two ends, so 0.6-1.2 dB total)
- SoC and DRAM package insertion loss: 0.3-0.5 dB per end (0.6-1.0 dB total)
- Reflection-induced loss (return loss better than -15 dB): 0.2-0.4 dB
- Crosstalk-induced eye closure: typically 0.2-0.5 dB equivalent

The budget is not purely additive because some losses are frequency-dependent and some are amplitude-dependent, but the sum gives a good first-order estimate. When the sum approaches the budget, the designer must take action: shorter routes, lower-loss dielectric, back-drilling, or reducing crosstalk sources. For LPDDR5X, exceeding the budget typically requires a stackup change (Megtron 6 to Megtron 7) or an aggressive PHY equalisation scheme (DFE on the DRAM receiver, introduced in LPDDR5X).

---

### Q2. How is insertion loss measured and characterised for a PCB LPDDR channel?

**Answer:**

Insertion loss is the magnitude of the transmission S-parameter (S21 for single-ended, SDD21 for differential) versus frequency. It is measured on a fabricated PCB test coupon or extracted from the 3D field-solver model of the design.

Measurement uses a vector network analyser (VNA) with calibrated probes landed on test coupons that replicate the target channel structure: BGA pad, via stub, signal trace, via transition, second trace, BGA pad. The coupon is embedded in the PCB panel next to the production board so its material and process variation match the production channel. The VNA sweeps from DC to beyond Nyquist (typically 20-30 GHz for LPDDR5X characterisation) and records the S-parameters. A 2x-through de-embedding technique removes the probe fixture contribution.

Extraction from the design uses a 3D full-wave solver (ANSYS HFSS, Cadence Clarity, Simbeor) on a cut-out of the layout containing the via transitions, fanout, and a representative trace segment. The solver produces a Touchstone S-parameter file which is then used in time-domain channel simulation.

Characterisation reports insertion loss at Nyquist, at the second harmonic, and the "knee" frequency where loss exceeds a threshold (typically 3 dB). The insertion loss should be monotonically decreasing (loss increases with frequency). Non-monotonic behaviour indicates resonances from via stubs, fibre-weave effects, or reference-plane discontinuities. Any resonance dip within Nyquist plus 20% is a concern and must be tracked down.

---

### Q3. What causes reflections in the PCB channel, and how are they minimised?

**Answer:**

Reflections occur at impedance discontinuities along the channel. The reflection coefficient at a discontinuity is Gamma equals (Z2 - Z1) divided by (Z2 + Z1). A 10% impedance change produces approximately 5% reflection; a 20% change produces approximately 10% reflection. Reflected energy returns to the source, bounces off the source impedance, and adds to subsequent symbols, producing inter-symbol interference (ISI).

Sources of PCB reflections in an LPDDR channel:

- BGA ball pad capacitance (0.3-0.8 pF depending on pad size and anti-pad). Appears as a local impedance dip of 20-40 ohm at the ball, lasting approximately 20-40 ps.
- Via transitions from signal layer to signal layer. The signal via itself has parasitic capacitance (0.3-1.0 pF) creating a low-impedance point, and the unused via stub acts as an open-ended transmission line that reflects at stub resonance frequencies.
- Trace width changes, for example when a trace necks down to pass between BGA balls or widens to enter a pad. Each width transition reflects proportional to the impedance change.
- Length-matching serpentines. A tight serpentine with trace-to-trace coupling effectively lowers the local impedance.
- Connector transitions (on automotive and industrial boards with socketed DRAM).
- Layer-to-layer transitions where the reference plane changes, creating a return-path discontinuity that looks like an inductive bump.

Minimisation techniques: keep BGA pads as small as the fab allows, optimise anti-pads to tune via impedance (larger anti-pad gives higher impedance), use via-in-pad with back-drilling to eliminate ball-to-via stubs, maintain constant trace width along the route, route serpentines near the receiver (where reflections have less time to cause ISI), and place ground stitching vias near every signal via to provide a low-inductance return path.

---

### Q4. How is crosstalk analysed and budgeted for LPDDR PCB routing?

**Answer:**

Crosstalk is the unwanted coupling of energy from an aggressor signal to a victim signal through mutual capacitance and mutual inductance. On PCBs, two types matter: near-end crosstalk (NEXT, coupling appears at the source end of the victim) and far-end crosstalk (FEXT, coupling appears at the far end). For LPDDR, FEXT dominates on long parallel-routed byte-lane traces while NEXT dominates at dense BGA fanout regions.

The crosstalk budget for LPDDR5 is typically 5-8% of the eye height, allocated as 3-5% from within-byte aggressors (DQ-on-DQ within the same byte lane) and 2-3% from between-byte and CA-on-DQ. At LPDDR5 VDDQ of 0.5V, 5% of a 0.5V swing is 25 mV, which is a significant fraction of the 60-80 mV eye height budget.

Design rules that control crosstalk:

- Minimum trace-to-trace spacing of 3x the trace width ("3W rule") for same-layer same-byte aggressors.
- Increased spacing of 5x trace width between different byte lanes, to let byte lanes be routed independently without one byte's aggressor affecting another.
- Routing orthogonally between adjacent signal layers (layer N traces run in X, layer N+1 in Y) so that coupling between layers is minimised.
- Stripline for the most sensitive signals because the enclosed field drops coupling by roughly 6 dB versus microstrip.
- Physical separation of CA and DQ (on different layers or with a ground guard trace) because a CA error corrupts commands while a DQ error corrupts data and CA is often more bursty.
- Short parallel run lengths: designers often quote a maximum parallel-run of 20-30 mm within a byte and 10-15 mm between bytes.

Crosstalk is simulated by including multiple aggressors and a victim in the 3D extraction, producing a multi-port S-parameter model. Time-domain simulation with worst-case patterns (aggressors all switching, victim held quiet) produces the coupled-in voltage which is subtracted from the eye height.

---

### Q5. What is the effect of via stubs on LPDDRx SI, and how does back-drilling help?

**Answer:**

A via stub is the unused portion of a through-hole via that extends past the layer where the signal actually exits. The stub is electrically an open-ended transmission line, and it resonates at frequencies where the stub length equals an odd multiple of one-quarter wavelength: f_res equals c divided by (4 times L_stub times sqrt(Dk)).

For a 60 mil (1.52 mm) via stub in FR-4 (Dk = 4.2), the first resonance is at roughly 24 GHz, well above LPDDR Nyquist even for LPDDR5X. But the stub does not only affect resonance — it adds a shunt capacitance that reduces the characteristic impedance and slows the effective propagation velocity. A 60 mil stub typically adds 0.3-0.5 pF at the via, producing a local impedance dip to 35-40 ohm. The reflection from this dip creates ISI and directly closes the eye by 5-10%.

As the board gets thicker, the stub gets longer. A 3.2 mm PCB with the routing layer near the top gives a stub approaching 3 mm, with first resonance near 12 GHz — within 2x of LPDDR5X Nyquist — and severely impacting the eye.

Back-drilling removes the unused portion of the via by drilling it out from the opposite side after plating. A typical back-drill leaves 6-10 mils (150-250 um) of residual stub beyond the signal layer, which pushes the first resonance above 100 GHz. The via still has its legitimate through-length contributing parasitic capacitance but the stub reflection is eliminated.

For LPDDR4/4X, back-drilling is a nice-to-have; for LPDDR5, it is recommended on thick boards; for LPDDR5X it is essential. The alternative is to use blind vias or microvias (laser-drilled, connect only adjacent layers, zero stub by construction) in the BGA fanout region and short stack-ups for the trace layers.

---

### Q6. How is eye-diagram analysis performed for an LPDDR PCB channel?

**Answer:**

Eye-diagram analysis is the time-domain visualisation of many unit intervals of data superimposed on a single axis, showing the cumulative effect of loss, ISI, crosstalk and jitter. For LPDDR, eye analysis is performed at the DRAM pin after the full channel — on-die PHY driver, SoC package, PCB channel, DRAM package, and up to the receive slicer.

The simulation flow uses IBIS-AMI models for the driver and receiver (provided by the SoC vendor and DRAM vendor), a channel S-parameter model for the PCB and packages (extracted from the physical design), and a pseudo-random bit sequence (PRBS) stimulus. The simulator convolves the channel impulse response with the stimulus, adds AMI model contributions, and collects the eye.

Key metrics extracted from the eye:

- Eye height at the sampling instant, measured as the vertical opening at the centre of the UI (typically 60-80 mV at the DRAM receiver for LPDDR5).
- Eye width, the horizontal opening at the decision threshold (typically 60-70% of UI; for LPDDR5 with a 156 ps UI, a 60% width means 94 ps of opening).
- Jitter histograms for rising and falling edges, split into random jitter (RJ) and deterministic jitter (DJ) components.
- Bathtub curves, the BER versus sampling position, used to extrapolate to low BER points such as 1e-12 or 1e-16.

For LPDDR, eye analysis is typically performed at two corners: "fast" (fast process, high voltage, low temperature, strong driver) which stresses crosstalk and overshoot, and "slow" (slow process, low voltage, high temperature, weak driver) which stresses loss and edge rates. Both corners must meet the eye specification with margin.

LPDDR5X introduces Decision Feedback Equalisation (DFE) at the receiver, which cancels post-cursor ISI. The eye analysis must be performed with DFE enabled, and the DFE tap values must be optimised as part of the simulation. LPDDR5X also introduces an optional 4-tap FFE at the transmitter.

---

### Q7. What is the role of write leveling and read training in compensating PCB SI imperfections?

**Answer:**

Training is how LPDDR closes the gap between the large absolute uncertainties of a physical channel and the tight timing requirements of high-speed signalling. It compensates for fixed skews (fly-by trace length differences, die-to-die variation, package skew) and semi-fixed parameters (DQ delay relative to DQS, VREF level). Without training, LPDDR5 could not meet its timing at any plausible PCB geometry.

Write leveling aligns DQS with CK at the DRAM, compensating for the flight-time difference between CA/CK and DQ/DQS on the PCB. Fly-by CA routing deliberately places DRAM sites at different CK arrival times; the controller must learn each site's offset and delay DQS accordingly when writing. Write leveling runs at initialisation: the controller sends DQS pulses and reads back the CK state that DQS captured; it adjusts DQS phase until CK just transitions to 1.

Read training discovers the correct DQS delay and strobe gate timing for reads. Because the DRAM is the DQS source for reads, the controller has no prior knowledge of when DQS will arrive. Read training uses a fixed DRAM-generated pattern, sweeps the DQS capture delay, finds the passing window, and sets the delay at the window centre.

Read DQ training (per-bit deskew) aligns each DQ bit to DQS. Intra-byte PCB length differences up to plus/minus 1-2 mm (equivalent to plus/minus 10-15 ps) are removed by per-bit delay lines in the PHY. This relaxes PCB length matching from the sub-millimetre tolerance that would otherwise be required.

VREF training finds the optimal receiver threshold level by sweeping VREF and measuring the passing eye height at each setting. This compensates for static offsets caused by VDDQ IR drop, driver mismatch, and PCB loss-induced DC wander.

CA training (LPDDR5 introduced CBT, CA bus training) aligns CA bits to CK. LPDDR5X has enhanced training with on-die storage of per-DQ/per-CA delay settings.

The SI consequence of training is that the PCB designer does not need to meet sub-picosecond absolute skew targets — training handles the tens of picoseconds of PCB-introduced skew. The PCB designer does need to meet the jitter, loss, crosstalk and reflection budgets, because training cannot compensate those (they vary per-symbol and per-aggressor).

---

### Q8. How does fibre-weave effect impact LPDDR5X differential pairs, and how is it mitigated?

**Answer:**

Fibre-weave effect is the Dk variation along a trace caused by the periodic pattern of glass fibre bundles and resin-filled gaps in the PCB dielectric. A trace routed along a glass bundle sees higher Dk (slower propagation) than a trace routed along a resin gap (lower Dk, faster propagation). The two halves of a differential pair routed on a coarse weave may therefore see different Dk, producing intra-pair skew that degrades the common-mode rejection and creates mode conversion.

For LPDDR5X at 8533 MT/s (UI = 117 ps), the differential skew budget for DQS, WCK and CK is approximately 1-2 ps. A 25 mm trace pair on a coarse 106 weave (resin-rich gaps roughly 0.1 mm wide, Dk variation of plus/minus 0.15) can accumulate 3-5 ps of differential skew purely from weave effect, exceeding the budget.

Mitigation techniques:

- Use fine-weave or spread-glass styles (3313, 2116, 1078 with mechanically spread glass) which reduce the Dk variation to plus/minus 0.02-0.05.
- Rotate the PCB panel 10-15 degrees from the weave axis so that a straight trace cuts across multiple fibre bundles, averaging out the variation. This is sometimes called "zig routing" or "weave angling". Some fabs offer this as a standard option.
- Add small routing jogs (less than 50 um) at regular intervals to force the trace to see different weave positions. This is effective but adds routing complexity.
- Use low-weave laminates such as Megtron 7 or Megtron 8 which are specified with tight Dk uniformity.
- For differential pairs, keep the pair tightly coupled (spacing less than 2x trace width) so both halves see the same bundle on average; loosely coupled pairs are more sensitive to weave position.

Fibre-weave effect was first recognised as a serious issue for 10G-20G SerDes; LPDDR5X is the first LPDDR generation fast enough to care. LPDDR4/5 at under 7 GHz Nyquist is essentially immune.

---

### Q9. What is the role of simultaneous switching output (SSO) noise in LPDDR PCB SI?

**Answer:**

Simultaneous switching output noise, also called SSO or delta-I noise, arises when many drivers transition simultaneously and pull a large transient current through the shared PDN inductance. The resulting voltage drop on VDDQ and bounce on VSS is coupled into every other driver and receiver sharing the same supply. This causes a data-dependent shift in the received signal that looks like deterministic jitter and amplitude noise.

For LPDDR, an x8 byte lane with 8 DQ plus DQS plus DQS_n all switching from 0 to 1 or 1 to 0 can pull several hundred mA of transient current through the package bump and PCB via inductance within 50-100 ps. If the loop inductance from DRAM die to decoupling capacitor is 500 pH, a 300 mA transient produces V equals L times dI/dt equals 500 pH times 300 mA divided by 100 ps equals 1.5V — which is larger than VDDQ itself, clearly untenable. In practice on-die decoupling and package capacitance absorb most of this transient, but 20-50 mV of residual VDDQ bounce is typical and contributes directly to the jitter budget.

PCB-level techniques to reduce SSO impact:

- Place bulk and mid-frequency decoupling capacitors as close to the SoC VDDQ BGA balls as possible, minimising the ESL loop inductance.
- Use many small capacitors in parallel rather than one large capacitor, reducing the effective ESL.
- Maintain a solid VDDQ plane close to an adjacent ground plane with minimal dielectric (embedded capacitance approach) for a distributed decoupling effect.
- Provide multiple VDDQ and VSS BGA balls per byte lane and fan them out with short wide traces or plane connections.
- On the PCB, locate DQ and VDDQ vias close together so their return currents share the same ground vias.

SSO is primarily a PDN and package problem, but PCB layout decisions (decoupling placement, via pattern, plane proximity) directly affect the residual bounce seen by the DRAM. The SI engineer and PI engineer must collaborate — SSO cannot be treated as purely one or the other.

---

### Q10. How does the PCB SI analysis integrate with on-die PHY and package models?

**Answer:**

A full channel analysis for LPDDR combines three separately-developed models: the on-die PHY IBIS-AMI model, the package electrical model, and the PCB electrical model. Each is developed by a different team with different tools, and the integration happens at the system-level SI simulator.

The on-die PHY model is typically delivered as an IBIS-AMI package. The IBIS portion describes the analogue driver and receiver (output impedance, edge rate, capacitance, VREF levels) while the AMI portion describes the equalisation algorithms (DFE taps, FFE coefficients, VREF adaptation). The AMI model is called iteratively by the simulator to adapt the equaliser to the channel.

The package model is an S-parameter block extracted from the package layout (substrate traces, bumps, bondwires or flip-chip, solder balls). It is typically delivered as a Touchstone file with dimensions matching the number of signals in a byte lane (20 ports for x8: 8 DQ, DM, DQS, DQS_n, 8 victims, 4 aggressors as an example). The package model includes the package power distribution impedance, which couples into the signal channel through SSO.

The PCB model is also an S-parameter block, covering the BGA fanout, via transitions, trace routing, and the far-end BGA fanout. It is extracted by a 3D solver on a representative section of the layout. For a full byte lane simulation, the PCB model is 16-20 ports.

The three models are chained together in the channel simulator: AMI driver -> SoC package -> PCB -> DRAM package -> AMI receiver. The simulator applies a test pattern, runs the channel, and collects the eye at the AMI receiver sampling point. Results are compared against the LPDDR specification eye mask or a derived system eye mask.

Common integration pitfalls: port-numbering mismatch between package and PCB models (each model assumes a different port ordering for DQ bits); impedance discontinuity at the boundary between models if they use different reference impedances; missing power-domain ports (if the PCB or package model does not include VDDQ, the SSO effects are under-estimated); and differing frequency ranges or DC points (if one model stops at 10 GHz while another goes to 40 GHz, the simulator must handle the mismatch).

---

See also:
- [PCB Stackup Choice](pcb_stackup_choice.md)
- [PCB Power Integrity](pcb_power_integrity.md)
- [PCB Timing](pcb_timing.md)
- [SI for LPDDR (on-die)](../05_signal_and_power_integrity/si_for_lpddr.md)
