# PCB Stackup Choice for LPDDRx

This section covers PCB stackup design decisions for LPDDRx interfaces. The stackup is the foundational constraint that determines achievable impedance control, insertion loss, crosstalk, routing density, and PDN impedance. Stackup decisions made at schematic-capture time propagate through every downstream SI, PI, timing and thermal analysis and are expensive to change later.

---

### Q1. What is the minimum layer count required for a PCB routing an LPDDR5/5X interface, and what drives it?

**Answer:**

The minimum layer count for a PCB routing a single x32 (two-channel) LPDDR5 or LPDDR5X interface in a non-PoP configuration is typically 8 layers, and 10-12 layers are common when other high-speed interfaces (PCIe, MIPI, USB) share the board. The layer count is driven by the BGA fanout requirement, the need for adjacent reference planes on every signal layer, the power distribution requirement, and the isolation requirement between different high-speed domains.

BGA fanout is usually the dominant factor. For a 0.5-0.65 mm ball-pitch SoC with two LPDDR channels (approximately 160-200 signal balls including DQ, DQS, CA, CK, ZQ and associated ground), escaping all signals to the package perimeter requires roughly 3-4 signal layers when combined with via-in-pad or dogbone fanout. Each of those signal layers must have an adjacent ground plane for a clean return path, doubling the layer count to 6-8 layers for signalling alone. Add a dedicated VDDQ power plane, a VDD2/VDD1 plane and a plane-pair for non-memory supplies and the minimum rises to 8-10 layers. For LPDDR5X at 9600 MT/s, an additional stripline routing layer is often added to enable back-drilling headroom and to keep the most critical signals (DQS, WCK) on a low-loss stripline, pushing the total to 10-12 layers.

Cost scales super-linearly with layer count (each additional layer pair adds lamination, drilling, and yield loss), so the layer count is a key cost driver and must be minimised through careful PHY bump map design, channel orientation relative to the BGA edge, and aggressive use of via-in-pad. Mobile PoP designs avoid this problem entirely by routing LPDDR in the package stack, leaving only 4-6 layer mainboards.

---

### Q2. How are signal layers allocated to LPDDRx signal classes in a typical stackup?

**Answer:**

Signal layer allocation for LPDDRx balances routing length, loss budget, crosstalk sensitivity and back-drill feasibility. The common allocation strategy groups signals by their criticality and routing constraints.

DQ byte lanes are typically routed on a stripline layer adjacent to a solid ground plane. Stripline gives better shielding, lower crosstalk from surface signals, and more predictable impedance than microstrip. The trade-off is higher dielectric loss (because both sides see dielectric rather than air) and roughly 20-30% slower propagation velocity (approximately 7 ps/mm versus 5.5 ps/mm for microstrip). For LPDDR4/4X at 4266 MT/s, microstrip on an outer layer is acceptable; for LPDDR5 and beyond, stripline is preferred for DQ.

DQS and WCK differential pairs are always routed on the same layer as the DQ byte they strobe. Routing strobe and data on different layers would introduce a per-via-pair skew that is difficult to length-match and would compromise the intra-byte skew budget. Co-layer routing also equalises temperature and Dk-variation effects between data and strobe.

CA and CK are routed on a separate stripline layer, often one plane-pair deeper than DQ. CA has much lower bandwidth requirements than DQ (the CA rate is half the DQ rate in LPDDR4/5, one-quarter in LPDDR5X), so CA loss budget is relaxed. However, CA is a fly-by or tree topology driving multiple DRAM sites and is highly crosstalk-sensitive because any CA bit error produces a command failure; isolating CA on its own layer away from DQ reduces aggressor exposure.

Reference and auxiliary signals (ZQ, RESET_n, ODT, CKE) are routed on any convenient layer with no special constraints beyond impedance control.

---

### Q3. What dielectric material choices are available, and how do they map to LPDDR generation?

**Answer:**

PCB dielectric materials vary by loss tangent (Df), dielectric constant (Dk), glass-weave style, thermal stability, and cost. The choice is driven primarily by the loss budget at the Nyquist frequency of the target LPDDR generation.

Standard FR-4 (Df approximately 0.02 at 1 GHz, Dk approximately 4.2-4.5) is inexpensive and adequate for LPDDR4 at 3200-4266 MT/s on short PCB routes (less than 30 mm). Insertion loss at the 1.6-2.1 GHz Nyquist is approximately 0.2-0.3 dB/cm, so a 30 mm trace loses 0.6-0.9 dB, which fits within a typical 3-4 dB channel budget.

