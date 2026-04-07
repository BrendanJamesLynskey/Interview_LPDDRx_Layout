# Floorplanning for LPDDR

This section covers the floorplanning methodology for LPDDR PHY blocks on an SoC die. Floorplanning is the first and most impactful step in the physical design flow, as it determines the overall arrangement of blocks that constrains all subsequent routing and timing optimisation.

---

### Q1. What are the key considerations when placing an LPDDR PHY on an SoC die?

**Answer:**

The placement of the LPDDR PHY on the SoC die is driven by several interrelated considerations. The bump map alignment is the most fundamental constraint: the PHY IO cells must be positioned so that the on-die signal pads align with the corresponding package bumps, which in turn must align with the DRAM ball map in the PoP stack. Any misalignment forces long redistribution routing in the package, adding delay and impedance discontinuities.

Proximity to the memory controller is important because the DFI interface between the controller and PHY operates at high frequency (800 MHz or more). Long routing between these blocks increases wire delay and makes timing closure difficult. The controller should be placed as close as possible to the PHY, typically directly behind (inward from the die edge) the PHY blocks.

Die edge availability must be considered because the PHY IO cells must be at the die periphery where package bumps are located. The designer must allocate sufficient die edge length for the PHY while leaving room for other IO (PCIe, USB, display, camera interfaces). The LPDDR PHY typically requires 2-5 mm of die edge.

Thermal considerations are relevant because the IO cells dissipate significant power (from driver switching and ODT). If the LPDDR PHY is placed adjacent to another high-power block (such as a GPU), the combined thermal load could create a hot spot that degrades performance. The floorplan should distribute high-power blocks to avoid thermal clustering.

Signal integrity isolation requires keeping the LPDDR PHY away from noisy blocks (such as high-speed SerDes or RF circuits) that could couple noise through the substrate or power grid. Conversely, the LPDDR PHY's switching noise should not disturb sensitive analog circuits nearby.

---

### Q2. How is the PHY oriented relative to the DRAM in a PoP configuration?

**Answer:**

In a Package-on-Package (PoP) configuration, the DRAM package sits on top of the SoC package. The DRAM's ball map faces downward, connecting to the SoC's top package surface through Through-Mold Vias (TMVs) or similar structures. The SoC die connects to the package substrate through flip-chip bumps.

The PHY orientation must account for the signal path from the SoC die bumps through the SoC package substrate, through the TMVs, through the DRAM package substrate, and finally to the DRAM die bumps. This path introduces a physical mapping between the SoC die coordinates and the DRAM ball coordinates, which may include a mirror, rotation, or offset.

In a typical PoP stack, the SoC die is flip-chip mounted (face down) on the SoC package substrate. The DRAM die is wire-bonded or flip-chip mounted (depending on the generation) on the DRAM package substrate. The TMVs connect the top surface of the SoC package to the bottom surface of the DRAM package, passing through the mold compound that encapsulates the SoC die.

The PHY designer must work with the package team to determine the exact coordinate mapping. A common configuration has the LPDDR PHY IO cells along the top edge of the SoC die (which becomes the bottom edge after flip-chip mounting, closest to the top of the SoC package where the TMVs emerge). The DRAM ball map is mirrored relative to the SoC bump map because the two packages face each other.

For layout, this means the PHY channel assignments must match the DRAM channel positions after accounting for the mirror. If channel 0 of the DRAM is on the left side of the DRAM ball map, it will appear on the right side of the SoC bump map (due to the mirror). The PHY floorplan must account for this reversal.

---

### Q3. What is channel abutment, and why is it important for LPDDR PHY layout?

**Answer:**

Channel abutment refers to the practice of placing adjacent channel PHY blocks so that their boundaries share common structures (power rails, well taps, guard rings) without wasted space between them. Proper abutment is critical for area efficiency and for maintaining consistent electrical characteristics across channels.

In a 4-channel LPDDR5 PHY, the four byte lanes and their associated CA lanes are placed side by side along the die edge. If the boundaries between channels are poorly designed, gaps appear that waste die area, create discontinuities in the power grid, and introduce asymmetries in the routing environment.

A well-designed abutment scheme ensures that power rails (VDDQ, VSS, VDD2) connect seamlessly across channel boundaries, maintaining low impedance. Well taps and substrate contacts are shared at the boundary, providing continuous latch-up protection. Guard rings between voltage domains continue across channel boundaries without breaks. The routing channels between channels are defined and consistent, allowing predictable interconnect between the PLL (which is shared) and individual channel blocks.

