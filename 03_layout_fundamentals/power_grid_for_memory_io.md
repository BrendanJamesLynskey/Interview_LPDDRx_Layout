# Power Grid for Memory IO

This section covers the design of the power distribution network (PDN) for LPDDR memory IO regions. The power grid is one of the most layout-critical aspects of LPDDR PHY design because the low VDDQ voltage leaves very little margin for IR drop and noise.

---

### Q1. What are the power delivery requirements for an LPDDR5 IO region?

**Answer:**

The LPDDR5 IO region has stringent power delivery requirements driven by the low VDDQ voltage (0.5V) and the high transient currents from output drivers. The key requirements are maintaining DC IR drop below 3-5% of VDDQ (15-25 mV maximum), providing AC impedance low enough to keep transient voltage droop below 5% during switching events, supplying peak currents of 100-200 mA per byte lane (8 DQ drivers switching simultaneously plus ODT current), and ensuring resonance-free PDN impedance across the frequency range from DC to the data rate.

The total VDDQ current for a 4-channel LPDDR5 x32 interface can reach 400-800 mA during worst-case switching patterns. This current flows through the on-die power grid, through the bump and package interconnect, and ultimately from the voltage regulator on the PCB or in the PMIC (Power Management IC).

The power grid must handle both the DC (average) current and the AC (transient) current. The DC current determines the IR drop, which is addressed by having sufficiently wide and numerous metal straps in the power grid. The AC current determines the transient voltage droop (Ldi/dt noise), which is addressed by having low-inductance connections and sufficient decoupling capacitance close to the switching circuits.

The VDDQ power grid is separate from other power domains (VDD1, VDD2, core VDD). Each domain has its own power grid with independent routing. The VSS (ground) grid is shared but must be robust enough to serve as the return path for all domains without excessive ground bounce.

---

### Q2. How is the VDDQ power grid structured in the PHY layout?

**Answer:**

The VDDQ power grid in the PHY layout is typically structured as a mesh of horizontal and vertical metal straps on the upper metal layers, connected to the IO cells through a via stack. The structure follows a hierarchical approach.

The top metal layers (M12-M15 in a 15-metal process) carry wide VDDQ and VSS straps that form the coarse grid. These straps are 5-20 um wide, spaced 10-40 um apart, and carry the bulk of the current from the package bumps to the IO region. The VDDQ bumps connect to these top metal straps through via stacks.

The intermediate metal layers (M8-M11) carry narrower VDDQ and VSS straps that form a finer mesh. These straps distribute current from the coarse grid to individual IO cells. They are typically 2-5 um wide, spaced 5-15 um apart, and run orthogonally to the top-layer straps.

The lower metal layers (M1-M7) carry the local VDDQ and VSS connections within each IO cell. These are part of the IO cell's internal power routing, designed by the custom/analog layout team.

The power grid mesh density (number of straps per unit area) must be highest in the IO cell region, where the current density is greatest. As the grid extends inward from the die edge (toward the serialisers and FIFOs), the current density decreases, and the grid can be sparser.

The grid design must also consider the alternating VDDQ/VSS pattern: every VDDQ strap should have an adjacent VSS strap to provide a low-inductance current loop. The spacing between VDDQ and VSS straps should be minimised (subject to DRC rules) to reduce loop inductance.

Decoupling capacitors (MIM or MOM) are placed within the power grid, typically between the intermediate and lower metal layers. These capacitors provide local charge storage that responds to high-frequency current demands faster than the distant package and PCB decoupling.

---

### Q3. What types of on-die decoupling capacitors are used, and where are they placed?

**Answer:**

On-die decoupling capacitors provide local charge storage that responds to high-frequency current transients. Several types are used in LPDDR PHY layouts.

MIM (Metal-Insulator-Metal) capacitors are formed by two parallel metal plates separated by a thin dielectric layer. They are available in most advanced process nodes as a special option (requiring an additional mask). MIM capacitors provide high capacitance density (typically 10-20 fF/um^2) and low ESR (Equivalent Series Resistance), making them effective for decoupling at frequencies from 100 MHz to several GHz. They are placed in the PHY layout between dedicated metal layers (the MIM top and bottom plates) and consume planar area without affecting the standard metal routing.

MOM (Metal-Oxide-Metal) capacitors are formed by interdigitated metal fingers on standard metal layers. They use the parasitic capacitance between closely spaced metal lines, without requiring any special process options. MOM capacitors have lower density than MIM capacitors (typically 2-5 fF/um^2) but can be placed on any metal layer and can be inserted into otherwise unused routing areas.

