# PCB Power Integrity for LPDDRx

This section covers PCB-level power integrity (PI) for LPDDRx interfaces. PCB PI is concerned with delivering clean, low-impedance power to the SoC LPDDR supplies (VDDQ, VDD2, VDD1, and their ground returns) and, in non-PoP designs, to the DRAM supplies. A well-designed PCB PDN must hit the target impedance from DC to the highest frequency where on-die and package decoupling take over, typically 100-500 MHz.

---

### Q1. What are the LPDDRx supply rails that the PCB must deliver, and what are their budgets?

**Answer:**

LPDDRx requires several distinct power rails, each with its own tolerance and current requirement. The PCB PDN must distribute each rail to the SoC and, for non-PoP, also to the DRAM package.

VDDQ is the IO supply used by the LPDDR drivers and receivers. It is the most sensitive rail because its noise directly appears on the signal swing. LPDDR4 uses 1.1V, LPDDR4X uses 0.6V, LPDDR5 uses 0.5V, and LPDDR5X uses 0.5V or 0.35V in low-power modes. The tolerance is typically plus/minus 3-5% static and plus/minus 5-8% total (including AC ripple). Current depends on the byte lane count and activity; a single x16 channel running at 6400 MT/s with 50% switching can draw 200-400 mA average and 1-2 A peak transient. A dual-channel SoC can hit 500 mA average and 3 A peak.

VDD2 (also called VDD2H) is the core supply for the DRAM and the controller IO logic. LPDDR4/4X uses 1.8V, LPDDR5/5X uses 1.05V. Tolerance is plus/minus 5%. Current is typically 100-300 mA per channel.

VDD1 is the charge-pump supply for the DRAM's internal boost circuits. It is 1.8V with plus/minus 5% tolerance and 10-50 mA current, depending on DRAM mode. VDD1 is usually derived from the same PMIC rail as VDD2 through a ferrite-bead filter.

VDDQ_CA (LPDDR5X only) is an optional separate supply for the CA drivers, introduced to isolate CA noise from DQ.

Each rail is typically fed from a dedicated buck converter output or LDO, with separate decoupling. Some designs share VDD2 between LPDDR and other SoC blocks, but VDDQ is almost always dedicated to LPDDR to keep the noise environment controlled.

---

### Q2. How is the PCB PDN target impedance calculated for VDDQ?

**Answer:**

The target impedance Z_target is the maximum allowable PDN impedance at any frequency such that the worst-case transient current does not cause a voltage excursion exceeding the noise budget. The formula is Z_target equals V_ripple divided by I_transient.

For an LPDDR5 VDDQ rail at 0.5V with a plus/minus 5% ripple budget, V_ripple is 25 mV. The transient current I_transient is estimated from the worst-case switching pattern: for a single x16 channel with all 16 DQ plus 2 DQS plus 2 DQS_n switching simultaneously at 6400 MT/s, the per-bit driver current is approximately V/Z_drv times 0.5 (half duty, half at high and half at low), so per bit it is 0.5V/40 ohm times 0.5 equals 6.25 mA average and approximately 12.5 mA peak. For 20 signals switching together, peak is 250 mA. In practice switching patterns are less than 100% aligned, but the peak-to-average ratio is 4-6x so the transient I_transient is approximately 0.5-1.0 A.

For I_transient equals 0.8 A and V_ripple equals 25 mV, Z_target equals 25 mV divided by 0.8 A equals 31 mohm. The PCB PDN must present less than 31 mohm across the frequency range where PCB decoupling is effective. Below 1 MHz the PMIC regulation loop takes over; above 500 MHz the package and on-die decoupling dominate. The PCB PDN must therefore hit 31 mohm from approximately 1 MHz to 500 MHz.

For LPDDR5X at 8533 MT/s, the transient current rises proportionally, giving Z_target of roughly 20-25 mohm over a wider frequency range (up to 700 MHz). Hitting this requires more capacitors, tighter placement, and often embedded capacitance.

The target is not a hard cliff — violating it by 20-30% may still be acceptable if the offending frequency is not excited by the actual switching pattern, but it is a safe design target that removes the need for detailed stimulus analysis.

---

### Q3. How are decoupling capacitors chosen and placed on the PCB for LPDDR?

**Answer:**