The abutment approach typically defines a fixed boundary cell that sits between channels, containing the shared well taps, power rail connections, and guard ring segments. This boundary cell is a fixed-height, variable-width block that is instantiated at each channel junction.

For the layout engineer, abutment must be considered from the earliest floorplanning stage. The channel blocks must be designed with compatible boundary definitions: the same power rail pitches, the same well orientations, and the same metal layer usage at the boundaries. If channels are designed independently and then forced together, mismatches at the boundaries create design rule violations and functional issues.

---

### Q4. How does the memory controller placement affect the overall floorplan?

**Answer:**

The memory controller is a large digital block (typically 0.5-2 mm^2 in advanced nodes) that handles command scheduling, data buffering, quality-of-service arbitration, and the DFI protocol. Its placement relative to the PHY directly affects the DFI interface timing and the overall die floorplan.

The ideal placement puts the controller directly behind the PHY, with the DFI interface boundary running parallel to the die edge. This minimises the DFI routing distance, which is critical because the DFI bus is wide (256-512 bits for write data, plus command/address and control signals) and operates at high frequency. A typical DFI data bus for a 4-channel LPDDR5 x32 interface is 512 bits wide (32 DQ x 16 prefetch), plus approximately 128 bits of command/address/control signals, totalling roughly 640 signal wires. This bus must traverse from the controller to the PHY with adequate timing margin.

If the controller cannot be placed directly behind the PHY (due to other blocks occupying that area), the DFI bus must be routed through the congested core logic area, consuming metal resources and adding delay. In extreme cases, repeaters may be needed on the DFI bus to meet timing, further increasing area and power.

Some SoC designs use multiple memory controllers (one per channel pair or one per channel), in which case each sub-controller can be placed behind its associated PHY channels. This distributed approach reduces DFI bus width per controller instance and shortens the routing distance, at the cost of increased controller area (duplication of common logic) and more complex inter-controller communication for coherency.

The memory controller also requires access to the system interconnect (bus fabric), which is typically located in the centre of the die. The controller must have routing paths to the fabric that do not interfere with the PHY signal routing. This is usually achieved by routing the fabric interface on different metal layers from the PHY signals.

---

### Q5. What are the floorplanning considerations for the PLL placement within the PHY?

**Answer:**

The PLL is an analog block that generates all high-frequency clocks for the PHY. Its placement within the PHY floorplan is driven by noise sensitivity, clock distribution symmetry, and physical isolation requirements.

Central placement is preferred so that the clock distribution paths to each byte lane are approximately equal in length. If the PLL is at one end of the PHY, the nearest byte lane might see 200 um of clock routing while the farthest sees 1500 um, creating a systematic skew that must be compensated by the clock tree buffers. Placing the PLL centrally reduces the maximum clock route length and the worst-case skew.

Noise isolation is critical because the PLL's VCO (Voltage-Controlled Oscillator) is sensitive to substrate noise and power supply noise. The PLL should be surrounded by a deep N-well guard ring to isolate it from substrate noise generated by the digital switching in the byte lanes. The PLL's power supplies (both analog VDD and VSS) should have dedicated bumps or be fed from a separate on-chip LDO regulator with filtering.

Keep-out zones around the PLL prevent digital logic from being placed too close. A typical keep-out is 30-50 um on all sides, which creates a dead zone in the PHY layout. The floorplan must account for this dead zone and use it for passive elements (decoupling capacitors) or routing channels rather than leaving it empty.

The PLL output requires careful routing to preserve signal quality. The high-frequency clock (which may be several GHz) should be routed on a dedicated metal layer with shielding on adjacent layers. The routing should avoid sharp bends and maintain consistent impedance.

In some PHY architectures, two PLLs are used: one for the left pair of channels and one for the right pair. This reduces the clock distribution distance and provides redundancy, at the cost of increased area and the need to synchronise the two PLLs.

---

### Q6. How do ESD structures and pad rings affect the PHY floorplan?

**Answer:**

ESD (Electrostatic Discharge) protection structures and the pad ring are physical elements at the die periphery that occupy area and impose constraints on the PHY floorplan.