NMOS decoupling capacitors use NMOS transistors with their gate connected to VDDQ and source/drain connected to VSS. The gate capacitance provides decoupling. These are effective at lower frequencies and provide relatively high capacitance density in the active silicon area. They are often placed in the guard ring areas or in unused spaces within the IO cell array.

For placement, the principle is to place decoupling as close as possible to the noise source (the switching drivers). MIM capacitors should be placed directly over or adjacent to the IO cells, on the metal layers between the IO cell and the bump pad layer. MOM capacitors can be distributed throughout the PHY area on any available metal layers. NMOS decoupling can fill any empty silicon area within the PHY.

The total on-die decoupling for a single byte lane (8 DQ) should be approximately 100-500 pF for effective high-frequency decoupling. This is distributed across MIM (50-200 pF), MOM (20-100 pF), and NMOS (50-200 pF) capacitors.

---

### Q4. How is IR drop analysed for the LPDDR PHY power grid?

**Answer:**

IR drop analysis determines the static voltage drop from the package bumps to each IO cell under DC current loading conditions. This analysis is performed using power grid analysis tools such as Cadence Voltus, Synopsys RedHawk, or ANSYS RedHawk-SC.

The analysis flow begins with extracting the power grid structure from the layout database. The extraction produces a resistive network model that includes every metal strap, via, and bump in the VDDQ and VSS grids. The resolution of the extraction must be fine enough to capture the detailed routing within the IO cells (typically 0.1-0.5 um grid for the analysis mesh).

Current sources are then attached to each IO cell location in the resistive network. The current values represent the worst-case DC current draw: all drivers active, all ODTs enabled, worst-case data pattern (all zeros or all ones, depending on the driver topology). For 8 DQ pins at 6.25 mA each plus ODT current, the total per-byte current is approximately 50-100 mA.

The tool solves the resistive network (using sparse matrix methods) to determine the voltage at each node. The IR drop is the difference between the bump voltage (which is assumed to be at the nominal VDDQ) and the voltage at the IO cell supply pin. The results are visualised as a colour-coded map showing the IR drop distribution across the grid.

Acceptable IR drop depends on the noise budget allocation. Typically, the DC IR drop is allocated 5-10 mV (1-2% of VDDQ at 0.5V), leaving the remaining budget for AC noise. If the analysis shows IR drop exceeding this threshold, the layout must be improved by widening power straps, adding more via connections, or placing additional VDDQ bumps closer to the high-current regions.

The analysis should be repeated for multiple scenarios: all channels active (worst-case total current), single channel active (worst-case current density), and different data patterns.

---

### Q5. How does ESD protection affect the power grid design?

**Answer:**

ESD protection structures are integral to the power grid because they provide the discharge paths for electrostatic events. The ESD clamp cells (placed between VDDQ and VSS in the pad ring) must be connected to robust power rails that can carry the transient ESD current (several amps for nanoseconds) without damage.

The VDDQ and VSS rails feeding the ESD clamps must be separate from or at least wider than the normal IO power distribution, because the ESD current is much higher than the normal operating current. If the ESD current flows through the same narrow metal straps that feed the IO cells, the resistive voltage drop could damage the thin metal or the underlying transistors.

ESD clamp placement must ensure that every IO pad has a low-impedance path to the nearest clamp. The JEDEC CDM (Charged Device Model) specification requires that the resistance from any pad to the nearest clamp be less than a few ohms, including the metal routing resistance. This constrains the maximum spacing between clamps along the pad ring.