The decoupling capacitor strategy for LPDDR PCB uses a hierarchy of capacitor values, each effective in a different frequency range, combined to achieve low PDN impedance from DC to hundreds of MHz.

Bulk capacitors (22-100 uF, typically ceramic X5R/X7R 0805 or tantalum) are placed within 5-10 mm of the SoC BGA and handle frequencies from 1 kHz to 1-5 MHz. They support large, slow transients such as the inrush at memory controller wake-up or bank activation bursts. Typical placement: 4-8 bulk caps per VDDQ rail around the BGA perimeter.

Mid-frequency capacitors (1-10 uF ceramic X5R/X7R, 0402 or 0201) handle 1 MHz to 50 MHz and are placed within 2-5 mm of the BGA. Their self-resonant frequency (SRF) is typically 5-20 MHz, governed by capacitance and package inductance. Typical placement: 10-20 per VDDQ rail.

High-frequency capacitors (100 nF to 1 uF, 0201 or 01005) handle 50 MHz to 500 MHz and must be placed as close as possible to the BGA — within 1-2 mm, ideally via-in-pad under the BGA. Their SRF is 30-100 MHz, and their ESL (equivalent series inductance, 0.5-1.5 nH) dominates above SRF. Typical placement: 15-40 per VDDQ rail, densely packed.

Very high frequency capacitors (1-10 nF, 01005 or embedded MLCC) and embedded capacitance (thin prepreg between VDDQ and ground planes providing 0.3-1 nF/cm^2 distributed) handle 500 MHz to several GHz. These are the only effective decoupling above 500 MHz because all discrete caps are inductance-limited at those frequencies.

Placement rules: minimise the loop inductance from cap to IC pin by placing the cap on the same side as the BGA when possible, using via-in-pad for the cap pads, and placing ground return vias adjacent to the cap VDDQ vias. A single ground via and a single VDDQ via in close proximity (approximately 0.5 mm apart) gives roughly 0.5-1.0 nH of loop inductance; anything larger than this defeats the purpose of small caps.

Each capacitor's effectiveness is ESL-limited at high frequencies. A 100 nF cap with 0.8 nH ESL has SRF at 17 MHz; above that its impedance rises as jwL and is no better than a 0.8 nH inductor. Multiple small caps in parallel reduce the effective ESL proportionally: 10 caps with ESL 0.8 nH each, if placed close together so their loop inductances are independent, give 80 pH of effective inductance. This is the primary reason LPDDR PCBs use many small caps rather than a few large ones.

---

### Q4. What is embedded capacitance and when is it used for LPDDR?

**Answer:**

Embedded capacitance is a PCB construction technique where a thin high-Dk dielectric (typically 25-50 um of epoxy-filled with barium titanate particles, Dk 10-40) is laminated between a VDDQ plane and an adjacent ground plane. This forms a large, distributed parallel-plate capacitor across the entire area of the board, with capacitance density of 0.3-3 nF per cm squared.

The advantage of embedded capacitance is ultra-low ESL because there is no discrete capacitor body and no vias: the capacitance is distributed under the exact location of the current sink, eliminating the loop inductance penalty. It is effective up to several GHz where discrete caps have long since become inductive. For a typical LPDDR BGA footprint of 15x15 mm, embedded capacitance at 1 nF/cm^2 provides approximately 2.25 nF of distributed capacitance directly under the BGA, which is invaluable at 500 MHz to 2 GHz.

Embedded capacitance is not a substitute for discrete caps at low and mid frequencies — its absolute capacitance is small compared to a single 10 uF cap. It complements discrete decoupling at high frequencies where discrete caps have run out.

Drawbacks: the thin dielectric is less mechanically robust and has lower insulation voltage (typically 50-100 V, acceptable for LPDDR but not for power supplies over 12V), the Dk is higher so traces on adjacent layers must be adjusted for impedance, and the material cost is higher. Some embedded capacitance materials are sensitive to humidity and temperature cycling, which matters for automotive designs.

For LPDDR5X on high-density boards, embedded capacitance under the SoC VDDQ plane is increasingly common. LPDDR4/5 on standard FR-4 stackups typically rely on discrete decoupling alone.

---

### Q5. How is PCB IR drop analysed for LPDDR power rails?