Mid-loss materials such as Isola 370HR, Panasonic Megtron 4, or Nelco 4000-13 (Df approximately 0.010-0.014) are the standard choice for LPDDR4X and LPDDR5 up to 6400 MT/s. These materials give 30-50% lower loss at the same frequency without a large cost premium.

Low-loss materials such as Megtron 6 (Df approximately 0.004), Megtron 7, or Rogers RO4350B are required for LPDDR5X at 8533-9600 MT/s on PCB trace lengths above 25-30 mm. The Nyquist frequency rises to 4.3-4.8 GHz where every additional 0.005 Df costs roughly 0.15 dB/cm.

Ultra-low-loss materials (Megtron 8, Tachyon 100G, Df less than 0.003) are currently over-specified for LPDDR but are common on server boards that share the stackup with 56G or 112G SerDes.

The glass-weave style matters as much as the bulk Df for LPDDR5X. A coarse 106 or 1080 weave creates periodic Dk variation along the trace, producing a fibre-weave skew effect on differential pairs (DQS, WCK). Spread-glass or mechanically spread 3313/2116 weaves, or rotating the PCB 10-15 degrees from the weave axis, mitigates this.

---

### Q4. How is target impedance chosen for LPDDRx, and why is it commonly 40 or 50 ohm?

**Answer:**

The target characteristic impedance for LPDDRx single-ended signals is typically 40 ohm for LPDDR4X/5 and 40 or 50 ohm for earlier generations, with the corresponding differential impedance at 80 or 100 ohm. The choice balances driver capability, power, crosstalk, and routing density.

LPDDR drivers are push-pull CMOS with programmable output impedance (typically 34, 40, 48 or 60 ohm). The driver impedance is matched to the trace impedance so that the driver presents a matched termination to back-reflections from the far end; this is "source-series termination" (SST) topology. Matching driver impedance to trace impedance also maximises the voltage delivered to the far end for a given supply voltage (V_far equals V_supply/2 for a matched 1:1 divider). Lower-impedance traces (40 ohm) allow a lower driver impedance, which gives faster edge rates and a stronger signal for the same driver size but increases the current required and therefore power and SSO noise. Higher-impedance traces (50 ohm) reduce the current draw but require larger drivers for the same edge rate.

LPDDR4X and LPDDR5 moved to 40 ohm to improve the edge rates at 0.6V and 0.5V VDDQ respectively. The lower supply voltage reduces the available signal swing, so the design trades power for edge rate by using a lower target impedance. LPDDR5X continues the 40 ohm target.

For PCB layout, lower trace impedance means wider traces for the same stackup. A 40 ohm microstrip is roughly 30% wider than a 50 ohm microstrip at the same dielectric thickness, consuming more routing resource. Designers often compensate by using thinner signal-to-reference dielectric (50-75 um) to keep traces narrow while holding 40 ohm.

---

### Q5. What are the trade-offs between microstrip and stripline for LPDDR routing?

**Answer:**

Microstrip is a trace on an outer layer with a single reference plane below (dielectric on one side, air on the other). Stripline is an inner-layer trace with reference planes both above and below. Each has distinct electrical characteristics that influence LPDDR performance.

Microstrip advantages: lower dielectric loss because half the field lines travel through air (air Df is effectively zero), faster propagation velocity (approximately 5.5-6.5 ps/mm versus approximately 7-8 ps/mm for stripline at Dk=4.2) which slightly relaxes time-of-flight budgets, and easier probing and rework because the trace is accessible. Microstrip disadvantages: greater radiation (the trace acts as a weak antenna because the fields are not enclosed), greater susceptibility to EMI ingress, stronger crosstalk to adjacent microstrip traces, and impedance sensitivity to solder mask coverage and thickness.

Stripline advantages: lower crosstalk because the trace is shielded by two ground planes, better EMI containment (fields are fully enclosed), and more predictable impedance because the solder mask does not affect the field. Stripline disadvantages: higher dielectric loss (all field lines travel through the dielectric), slower propagation velocity, more complex back-drilling (the trace is embedded), and heat trapping (no convective cooling from the trace itself).

For LPDDR5X at 8533 MT/s and above, stripline is preferred for DQ, DQS, WCK and CK despite the higher loss, because crosstalk and EMI become the dominant SI limits. For LPDDR4 and LPDDR4X, microstrip is acceptable and often used because the loss budget is relaxed. Dual-stripline (two signal layers between the same plane pair) saves a plane but introduces layer-to-layer crosstalk and should be avoided for adjacent DQ byte lanes.

---

### Q6. How does the stackup influence back-drilling and via-stub control?

**Answer:**

Back-drilling is a secondary drilling operation that removes the unused portion of a through-hole via after plating, reducing the via stub. The stackup constrains how effective back-drilling can be because the drill must stop at a specified depth without damaging the signal layer or the reference plane below it.