The power clamp cells also provide decoupling capacitance (from the large NMOS clamp transistor's gate capacitance). This decoupling is beneficial for normal operation, but the clamp cell design must ensure that the clamp does not trigger during normal supply voltage fluctuations. The trigger voltage of the clamp must be above the maximum expected VDDQ overshoot.

For layout, the ESD rails form a dedicated ring around the IO region. This ring connects to the main VDDQ and VSS power grid through multiple taps. The ring must be continuous (no breaks) and wide enough to carry the ESD current (typically 10-20 um wide on the top metal layer).

Guard rings around the PHY (between the VDDQ domain and adjacent voltage domains) also serve an ESD function. They provide a defined discharge path for cross-domain ESD events and prevent latch-up triggering in the IO cells.

---

### Q6. What is the role of guard rings in the LPDDR PHY power grid, and how are they implemented?

**Answer:**

Guard rings are continuous rings of doped semiconductor (N+ and P+ implants) connected to power or ground rails that provide electrical isolation between different circuit blocks or voltage domains. In the LPDDR PHY, guard rings serve several purposes.

Latch-up prevention is the primary function. CMOS circuits are susceptible to latch-up, a parasitic thyristor effect where the PMOS-substrate-NMOS path creates a regenerative current loop. Guard rings break this path by collecting stray carriers before they can trigger the latch-up. In IO cells, where large currents flow through the substrate during switching, latch-up risk is elevated.

Noise isolation between the VDDQ domain (noisy, with high-speed switching) and the VDD2/core domains (sensitive, containing PLL and VREF generators) is achieved by placing guard rings at the domain boundary. The guard rings absorb substrate noise generated by the IO switching, preventing it from reaching the sensitive circuits.

ESD current steering during ESD events directs the discharge current along defined paths through the guard rings rather than allowing it to flow through sensitive circuit areas.

Guard rings are implemented as follows. A P+ guard ring is placed in the P-substrate, connected to VSS. It surrounds the N-well regions of PMOS devices. An N+ guard ring is placed in an N-well, connected to VDDQ or VDD. It surrounds the P-well regions of NMOS devices. A deep N-well ring (if available in the process) provides additional isolation by creating a buried barrier that blocks substrate current flow.

For layout, guard rings consume area (typically 2-5 um wide plus spacing rules on each side). The layout engineer must plan for guard ring area during floorplanning. Guard rings must be continuous (no breaks) around the protected region. Every ring must have regular contacts to its power rail (typically every 5-10 um along the ring length) to maintain low resistance. Guard rings at voltage domain boundaries (VDDQ to core VDD) must connect to the appropriate supply for each domain, with a transition structure at the boundary.

---

### Q7. How is the PDN impedance profile designed for LPDDR5?

**Answer:**

The PDN (Power Distribution Network) impedance profile describes how the impedance seen by the IO circuit varies with frequency. The target is a flat, low-impedance profile from DC to beyond the data rate, with no resonance peaks that could amplify noise at specific frequencies.

The target impedance is derived from the noise budget and the maximum switching current:

```
Z_target = V_noise_max / I_switching_max
```

For LPDDR5 at 0.5V VDDQ with a 5% noise budget (25 mV) and 200 mA maximum switching current:

```
Z_target = 25 mV / 200 mA = 125 mohm
```

This is an extremely low impedance that must be maintained across all frequencies. Different decoupling elements dominate at different frequency ranges. Package and PCB bulk capacitors (1-10 uF) provide low impedance at low frequencies (1 kHz to 10 MHz). Package capacitors (10-100 nF) cover the mid-frequency range (10 MHz to 500 MHz). On-die MIM and MOM capacitors (100-500 pF) provide decoupling at high frequencies (500 MHz to 5 GHz). On-die intrinsic capacitance (gate oxide, junction capacitance) provides decoupling at the highest frequencies (above 5 GHz).

The challenge is avoiding resonance between decoupling stages. When the inductive impedance of the package interconnect (bumps, traces, vias) resonates with the capacitance of the on-die decoupling, the PDN impedance peaks at the resonant frequency. This peak can exceed the target impedance and amplify noise at that frequency.

For layout, the resonance frequency depends on the package inductance and on-die capacitance:

```
f_resonance = 1 / (2 * pi * sqrt(L_pkg * C_die))
```

With L_pkg = 100 pH (typical bump + package trace inductance) and C_die = 200 pF:

```
f_resonance = 1 / (2 * pi * sqrt(100e-12 * 200e-12)) = 1.13 GHz
```

This resonance falls within the LPDDR5 operating frequency range and must be damped. Damping is achieved by the ESR of the on-die capacitors and the resistance of the power grid. The layout engineer can increase the damping by adding resistive elements (using narrow metal connections with intentional resistance) in series with some decoupling capacitors.

---

### Q8. How does the ground bounce phenomenon affect the LPDDR power grid, and how is it mitigated?

**Answer:**

Ground bounce occurs when a large number of output drivers switch simultaneously, creating a transient current spike through the VSS (ground) path. The inductance in the ground path (bond wires, bumps, package traces, on-die routing) converts this current spike into a voltage spike on the local ground reference, causing the ground potential to momentarily "bounce" above the ideal zero voltage.

For LPDDR5, ground bounce is particularly problematic because the signal swing is only 0.5V. A ground bounce of 25 mV (which is a modest Ldi/dt event) shifts the local ground reference by 5% of VDDQ, directly reducing the noise margin for all signals referenced to that ground.

The magnitude of ground bounce is:

```
V_bounce = L_ground * di/dt
```

For 8 DQ pins switching simultaneously with 12.5 mA per pin in 100 ps (typical transition time):

```
di/dt = 8 * 12.5 mA / 100 ps = 100 mA / 100 ps = 1 A/ns = 10^9 A/s
V_bounce = L_ground * 10^9
```

For V_bounce < 25 mV: L_ground < 25 pH

This is an extremely low inductance that requires careful layout design.

Mitigation strategies include minimising ground inductance by using multiple parallel VSS bumps, keeping VSS strap width wide, and using many via connections between metal layers. Using ground shields between signal traces provides additional return current paths with low inductance. Staggering driver switching through slew rate control or skewed clock phases reduces the number of simultaneous transitions. Placing decoupling capacitors close to the drivers provides local current sourcing that reduces the current drawn through the inductive package path.

For layout, the most effective mitigation is ensuring that every VDDQ driver has a low-inductance return current path to VSS. This means placing VSS bumps adjacent to VDDQ bumps, running VSS straps parallel and adjacent to VDDQ straps on every metal layer, and providing multiple via stacks from the driver's source connection to the VSS bumps.

---

### Q9. How does the power grid differ between the IO region and the digital logic region of the PHY?

**Answer:**

The IO region and the digital logic region of the PHY have fundamentally different power grid requirements, reflecting their different current profiles and voltage domains.

The IO region operates on VDDQ (0.5V) and has high transient current demands (driver switching, ODT). The current is concentrated in a narrow strip along the die edge where the IO cells are located. The power grid must handle peak currents of 100-200 mA per byte lane with very low impedance (less than 125 mohm). The metal straps must be wide and dense, with multiple layers dedicated to VDDQ and VSS. The decoupling must be aggressive (hundreds of picofarads per byte lane).

The digital logic region (serialisers, FIFOs, training logic) operates on the core voltage domain (0.5-0.8V) and has moderate, more uniformly distributed current demands. The current profile is dominated by clock toggling and data switching in the FIFOs, with lower peak-to-average ratio than the IO region. The power grid can use the standard digital backend power grid methodology (power stripes, follow pins, via arrays) with moderate metal widths.

The transition between domains requires level shifter cells that tap into both power grids. At the boundary, both the VDDQ and core power grids must have adequate supply, creating a region where two power grids overlap. The layout must ensure that neither grid has a weak point at this boundary.

The VDD2 domain (powering the PLL and analog blocks) has the most stringent noise requirements but the lowest current demands. The VDD2 grid can be relatively sparse (fewer, narrower straps) but must be exceptionally clean, with dedicated decoupling and isolation from the noisy VDDQ and core domains. The VDD2 grid may use a dedicated on-chip LDO (Low Dropout Regulator) rather than direct connection to package bumps, providing additional filtering.

---

### Q10. How is the power grid verified for electromigration reliability?

**Answer:**

Electromigration (EM) is the gradual displacement of metal atoms caused by high current density in interconnect wires. Over time, EM can cause voids (open circuits) or hillocks (short circuits) in the metal, leading to circuit failure. The power grid, which carries the highest sustained currents in the LPDDR PHY, is the most susceptible to EM.

EM verification checks that the current density in every metal segment and via does not exceed the technology-specified limits. These limits depend on the metal width, metal layer, temperature, and the expected product lifetime (typically 10 years for consumer electronics, 15-20 years for automotive).

For LPDDR PHY power grids, EM limits are particularly challenging because the current density in the IO region is high. A 10 um wide metal strap carrying 50 mA has a current density of 50 mA / (10 um x 0.1 um thickness) = 5 x 10^4 A/cm^2, which approaches the EM limit for many metal layers (typically 1-10 x 10^4 A/cm^2 for DC current).

The verification flow uses the same extracted power grid model as IR drop analysis, but instead of checking voltage drops, it checks current density in every element. The current loading model must include worst-case sustained current (not just peak transient current, as EM is a long-term effect driven by average current).

For signal routing (DQ, DQS, CA), EM is typically less critical because signal wires carry AC current with zero DC component (the average current over time is zero for random data). However, clock signals (CK, WCK) carry AC current with a non-zero RMS value, and some processes define AC EM limits that are less conservative than DC limits.

When EM violations are found, the layout must be fixed by widening the violating metal straps, adding parallel straps to share the current, adding more vias (each via has an EM limit, and multiple parallel vias share the current), or rerouting to use metal layers with higher EM limits (thicker metal layers have higher current capacity).

---

See also:
- [Floorplanning for LPDDR](floorplanning_for_lpddr.md)
- [Bump and Ball Map Design](bump_and_ball_map_design.md)
- [Power Integrity for LPDDR](../05_signal_and_power_integrity/power_integrity_for_lpddr.md)
- [Worked Problem: IO Power Grid Design](worked_problems/problem_03_io_power_grid_design.md)
