# PCB Timing for LPDDRx

This section covers timing analysis for LPDDRx interfaces at the PCB level. PCB timing is concerned with flight times, skew across matched groups, jitter accumulation, setup and hold margins at the DRAM receiver, and the way PCB physical layout interacts with the training-based LPDDR timing model. The PCB adds tens to hundreds of picoseconds of delay and skew to a signalling system whose unit interval at LPDDR5X is only 117 ps, so PCB timing choices directly determine whether the link closes.

---

### Q1. What is the unit interval (UI) for each LPDDRx generation, and what does this mean for PCB timing budgets?

**Answer:**

The unit interval is the duration of one data bit, equal to the reciprocal of the data rate.

- LPDDR4 at 3200 MT/s: UI equals 312.5 ps
- LPDDR4X at 4266 MT/s: UI equals 234 ps
- LPDDR5 at 6400 MT/s: UI equals 156 ps
- LPDDR5X at 8533 MT/s: UI equals 117 ps
- LPDDR5X at 9600 MT/s: UI equals 104 ps
- LPDDR6 at up to 14400 MT/s (under development): UI equals 69 ps

The setup and hold window at the receiver is a fraction of the UI, typically 30-40% for LPDDR, which leaves 70-60% of UI for total accumulated jitter, ISI, crosstalk and skew. For LPDDR5X at 117 ps UI, a 35% setup+hold window means the receiver requires 41 ps of clean eye width, and 76 ps of UI is available for all impairments combined. Every picosecond of PCB-introduced jitter or skew consumes part of this budget.

PCB propagation velocity is approximately 6-7 ps/mm on stripline (Dk = 4.2). Every 1 mm of trace uses 6-7 ps of the budget. A 25 mm trace contributes 150-175 ps of flight time, which must be length-matched against other signals in the same group to within typically 1-3 ps — i.e., 0.15-0.5 mm physical length matching.

The key insight: as data rates rise, the PCB length matching tolerance scales inversely. LPDDR4 at 312 ps UI can tolerate 5-10 mm of length mismatch (and uses training to absorb it); LPDDR5X at 117 ps UI tolerates 0.5-2 mm after training support.

---

### Q2. What are the intra-byte, inter-byte, and CA-to-CK skew budgets for LPDDR5 on PCB?

**Answer:**

LPDDR uses three distinct matching groups with different tolerances, corresponding to different phases of the training flow.

Intra-byte skew: the skew between the 8 DQ bits of a byte lane and the DQS differential pair that strobes that byte. Per-bit deskew delay lines at the PHY receiver can shift each DQ bit by up to 10-20 ps relative to DQS, so the PCB-delivered intra-byte skew should not exceed 5-10 ps (half the delay-line range, to leave training margin). At 6-7 ps/mm this means matching DQ to DQS within 0.7-1.5 mm of physical length. Data mask (DM) and data bus inversion (DBI) signals are also part of the intra-byte group.

Inter-byte skew: the skew between different byte lanes (byte 0 versus byte 1 of a x16 channel). Byte lanes are independently trained — each byte has its own DQS and its own per-bit delay lines — so inter-byte matching is relaxed. Typical PCB matching is plus/minus 5-10 mm, corresponding to plus/minus 30-70 ps, which is absorbed by the per-byte timing adjustment during training. The only hard constraint is that byte-to-byte skew must not exceed the controller's inter-byte alignment range, typically 1-2 UI.

CA-to-CK skew: the Command/Address bus runs at half the DQ rate in LPDDR4/5 (so 3200 MT/s for LPDDR5 at 6400 MT/s) and one-quarter the DQ rate in LPDDR5X (so 2133 MT/s for LPDDR5X at 8533 MT/s). CA is launched with CK as its strobe. The CA-to-CK matching must be tight enough for the DRAM to sample CA on the correct edge: the LPDDR5 CA sampling window is approximately 40% of CA UI, leaving 60% for impairments. A 312 ps CA UI gives 187 ps of budget; 10 ps of PCB skew is comfortable. CA training (CBT, CA Bus Training) provides per-bit CA deskew at initialisation, so PCB matching of plus/minus 1-2 mm (6-14 ps) is sufficient.