The back-drill stop point is typically 4-8 mils (100-200 um) above the signal layer using the via. The plane-to-plane separation must accommodate this tolerance, so the dielectric between the signal layer and the next plane below must be at least 8-12 mils thick. A very tight stackup (plane-to-plane less than 6 mils) may not permit back-drilling at all. For LPDDR5X designs on thick PCBs (2 mm and above), back-drilling is essential to reduce via stub resonance below the Nyquist frequency. This must be planned at stackup design time.

The signal layer closest to the surface suffers the worst stubs when through-hole vias drop to deep inner layers. Placing the LPDDR DQ stripline layer on layer 3 (plane on layer 2, stripline on layer 3, plane on layer 4) minimises the stub below the signal exit point. If the DQ layer is on layer 7 of a 10-layer board, the via stub above the signal is 6 layers long — typically 60-80 mils — which resonates near 8-10 GHz and directly hits the LPDDR5X Nyquist.

An alternative to back-drilling is blind or buried vias, which only connect a subset of layers and have no stub by construction. Blind vias are drilled before lamination of the remaining layers and are more expensive. The stackup must define which layer pairs use blind vias. HDI (high density interconnect) stackups use laser-drilled microvias in the outer build-up layers (typically 1-2 layers on each side), eliminating stubs in the BGA fanout region.

---

### Q7. How are reference-plane changes handled when a DQ byte crosses layers?

**Answer:**

When a DQ trace transitions between signal layers through a via, the return current must also change reference planes. If the two reference planes are both ground, a nearby ground stitching via provides a low-inductance return path. If the two planes are a ground and a power plane, the return current must flow through the decoupling capacitance between the planes, which is high-impedance at LPDDR frequencies. This creates a return-path discontinuity that shows up as a common-mode current, an impedance bump at the via, and a crosstalk aggressor for other signals sharing the plane pair.

Layout rules for LPDDR mandate a ground stitching via within 0.5-1.0 mm of every signal via for high-speed signals. For LPDDR5X, the rule tightens to a ground stitching via per signal via (1:1 ratio) for DQS and WCK.

The stackup simplifies the problem if the routing layers are arranged ground-signal-ground-signal-ground so that any layer transition sees a ground plane on both sides. A stackup that interleaves power and ground planes between signal layers creates transitions where the return path is not obvious and should be avoided for LPDDR-heavy boards.

For the specific case of the VDDQ power plane sitting between DQ signal layers, the stackup must provide close-coupled ground planes on the other side of each DQ layer and embedded capacitance (thin prepreg, less than 50 um, between VDDQ and adjacent ground) to provide a high-frequency return path. Embedded capacitance is an effective technique but adds cost and must be specified at stackup time.

---

### Q8. How is the PCB stackup verified against LPDDRx impedance and loss targets before fabrication?

**Answer:**

Stackup verification uses a combination of 2D field-solver calculations, 3D full-wave simulation of critical structures, and fabrication coupons. The verification happens in three phases: design-time, pre-fab, and post-fab.

At design time, a 2D field solver (Polar Si9000, Cadence Sigrity, HyperLynx Stackup Editor) computes the characteristic impedance for each trace width and dielectric thickness combination at each signal layer. The output is an impedance table used by the layout tool to apply impedance-controlled widths when routing. The solver also produces insertion loss (dB/cm) as a function of frequency for each stackup configuration, which feeds the channel budget. The designer iterates the stackup until impedance, loss and routing density are all acceptable.

Pre-fab verification sends the stackup to the PCB manufacturer, who runs their own field solver with their actual material parameters and adjusts trace widths to compensate for etch compensation, plating variation, and Dk tolerance. The fab returns a revised stackup with corrected trace widths; the layout is adjusted to match. This step is essential because the fab's process tolerances (Dk plus/minus 5%, dielectric thickness plus/minus 10%, etch plus/minus 12 um) can shift the impedance by 5-10% if not compensated.

Post-fab verification uses impedance test coupons on each panel. The coupon contains short trace segments at each impedance target, tested with a TDR (time-domain reflectometer). The typical tolerance is plus/minus 10% for LPDDR4/5 and plus/minus 7-8% for LPDDR5X. Panels out of tolerance are scrapped or reworked. For safety-critical applications (automotive), every panel is tested; for mobile, sample testing is common.

---

See also:
- [PCB Routing for LPDDR](../06_package_and_system/pcb_routing_for_lpddr.md)
- [PCB Signal Integrity](pcb_signal_integrity.md)
- [PCB Power Integrity](pcb_power_integrity.md)