**Answer:**

PCB IR drop analysis is a DC simulation that computes the steady-state voltage at every pin of every load, given the power plane geometry, via patterns, source locations, and load current distribution. For LPDDR, IR drop is usually dominated by the via pattern and plane cut-outs rather than by sheet resistance, because the power plane resistance is small (a 1 oz copper plane has sheet resistance of 0.48 mohm per square).

The analysis flow is: import the PCB layout, define the power source (PMIC output), define each load (SoC BGA VDDQ balls, DRAM BGA VDDQ balls for non-PoP) with their worst-case current, run the DC solver, and inspect the voltage at each load pin. The output is a coloured voltage map showing which regions are at the target voltage and which regions have sagged. Current-density maps show where the current is flowing and whether any region is saturated.

Common IR drop problems for LPDDR:

- The VDDQ plane is cut by trace routing on the adjacent layers, reducing the effective plane area. The remaining plane has high local current density near the cut edges.
- Too few vias connect the VDDQ plane to the SoC BGA, so all the current must pass through a small number of vias. Each via has resistance of approximately 1-2 mohm, so 5 vias in parallel carry 1 A with 0.4 mV of drop — acceptable; but 2 vias in parallel give 1 A with 1 mV of drop, marginal; 1 via gives 1 A with 2 mV of drop, unacceptable.
- The PMIC is placed far from the SoC (more than 30 mm), so the DC IR drop along the VDDQ path is significant. A 15 mm wide, 15 mm long VDDQ plane segment has resistance 0.48 mohm per square times 1 equals 0.48 mohm, giving 0.48 mV at 1 A. Narrower paths or longer routes increase this proportionally.
- VDDQ pass-through regions between BGAs where the plane is necked down to pass between via fields.

The DC IR drop should be budgeted separately from the AC ripple, with the PCB DC drop target typically 5-10 mV out of the 15-25 mV total budget. The remainder is reserved for package and on-die drop plus AC ripple.

---

### Q6. Where should the PMIC be placed relative to the SoC for LPDDR power delivery?

**Answer:**

The PMIC (Power Management IC) should be placed as close to the SoC as the mechanical and thermal constraints allow, typically within 10-25 mm of the SoC BGA edge. Closer placement reduces the PCB plane resistance (better DC IR), reduces the plane inductance between PMIC output caps and SoC (better AC impedance), and reduces the exposed trace length for EMI coupling.

Specific considerations for LPDDR:

- VDDQ output of the PMIC should have its output capacitors placed immediately at the PMIC output, then the VDDQ plane should route directly to the SoC BGA VDDQ balls. A clean, wide, uncut plane gives the lowest impedance.
- If the PMIC is a multi-output device with VDDQ, VDD2, VDD1 all on the same chip, the outputs should be placed on the side of the PMIC facing the SoC.
- The PMIC output switching node (SW pin of a buck regulator) is a noise source and should be placed away from LPDDR signals. The switching frequency of mobile PMICs is typically 2-4 MHz, below LPDDR Nyquist, but harmonics extend to hundreds of MHz.
- Thermal coupling matters: the PMIC dissipates significant power (a 1 A VDDQ buck at 85% efficiency dissipates 100-150 mW; add VDD2 and other rails and the PMIC may dissipate 500-1000 mW). If the PMIC is placed too close to the LPDDR DRAM (non-PoP), it can raise the DRAM junction temperature and push it towards thermal derating.
- The ground return path must be short: a single continuous ground plane under both the PMIC and the SoC provides the return path automatically.

For PoP designs, the PMIC is usually placed directly adjacent to the SoC package on the PCB, 5-15 mm from the BGA edge. For non-PoP automotive or embedded designs, the PMIC may be placed further away due to thermal separation or mechanical constraints, and the designer must compensate with wider planes and more decoupling.

---

### Q7. How is PDN impedance simulated for a PCB with LPDDR?

**Answer:**

PCB PDN impedance simulation uses a frequency-domain solver to compute Z(f) looking into the PDN from the SoC VDDQ pin, including the effects of the plane geometry, via connections, discrete capacitors (with their ESL and ESR models), and the PMIC source impedance.