CK differential pair matching: CK to CK_n must be matched to approximately 1-2 ps (0.15-0.3 mm) to preserve CK duty cycle, because any intra-pair skew translates directly to duty-cycle distortion. Similarly, DQS to DQS_n and WCK to WCK_n. This is tighter than any other budget on the PCB and requires careful trace-pair routing.

---

### Q3. How is length matching performed on PCB for LPDDR, and what techniques ensure tight matching?

**Answer:**

Length matching on PCB uses serpentine (meander) routing to add length to shorter traces within a matched group. The layout CAD tool (Cadence Allegro, Mentor Xpedition, Altium Designer, KiCad) enforces matching constraints automatically by computing a path length for each trace and adjusting the serpentine amount to equalise them.

The matching constraint is expressed as a target length and a tolerance. For example, "match all DQ0-DQ7 and DQS/DQS_n to within plus/minus 0.15 mm of DQS". The tool computes the longest trace in the group, then adds serpentines to the others until they reach the same length. The designer sets the serpentine parameters: minimum amplitude (3x trace width or greater to avoid self-coupling), segment pitch, and which zones permit serpentine.

Best practices:

- Place serpentines near the receiver rather than the driver. Reflections from serpentine impedance discontinuities have less time to cause ISI if they happen late in the trace.
- Use rounded or chamfered corners, not 90-degree corners. Each 90-degree corner adds capacitance and discontinuity equivalent to approximately 0.2 pF, enough to affect high-frequency matching.
- Maintain 3W (3x trace width) spacing inside the serpentine to avoid self-coupling between adjacent serpentine segments, which would lower the effective impedance and change the propagation velocity.
- Avoid serpentining differential pairs individually — serpentine both halves together as a pair, so the intra-pair matching is preserved.
- Keep the serpentine out of the via-heavy fanout region; the fanout already introduces delay and managing both together is error-prone.

For very tight matching on LPDDR5X, the tool may need to match at the level of individual trace segments, accounting for different layer propagation velocities (microstrip versus stripline) and via delay contributions. This is called "electrical length matching" or "delay-based matching" rather than physical-length matching and requires the tool to know the propagation constant of each layer.

Absolute length limits also apply: no DQ trace should exceed the channel loss budget, which for LPDDR5X on Megtron 6 limits trace length to approximately 35-40 mm between SoC and DRAM for non-PoP designs.

---

### Q4. What is flight-time skew, and how does it differ from physical length skew?

**Answer:**

Physical length skew is the geometric length difference between two traces, measured in millimetres. Flight-time skew is the propagation delay difference between two traces, measured in picoseconds. They are related by the propagation velocity, but physical length matching alone does not guarantee flight-time matching when traces pass through different environments.

Sources of flight-time skew beyond physical length:

- Layer changes. A DQ bit routed partly on a microstrip top layer (5.5 ps/mm) and partly on a stripline inner layer (7 ps/mm) has a different effective delay per mm. Two equal-length traces with different layer distributions have different flight times.
- Via transitions. Each signal via adds 5-20 ps of delay depending on its length, pad size, and stub. If one bit has two vias and another has three, the delays differ.
- Fibre weave variation. Two traces routed in different locations on a coarse-weave board see different local Dk and therefore different propagation velocity. This shows up as random skew within a group, typically up to 1-3 ps per 25 mm of trace.
- Package skew. The SoC package internal routing and the DRAM package internal routing add per-bit delays that vary by tens of picoseconds. The PCB designer usually receives a package delay table from the vendors and must compensate at the PCB layer.
- Trace length versus effective electrical length. A serpentine adds physical length but the signal propagates partly through the serpentine coupling, effectively slightly shortening the delay compared to a straight trace of the same total length.

Best practice is to use delay-based matching (flight-time matching) rather than physical-length matching for LPDDR5 and LPDDR5X. The layout tool computes the propagation delay for each trace segment based on the layer velocity, sums the via delays from a via delay model, and equalises the total delay. Some tools also support fibre-weave de-skew by applying a random perturbation to expected delay.

For LPDDR4/4X, physical-length matching is usually acceptable because the UI is large enough to absorb the flight-time discrepancies.

---

### Q5. How do package delays interact with PCB length matching?

**Answer:**