The pad ring is the continuous ring of IO pads around the die perimeter. For LPDDR, the pad ring in the PHY region contains the DQ, DQS, CA, CK, WCK, and power/ground pads, each with its associated ESD structures. The pad ring width (from the die edge inward to the first row of core logic) is typically 60-120 um, depending on the process and the ESD requirements.

The ESD structures require power clamp cells distributed around the pad ring. These clamp cells sit between VDDQ and VSS and provide a low-impedance discharge path during ESD events. The clamp cells are relatively large (50-100 um^2 each) and must be placed at regular intervals (typically every 200-400 um) around the pad ring perimeter.

The pad ring also contains corner cells (at die corners) and fill cells (between IO pads) that complete the continuous power and ground ring. The fill cells provide additional decoupling capacitance and ensure that the ESD rail is continuous.

For the PHY floorplan, the pad ring determines the available area for IO cells. The IO cell depth (from pad to the first core logic row) must fit within the pad ring width. If the IO cells are deeper than the pad ring, they extend into the core area and may conflict with standard cell rows.

The power clamp cells within the pad ring can be leveraged for decoupling. By placing additional MIM or MOS capacitors in the clamp cells, the layout engineer can increase the local decoupling near the IO drivers without consuming additional area.

The transition from the pad ring to the core PHY logic (serialisers, FIFOs) must be managed carefully. The power domains change at this boundary (from VDDQ in the pad ring to core VDD in the logic area), requiring level shifters and isolation cells.

---

### Q7. How should the floorplan handle multiple LPDDR interfaces on the same die?

**Answer:**

Many mobile SoCs include two or more independent LPDDR interfaces to provide sufficient memory bandwidth. For example, a high-end SoC might have two x32 LPDDR5 interfaces, each with 4 channels, for a total of 8 channels and a combined bandwidth of 51.2 GB/s at 6400 MT/s.

The floorplan for multiple LPDDR interfaces must consider die edge allocation, since each LPDDR interface needs 2-5 mm of die edge. Two interfaces might be placed on opposite edges of the die (top and bottom), on adjacent edges (top and right), or on the same edge (if the die edge is long enough). The choice depends on the DRAM placement in the PoP stack and the package routing constraints.

Memory controller placement for multiple interfaces requires careful interconnect planning. Each interface has its own memory controller, which must be close to its PHY. If the two interfaces are on opposite die edges, the controllers are naturally separated, which simplifies the floorplan. If the interfaces are on the same edge, the controllers are adjacent and may share some infrastructure.

The system interconnect must provide equal bandwidth to both controllers. If one controller has a longer path to the bus fabric, it may have higher latency, creating a NUMA-like (Non-Uniform Memory Access) topology that complicates software optimisation.

Power grid planning must ensure that each LPDDR interface has adequate VDDQ supply without the two interfaces interfering with each other through the power grid. The VDDQ domain for each interface should be independently decoupled, though they may share the same package power pins (with appropriate on-die separation).

Thermal planning becomes more critical with multiple interfaces. If both interfaces are on the same die edge, the combined power density in that region may exceed thermal limits.

---

### Q8. What role do physical design tools play in LPDDR PHY floorplanning?

**Answer:**

LPDDR PHY floorplanning uses a combination of custom/analog design tools and digital place-and-route tools, reflecting the mixed-signal nature of the PHY.

Cadence Virtuoso is commonly used for the custom/analog portions of the PHY: IO cell layout, PLL layout, ZQ calibration block, and VREF generators. These blocks require transistor-level layout with manual or semi-automated placement, precise matching, and custom routing that standard digital tools cannot provide.

Cadence Innovus or Synopsys ICC2 (IC Compiler II) are used for the digital portions of the PHY: serialisers, FIFOs, training FSMs, and the DFI interface. These blocks are synthesised from RTL (using Cadence Genus or Synopsys Design Compiler) and then placed and routed using the digital backend tools. The floorplan is defined in these tools by specifying the block boundaries, pin locations, and placement constraints.

The integration of custom and digital blocks requires a hierarchical approach. The custom blocks are imported as hard macros (LEF/DEF abstracts) into the digital tool, which places standard cells around them. The top-level floorplan is typically managed in the digital tool, with the custom blocks placed at fixed locations along the die edge.

Timing analysis during floorplanning uses Cadence Tempus or Synopsys PrimeTime to verify that the block-level timing budgets are achievable. Early-stage timing analysis with estimated wire loads helps identify placement issues before detailed routing.