The tools are Cadence Sigrity PowerSI and PowerDC, ANSYS SIwave and Q3D, HyperLynx PI, and Simbeor PI. The flow is: import the PCB layout, define the VDDQ net and its ground return, assign capacitor models (from the vendor Touchstone data or S2P files), define the source (PMIC output) and the sink (SoC BGA VDDQ balls), run the solver to produce Z(f).

The output is a PDN impedance plot from DC to multi-GHz. The designer checks: is the impedance below Z_target across the relevant frequency range; are there any resonance peaks (where plane inductance resonates with capacitor ESL, typically at hundreds of MHz); does the PDN meet the target with the worst-case capacitor tolerance.

Resonance peaks are the most common problem. A typical PCB PDN has an anti-resonance between the decoupling caps and the plane inductance, often in the 100-500 MHz range where LPDDR signalling is most sensitive. Peaking of 2-3x Z_target is common with a naive cap placement. Solutions include: adding capacitors at values that flatten the response (a mix of 100 nF, 22 nF, 10 nF, 4.7 nF), damping the resonance with ESR (intentional use of higher-ESR caps to widen the SRF notch), shortening the plane-to-cap loop to push the resonance above the band of interest, and using embedded capacitance to flatten the high-frequency region.

The PDN impedance should be simulated with both typical and worst-case capacitor values (tolerance, DC bias de-rating). X5R/X7R ceramic caps lose 30-80% of their nominal capacitance at their rated DC bias voltage, so a 10 uF 10V cap at 5V DC bias may effectively be 5-7 uF. This de-rating must be included in the model or the simulation gives an optimistic answer.

---

### Q8. What is the role of ferrite beads in LPDDR power delivery, and when should they be used?

**Answer:**

Ferrite beads are frequency-dependent impedance elements that look like a small resistor at low frequencies and a higher impedance (tens to hundreds of ohm) at higher frequencies due to the lossy nature of the ferrite material. They are commonly used to isolate noisy power rails or to filter high-frequency noise between power domains.

For LPDDR, ferrite beads are sometimes used on VDD1 (the charge-pump supply) to filter noise between the main 1.8V rail and the DRAM's sensitive charge-pump. They are also used on the PMIC-to-SoC path for VDD2 in some designs to attenuate PMIC switching noise before it reaches the LPDDR controller's PLL/DLL.

Ferrite beads should NOT be used on VDDQ. The reason: VDDQ is a high-current rail carrying sharp transients, and the ferrite bead's impedance at high frequencies is exactly the frequency range where the VDDQ PDN must be low-impedance. Putting a 50 ohm bead in series with VDDQ at 100 MHz would catastrophically raise the PDN impedance and kill the rail.

Ferrite bead selection matters: the bead's DC resistance must be low enough to meet the DC IR budget (typically 10-50 mohm for LPDDR-adjacent rails), its impedance curve must peak at the target filter frequency, and its current rating must exceed the worst-case load current with margin.

A common pitfall: ferrite beads have an anti-resonance with the capacitors on their far side. A bead with L_effective equals 1 uH paired with a 1 uF cap has resonance at 160 kHz; at the resonance, the PDN impedance peaks sharply. Q damping (using a bead with higher loss, or adding a small resistor in series with the cap) suppresses the peak. Without damping, the ferrite bead can actually create more noise than it filters.

The general rule for LPDDR: use ferrite beads for clean-supply isolation (PLL supplies, auxiliary low-current rails) but never in the main IO power path. For high-current filtering, use pi-filter topologies with capacitors on both sides of a bead, with careful resonance analysis.

---

### Q9. How does decoupling capacitor de-rating affect LPDDR PDN design?

**Answer:**

Multi-layer ceramic capacitors (MLCCs) with X5R, X7R, or X7S dielectrics have nominal capacitance that is significantly higher than their effective capacitance under typical operating conditions. The three main de-rating effects are DC bias, temperature, and aging. All three reduce the effective capacitance below the nameplate value, and they compound.

DC bias de-rating is the largest effect. Ceramic dielectrics are non-linear: the dielectric constant decreases as the applied electric field increases. A 10 uF 6.3V X5R 0402 cap used at 3.3V may have only 40-50% of its nominal capacitance — i.e., 4-5 uF effective. At 5V it may be 20-30% — i.e., 2-3 uF effective. For LPDDR5 VDDQ at 0.5V, de-rating is mild because the bias is small relative to the cap rating; for VDD2 at 1.05V the de-rating may be 10-20%; for VDD1 and VDD2H at 1.8V on a 6.3V-rated cap, de-rating is 20-40%.