Package delay is the propagation delay from the SoC die pad through the package substrate to the BGA ball, and similarly from the DRAM die to its BGA ball. Package delays typically range from 10-50 ps per ball, with 5-20 ps of per-ball variation across a byte lane. This variation appears as a per-bit skew that the PCB must compensate, or that training must absorb.

For PoP designs, the SoC-to-DRAM routing is entirely within the package stack (no PCB at all for LPDDR signals). The package team controls all routing and delivers a well-matched byte lane — typically within plus/minus 2-3 ps — at the ball level. The PCB timing problem is limited to power delivery.

For non-PoP designs, the PCB receives signals at the SoC BGA with some per-bit skew baked in from the SoC package, and the DRAM BGA similarly has per-bit skew in its package. The SoC vendor provides a package delay table: for each DQ ball, the physical delay from die to ball. The PCB designer adds this to the PCB trace delay and matches the total from die to die.

Example: if SoC DQ0 has 18 ps package delay and SoC DQ1 has 22 ps package delay, the PCB designer routes DQ0 4 ps longer than DQ1 to compensate. At 6 ps/mm stripline, this is a 0.67 mm length adjustment. The matching is done in the CAD tool by specifying per-bit offsets in the constraint.

The DRAM package introduces the same type of per-bit offset at the far end. For well-characterised DRAM, the vendor provides a similar table. If the DRAM vendor does not publish per-bit package skew, the designer treats it as a random variable absorbed by DQ deskew training.

Missing package delay compensation is a common cause of LPDDR timing marginality. The PCB designer who only matches PCB trace lengths and ignores package skew may fit a 5-10 ps per-bit error in, which consumes most of the DQ deskew training range and leaves no margin for voltage and temperature variation.

---

### Q6. What is tDQSCK, and how does it affect PCB timing analysis for reads?

**Answer:**

tDQSCK is the LPDDR DRAM read strobe skew: the uncertainty in the time from the read command arriving at the DRAM to the DRAM sending DQS back. It includes the internal DRAM pipeline delay, process variation, supply variation, and temperature variation. For LPDDR5, tDQSCK is specified in the JEDEC standard at approximately plus/minus 400 ps, which is over 2 full UIs at LPDDR5 rates.

This large tDQSCK means the controller cannot predict the DQS arrival time at the SoC with picosecond precision. Instead, the controller uses a DQS gate — a window during which it expects DQS to arrive — and the gate is trained at initialisation. The DQS gate must be wide enough to accommodate the tDQSCK range plus the PCB flight-time uncertainty.

The impact on PCB timing analysis is that the absolute flight time from SoC to DRAM does not directly affect read timing closure, because training compensates for the round-trip delay. What matters is the stability of the flight time: temperature-induced changes, voltage-induced changes, and aging. If the PCB trace length changes with temperature (CTE effects on Dk and physical length), the DQS arrival time shifts by a few picoseconds across the operating range, and the DQS gate must be wide enough to tolerate this. For LPDDR5X at 117 ps UI, a few picoseconds of drift is a significant fraction of the DQS gate.

The PCB designer's role in read timing is therefore: (1) keep the flight time stable across PVT (pick stackup materials with low Dk temperature coefficient), (2) meet the DQS-to-DQ skew budget for read (same as write, since DQS strobes DQ), and (3) provide stable VDDQ PDN to avoid VDDQ-induced driver delay modulation on the DRAM side.

The SoC-side read training happens once per boot (and periodically for long operation), so slow drifts are compensated. Fast drifts within a single burst (microsecond scale) cannot be trained out and must be engineered out through stable PDN and thermal design.

---

### Q7. How does jitter accumulate in an LPDDR PCB channel, and what is the PCB's contribution?

**Answer:**

Jitter is the deviation of signal edges from their ideal positions in time. It accumulates from multiple sources along the channel and is usually decomposed into random jitter (RJ, thermal-noise-driven, Gaussian) and deterministic jitter (DJ, pattern- or structure-driven, bounded).

PCB contributions to jitter:

- Dielectric loss creates pattern-dependent jitter: lossy channels slow rising edges for patterns with long run-lengths (the signal doesn't reach full swing), shifting the zero-crossing time. This is data-dependent jitter (DDJ), a form of ISI.
- Crosstalk from aggressors creates aggressor-dependent jitter. Each switching aggressor perturbs the victim edge by a few picoseconds. This is bounded but uncorrelated to the victim pattern, so it appears as additional deterministic jitter.
- Reflections from impedance discontinuities create echo jitter. A reflection from a via that returns after a full UI causes the next symbol edge to shift depending on the previous symbol. Again a form of DDJ.
- PDN noise creates supply-induced jitter. VDDQ ripple modulates the driver strength, shifting the edge timing. At the receiver, VDDQ ripple modulates the VREF and threshold, shifting the crossing detection.
- Fibre-weave skew creates per-bit random jitter between supposedly identical traces.

Typical PCB jitter contribution to an LPDDR5 channel: 3-6 ps RMS RJ (mostly from PDN noise and random crosstalk) and 8-15 ps pk-pk DJ (from ISI and aggressor crosstalk). The on-die PHY adds another 2-4 ps RJ and 3-8 ps DJ (clock PLL/DLL jitter, driver noise). The DRAM receiver adds 2-4 ps RJ.

Total jitter at BER 1e-12 is computed as 14.1 times RJ (for Gaussian RJ and double-sided BER target) plus the peak-to-peak DJ. For 6 ps RJ and 15 ps DJ: total jitter equals 14.1 times 6 plus 15 equals 85 plus 15 equals 100 ps. Against a 156 ps UI, this is 65% of UI — unacceptable. The actual design must reduce these numbers through shorter traces, lower-loss dielectric, and tighter PDN.

This is a key reason why LPDDR5X is hard to close on PCB: the jitter budgets at 117 ps UI are extremely tight. Many LPDDR5X designs use PoP or package-level routing specifically to avoid the PCB jitter contribution.

---

### Q8. How does the PCB timing model feed into LPDDR training algorithms?

**Answer:**

LPDDR relies on training to absorb static and slowly-varying timing uncertainties that would otherwise make closure impossible. The PCB designer does not need to meet the final receiver timing — training does that — but must deliver signals within the trainable range of each algorithm.

Training phases and their PCB timing implications:

- CA Training (CBT): trains per-CA-bit delay to CK. Trainable range is typically plus/minus 40-100 ps relative to CK. PCB skew in the CA group must fit within this range. For LPDDR5X, CBT is enhanced with finer step sizes and wider range.
- Write Leveling: aligns DQS to CK at the DRAM. Trainable range covers one full UI plus margin. Absolute CK-to-DQS PCB flight time is irrelevant; only the relative CK phase at the DRAM matters.
- DQ Write Training: trains per-bit write DQ delay to DQS at the DRAM. Trainable range is typically plus/minus 1/2 UI. PCB intra-byte skew must fit within this.
- DQ Read Training: trains per-bit read DQ delay, and trains the DQS gate timing. Similar range.
- VREF Training: trains DRAM and SoC receiver VREF level. Adjusts for offsets caused by PDN IR drop, loss-induced DC wander, and driver mismatch. Trainable range is typically plus/minus 10-15% of VDDQ.

The PCB designer's timing budget must fit inside the union of all these ranges, with margin. Typical PCB intra-byte skew must stay under plus/minus 10-15 ps (worst case over PVT) to leave training range for on-die variation and aging.

After training, the system operates with a fixed set of per-bit delays and VREF settings. Periodic re-training (during refresh pauses or when temperature crosses thresholds) adapts to slow drifts. Fast drifts (less than 1 ms) cannot be re-trained and must be engineered out.

A common design error is to treat training as a free budget: "we have 1 UI of range, so any PCB skew under that is fine". In reality the training range is shared with process variation (plus/minus 3-5 ps), voltage variation (plus/minus 2-4 ps), temperature variation (plus/minus 5-15 ps), and aging. The PCB should consume only about a third of the training range, leaving the rest for other sources.

---

### Q9. What is the role of the CK tree topology in CA timing, and how does it differ from on-die?

**Answer:**

The CK (command clock) signal must arrive at all DRAM CA pins simultaneously, or with a known offset that training can compensate. The CK routing topology on PCB determines how well this is achieved.

For a single DRAM per channel (the common PoP case), CK is a point-to-point pair from SoC to DRAM. The topology is trivial and timing is purely a matter of matched-pair routing.

For multi-DRAM configurations (non-PoP with two DRAM packages per channel, common in some automotive and HPC designs), CK must fan out to both DRAMs. The options are:

- T-topology (symmetric branch): CK routes to a central tee point, then branches to each DRAM with matched legs. This gives identical delay to both DRAMs but creates impedance discontinuity at the tee and reflections from both unterminated branches back towards the source. The reflections add ISI that degrades the CK duty cycle and edge rate.
- Fly-by topology: CK routes to DRAM A first, then continues to DRAM B. DRAM A sees CK earlier than DRAM B by the flight time between them. The fly-by topology has no branch point and no reflections from a tee, but introduces deliberate skew that must be compensated. Write leveling provides per-DRAM DQS delay to absorb this skew.

LPDDR uses fly-by routing for CA and CK in multi-drop configurations, relying on write leveling to compensate the per-DRAM delay. The PCB layout must route CA and CK along a single path that visits each DRAM in sequence, with termination at the far end. Termination for LPDDR CK is typically on-die, so no explicit far-end resistor is needed on the PCB.

The on-die CK distribution is different: on-die CK is an H-tree or mesh with matched delays to all IOs, and uses a DLL for local phase alignment. The PCB CK routing is more constrained by geometry and cannot use the same techniques.

For the PCB designer, the key decisions are: where to place the DRAMs relative to the CK routing (to keep the fly-by length manageable), what intra-pair matching to specify for CK/CK_n, and how to terminate CK if on-die termination is not sufficient.

---

### Q10. How is PCB timing closure verified before tape-out or manufacture?

**Answer:**

Timing closure verification combines static analysis (constraint-based, no simulation), channel simulation, and lab measurement on prototypes.

Static analysis uses the PCB layout tool's length matching report. The tool checks every constraint: is each DQ within the specified tolerance of DQS; are CK and CK_n matched; is the total byte length within the absolute limit. The output is a pass/fail report with the failing nets highlighted. This is the first line of defence and catches gross errors (unrouted serpentines, missed constraints). It runs as part of every ECO and is required to pass before fabrication.

Delay-based analysis (a refinement of length matching) uses per-layer propagation velocities and per-via delay models to compute actual flight time rather than physical length. This catches layer-mix issues that length-based matching would miss. Tools include Cadence Allegro Physical Viewer, Mentor HyperLynx Constraint Editor, and Altium PCB Inspector.

Channel simulation produces the final timing verdict by computing the eye diagram at the receiver and extracting the eye width. The channel model includes S-parameters for the extracted PCB, package models for SoC and DRAM, and IBIS-AMI models for PHY and DRAM. The output is an eye width in picoseconds. It must exceed the receiver's setup-plus-hold window with margin. A typical target is eye width greater than 40% of UI for LPDDR5, 35% for LPDDR5X.

Lab measurement on prototype boards is the final verification step. Techniques include:

- TDR measurement on test coupons to verify impedance control.
- Direct probing of a DRAM data signal with a high-bandwidth probe (though this is difficult for BGA-mounted DRAMs; often requires a specialised interposer).
- Training margin measurement via SoC debug registers. Most LPDDR controllers expose the training results (per-bit delay settings, VREF settings, DQS gate settings) and the eye margin (how far the settings can be moved before errors appear). This is the most accurate measure of real-world timing closure.
- BER sweep by running a memory test (mbwtest, stress-ng, Linux memtester) while varying VDDQ and temperature at the system level. A robust design should pass BER less than 1e-12 across the full PVT range.
- Shmoo plots of error rate versus a swept parameter (voltage, DQS delay, VREF). These reveal the timing margin at each corner.

A design that passes static analysis, simulation, and lab measurement across PVT is considered timing-closed.

---

See also:
- [PCB Stackup Choice](pcb_stackup_choice.md)
- [PCB Signal Integrity](pcb_signal_integrity.md)
- [PCB Thermal](pcb_thermal.md)
- [Length Matching and Skew (on-die)](../04_routing_and_matching/length_matching_and_skew.md)
- [LPDDR Signaling and Timing](../01_foundations/lpddr_signaling_and_timing.md)