Power grid analysis tools (Cadence Voltus, Synopsys RedHawk, or ANSYS RedHawk-SC) are used during floorplanning to verify that the planned power grid can support the LPDDR PHY's current demands. This early analysis can identify the need for additional power bumps or wider power straps before they become difficult to add.

Signal integrity simulation tools (ANSYS HFSS, Cadence Sigrity, Keysight ADS) are used to model the package and interconnect during floorplanning, verifying that the planned signal routing topology can support the target data rate.

---

### Q9. How does the floorplan account for testability and debug access to the PHY?

**Answer:**

Testability and debug access are important floorplanning considerations that are sometimes overlooked in the initial planning stages but can cause significant rework if not addressed early.

JTAG and scan access are needed for manufacturing test. The PHY's digital portions (FIFOs, training logic) are typically included in the scan chain for stuck-at fault testing. The scan chain routing must traverse the PHY area, connecting to the scan insertion points. The floorplan must provide routing channels for the scan chain, which can be wide (several hundred signals for a full-scan PHY).

Built-in self-test (BIST) for the LPDDR interface requires a BIST controller that can generate memory access patterns and check responses without the memory controller. The BIST block is typically 0.1-0.3 mm^2 and must be placed with access to the DFI interface and a connection to the JTAG controller.

Eye scan capability allows the PHY to sweep its sampling point across the data eye and report pass/fail results, creating a shmoo plot that characterises the timing and voltage margins. The eye scan hardware (variable delay, variable VREF, error counter) is part of the byte lane and is included in the floorplan. The results must be readable via JTAG or a memory-mapped register interface.

Loopback modes allow the PHY to test itself without an external DRAM. In loopback, the output driver feeds directly back to the input receiver (on-die loopback) or through the package and back (package loopback). The floorplan must ensure that the loopback path does not create electrical issues (such as driver-receiver contention).

Debug signals may be brought out to dedicated debug pads or multiplexed onto existing IOs. The floorplan should anticipate the routing of debug signals from the PHY interior to accessible pads.

---

### Q10. What are the common floorplan mistakes for LPDDR PHY, and how can they be avoided?

**Answer:**

Several common floorplanning mistakes can cause significant issues in LPDDR PHY design.

Bump map misalignment occurs when the PHY IO cells are placed without precise coordination with the package team. If the on-die pad locations do not align with the package bump locations, the redistribution layer (RDL) routing in the package becomes long and irregular, adding parasitic delay and impedance mismatch. This is avoided by establishing the bump map early (using the JEDEC ball map as the starting point) and constraining the PHY floorplan to match.

Insufficient power grid occurs when the floorplan does not allocate enough space for VDDQ power straps and decoupling capacitors. The LPDDR PHY has high current density (the IO cells are concentrated in a small area), and the power grid must be designed for the worst-case current draw. This is avoided by performing early-stage IR drop analysis and reserving metal layers for power routing.

PLL noise coupling occurs when the PLL is placed too close to the byte lanes or other digital logic without adequate guard rings. The substrate noise from digital switching modulates the VCO, creating jitter on all PHY clocks. This is avoided by maintaining keep-out zones around the PLL and placing it in a quiet region with dedicated power supplies.

Asymmetric clock distribution occurs when the PLL is placed at one end of the PHY rather than centrally. This creates unequal clock paths to different byte lanes, requiring compensation that consumes delay line range and reduces margin. This is avoided by placing the PLL centrally or using dual PLLs.

Thermal hot spots occur when the LPDDR PHY is placed adjacent to other high-power blocks (GPU, CPU cluster) without considering the cumulative thermal impact. This is avoided by performing early thermal analysis and distributing high-power blocks across the die.

Inadequate DFI routing channels occur when the area between the PHY and the memory controller is over-committed to other logic, leaving insufficient space for the wide DFI bus. This is avoided by reserving a clear routing corridor between the PHY and controller during floorplanning.

---

See also:
- [Bump and Ball Map Design](bump_and_ball_map_design.md)
- [Power Grid for Memory IO](power_grid_for_memory_io.md)
- [PHY Architecture](../02_physical_interface/phy_architecture.md)
- [Worked Problem: PHY Placement Strategy](worked_problems/problem_01_phy_placement_strategy.md)