Temperature de-rating for X5R is approximately plus/minus 15% from -55 to +85C. X7R is plus/minus 15% from -55 to +125C. C0G/NP0 dielectrics have negligible temperature dependence but are only available at small capacitance (less than 100 nF typically).

Aging is a gradual decrease of 1-5% per decade hour after the cap is soldered. This effect is usually negligible within the product lifetime for new caps, but caps stocked for months or years before assembly may show measurable loss.

Design implications: PDN simulations must use the de-rated capacitance, not the nameplate. A common workflow is to apply the derating factor to the cap model before import, or use cap vendor S-parameter data that includes bias de-rating. A typical design margin is to size caps at 2x the calculated value so that de-rating does not violate the PDN target.

LPDDR designs often use low-voltage caps (2.5V or 4V rating) on the VDDQ 0.5V rail to minimise de-rating — the higher the cap's voltage rating versus the applied voltage, the less the de-rating penalty. However, lower-rated caps may have other trade-offs (larger size for the same capacitance, lower temperature range).

---

### Q10. How does the PCB PDN integrate with the package and on-die PDN to form the full decoupling hierarchy?

**Answer:**

The complete LPDDR PDN is a hierarchy of decoupling elements spanning four physical levels, each effective in a different frequency range. Designing any one level in isolation is a mistake; the PCB PDN only has to meet the target impedance up to the frequency where the next level (package) takes over.

Level 1: PMIC and bulk caps (DC to 1 MHz). The PMIC's control loop regulates VDDQ to its set point with bandwidth of typically 10-100 kHz. Bulk caps at the PMIC output and near the SoC extend this range to approximately 1 MHz.

Level 2: PCB mid-frequency discrete caps (1 MHz to 50 MHz). These are the 1-10 uF ceramics discussed earlier. Their effectiveness depends on placement and loop inductance to the SoC BGA.

Level 3: PCB high-frequency caps plus package decoupling (50 MHz to 500 MHz). On the PCB side, 100 nF to 1 uF 0201 caps near the BGA contribute here. On the package side, the substrate has its own smaller caps (embedded in the package or placed as discrete 0201/01005 on the package lid) which take over where the PCB caps become inductance-limited by their ESL and by the via path up to the BGA ball.

Level 4: On-die decoupling (500 MHz and above). On the die, MIM (metal-insulator-metal) caps and MOM (metal-oxide-metal) caps are placed directly under the IO cells. The capacitance is small (tens of pF to a few nF per IO region) but the ESL is sub-pH, so they are effective well into the GHz range.

The hand-off frequency between levels is determined by the loop inductance of the level below. Roughly: bulk caps are effective up to 1/(2 pi sqrt(LC)) with L equals loop inductance from cap to load. A bulk cap with 10 nH of loop inductance and 10 uF of cap has SRF at 0.5 MHz; above 1 MHz it is ineffective and the next level takes over.

The design approach is to compute the PDN impedance contribution of each level in isolation, then combine them and check that the total is below Z_target across the full frequency range. If a gap exists (frequencies where no level is effective), that is where anti-resonance occurs and the impedance spikes. Common gap locations are 5-20 MHz (between bulk and mid-frequency) and 200-500 MHz (between PCB and package), and these are where most PDN problems show up.

For LPDDR, the PCB PDN must be designed with the package and on-die decoupling in mind. The SoC vendor typically provides a decoupling specification: a minimum cap count and value for the board, with placement guidelines. Meeting this spec usually guarantees a working design; beating it gives margin but adds BOM cost. Underspec'ing the PCB PDN and relying on on-die decoupling alone is not viable because the on-die cap cannot fight PCB-level current transients — it only smooths the high-frequency tail.

---

See also:
- [PCB Stackup Choice](pcb_stackup_choice.md)
- [PCB Signal Integrity](pcb_signal_integrity.md)
- [PCB Thermal](pcb_thermal.md)
- [Power Integrity for LPDDR (on-die)](../05_signal_and_power_integrity/power_integrity_for_lpddr.md)
