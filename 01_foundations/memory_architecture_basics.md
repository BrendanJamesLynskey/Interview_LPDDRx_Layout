# Memory Architecture Basics

This section covers the fundamental architectural concepts of LPDDR memory, including channels, ranks, banks, bank groups, prefetch length, and burst length. Understanding these concepts is essential for layout engineers because the physical implementation must reflect and support the logical architecture.

---

### Q1. What is the hierarchical organisation of an LPDDR memory system, from channel down to cell?

**Answer:**

An LPDDR memory system is organised as a hierarchy of structures, each with distinct physical and logical properties. At the top level, the interface is divided into channels. In LPDDR4, each die exposes two 16-bit channels, while in LPDDR5, each die exposes two 8-bit channels (resulting in four 8-bit channels for a typical x32 system using two dies).

Within each channel, there may be one or more ranks. A rank represents a complete set of DRAM arrays that respond to a single chip-select (CS) signal. In mobile systems, a single rank per channel is the most common configuration, though dual-rank configurations exist for higher-density applications.

Each rank contains multiple banks. LPDDR4 defines 8 banks per channel, organised into no bank groups. LPDDR5 introduces bank groups: typically 4 bank groups with 4 banks per group, totalling 16 banks per channel. Bank groups are important because accesses to different bank groups can be pipelined more efficiently than accesses within the same bank group, improving effective bandwidth.

Each bank consists of rows and columns of memory cells. A row is activated (opened) by the ACT command, which reads the entire row into the sense amplifiers (row buffer). Subsequent read or write commands access columns within the open row. The row size is typically 1KB or 2KB in LPDDR5.

At the lowest level, each memory cell consists of a single transistor and a capacitor (1T1C), storing one bit of data. The physical layout of the cell array uses specialised DRAM process technology with deep-trench or stacked capacitors that are fundamentally different from the logic process used for the SoC.

---

### Q2. How do channels map to physical pins and layout resources in an LPDDR5 interface?

**Answer:**

In LPDDR5, each 8-bit channel has a dedicated set of physical pins: 8 DQ (data) pins, 1 DQS/DQS_n differential strobe pair, 1 WCK/WCK_n differential write clock pair, 7 CA (command/address) pins, 1 CK/CK_n differential system clock pair, 1 CS_n chip select, and associated power and ground pins. This means each channel requires approximately 24 signal pins plus power/ground.

For a typical x32 LPDDR5 system using two dies, there are four independent 8-bit channels. The total signal pin count is approximately 96 signal pins plus substantial power and ground pins. On the SoC side, each channel corresponds to a byte lane in the PHY, and each byte lane has its own set of IO cells, serialisers/deserialisers, timing calibration circuits, and clock distribution.

From a layout perspective, the four channels must be placed so that their IO cells align with the corresponding bumps on the package. A common floorplan approach places two channels on one side of the memory controller and two on the other, creating a symmetric layout. Alternatively, all four channels may be placed along one edge of the die if the DRAM is located in a specific direction relative to the SoC.

The key physical resources per channel include the DQ byte lane (IO cells for 8 DQ pins, 1 DQS pair), the CA lane (IO cells for 7 CA pins, 1 CK pair), the WCK driver and distribution, ZQ calibration circuitry (typically shared across channels), and the FIFO and training logic. The total area of a single LPDDR5 channel PHY is typically 0.5-1.5 mm^2 in advanced process nodes (5nm-7nm), depending on the specific implementation.

---

### Q3. What is the difference between bank groups and banks, and why were bank groups introduced in LPDDR5?

**Answer:**

Banks are independent memory arrays within a single channel that can be in different states simultaneously -- one bank can be activating a row while another is performing a read or write. This bank-level parallelism is fundamental to achieving high bandwidth because it hides the latency of row activation and precharge operations.

Bank groups are a higher-level grouping introduced in LPDDR5 (borrowed from DDR4/DDR5). In LPDDR5, a typical configuration has 4 bank groups (BG0-BG3), each containing 4 banks, for a total of 16 banks per channel. The key advantage of bank groups is that accesses to different bank groups have shorter timing constraints than accesses within the same bank group. Specifically, the column-to-column delay for different bank groups (tCCD_L) is shorter than for banks within the same group, allowing the memory controller to interleave commands more efficiently.

From a layout perspective, bank groups primarily affect the memory controller design rather than the PHY layout directly. However, the memory controller's command scheduling logic must be aware of bank group boundaries to maximise bandwidth utilisation, and the CA bus timing must be designed to support the higher command rates enabled by bank group interleaving. The layout engineer must ensure that the CA signals can toggle at the required rate without excessive ISI (inter-symbol interference) or timing violations.

Bank groups also impact power consumption patterns. When the memory controller exploits bank group interleaving aggressively, the switching activity on the DQ bus becomes more continuous (fewer idle cycles), which increases the sustained power draw on the VDDQ supply. The power grid must be designed to handle this worst-case sustained switching scenario without excessive IR drop or voltage droop.

---

### Q4. What is the prefetch architecture, and how does it bridge the speed gap between the DRAM array and the IO interface?

**Answer:**

The DRAM core array operates at a much lower frequency than the IO interface because the fundamental cell access time (tRAS) is limited by the physics of charging and discharging the tiny storage capacitors through the access transistors. In LPDDR5 operating at 6400 MT/s, the IO toggles at 3200 MHz (DDR), but the core array operates at only 200 MHz. The prefetch architecture bridges this gap by fetching multiple bits from the array in parallel and serialising them onto the IO pins.

LPDDR5 uses a 16n prefetch, meaning 16 bits are fetched from the array for each DQ pin in a single internal access. These 16 bits are loaded into a buffer and then serialised onto the pin at the IO rate over 8 UI (Unit Intervals, where each UI is one half of a clock cycle at the data rate). With double data rate signaling, 16 bits are transmitted in 8 clock cycles, which corresponds to a burst length (BL) of 16. LPDDR5 also supports BL32, which fetches 32 bits per pin and transmits them over 16 clock cycles.

Inside the PHY, the prefetch architecture manifests as a wide parallel data path between the memory controller and the serialiser. For a single 8-bit channel with 16n prefetch, the internal data width is 8 DQ x 16 = 128 bits. This 128-bit bus must be routed from the memory controller through the PHY to the serialiser, which is located near the IO cells. The layout must accommodate this wide bus without creating congestion, and the timing of the parallel data must be managed to ensure reliable capture by the serialiser.

The serialiser typically uses a multi-phase clock derived from the PLL/DLL to select which of the 16 prefetched bits is driven onto the pin at each UI. The clock distribution for this multi-phase serialisation must be low-skew and low-jitter, as any timing error directly narrows the data eye at the receiver.

---

### Q5. How does the LPDDR5 addressing scheme work, and what are the implications for the CA bus?

**Answer:**

LPDDR5 uses a multiplexed Command/Address (CA) bus that carries both commands and addresses on the same set of pins. The CA bus is 7 bits wide per channel, and information is transmitted on both edges of the CK clock (double data rate on the CA bus). A command is issued over one or two clock cycles, with the command opcode and address fields packed into the available CA bits.

The basic command encoding uses two cycles: the first cycle (rising CK edge) carries the command opcode and part of the address, and the second cycle carries the remaining address bits. Some commands, such as MRW (Mode Register Write), require additional cycles. The CS_n (chip select) signal is asserted to indicate that valid command information is present on the CA bus.

For layout engineers, the double data rate CA bus means that the CA signals must meet the same signal integrity requirements as the data signals. The CA-to-CK timing must be tightly controlled, with setup and hold times that define the skew budget between CA pins and the CK clock. Typically, the CA-to-CK skew must be within a few hundred picoseconds, requiring length matching between all CA signals and the CK pair within a channel.

The CA bus operates at the CK frequency (half the data rate), so at 6400 MT/s, the CA bus toggles at 1600 MHz -- still a very high frequency that requires controlled-impedance routing, matched lengths, and proper termination. LPDDR5 introduced CA-ODT (on-die termination for the CA bus) to improve signal integrity at these speeds.

The 7-bit CA bus width per channel is relatively narrow, which simplifies routing compared to the wider address buses in DDR systems. However, the double data rate operation means that each CA bit carries more information per cycle, so timing violations on any single CA bit can corrupt the entire command.

---

### Q6. What is the role of the mode register in LPDDR devices, and how does it relate to physical design?

**Answer:**

LPDDR devices contain a set of Mode Registers (MR0 through MR63 in LPDDR5) that configure the operating parameters of the memory. These registers are written by the memory controller using MRW (Mode Register Write) commands sent over the CA bus and read using MRR (Mode Register Read) commands. The mode registers control critical parameters such as burst length, CAS latency, write latency, ODT values, drive strength, DQ VREF, CA VREF, and training modes.

From a physical design perspective, mode registers are relevant because they determine the electrical characteristics that the layout must support. For example, the ODT value programmed into the mode register determines the impedance that the IO cell presents, which affects signal reflections and termination. The drive strength setting determines the output impedance of the driver, which must be matched to the transmission line impedance for clean signaling. The VREF settings establish the voltage threshold for single-ended signal receivers, and any noise on the VREF distribution will directly affect timing margins.

The PHY must include logic to generate MRW commands during initialisation and training, and the mode register values must be stored in the SoC (typically in the memory controller's register file). The training algorithms iteratively adjust mode register values (such as VREF) while monitoring the bit error rate to find optimal operating points.

For layout, the VREF distribution is particularly important. LPDDR5 uses per-bit VREF training, meaning each DQ bit can have a slightly different optimal VREF. The VREF generator in the PHY must be capable of fine-grained adjustment, and the VREF distribution from the generator to each receiver must be clean and isolated from digital switching noise. This is typically achieved by routing VREF on dedicated metal layers with shielding.

---

### Q7. How does the refresh architecture in LPDDR5 differ from LPDDR4, and what are the system-level implications?

**Answer:**

DRAM cells lose their charge over time and must be periodically refreshed to maintain data integrity. LPDDR4 uses an all-bank refresh (REFab) that refreshes all banks simultaneously, pausing all access to the channel for the refresh duration (tRFC, typically 130-280ns depending on density). LPDDR4 also supports per-bank refresh (REFpb), which refreshes one bank at a time while other banks remain accessible, but this requires more frequent refresh commands.

LPDDR5 extends the refresh architecture with several improvements. It maintains both all-bank and per-bank refresh modes and adds support for different refresh rates based on temperature. At lower temperatures, the refresh interval can be extended (reducing refresh overhead), while at higher temperatures, more frequent refresh is needed. LPDDR5 also introduces self-refresh with temperature-controlled refresh rate, allowing the DRAM to autonomously adjust its refresh rate in low-power states.

For system-level physical design, refresh impacts bandwidth availability and power consumption. During refresh, the channel is unavailable for data transfers, creating periodic bandwidth holes. The memory controller must schedule refresh commands to minimise impact on latency-sensitive traffic. The refresh current spike (all banks activating simultaneously in REFab) creates a transient current demand on the power supply, which must be handled by the decoupling network.

For layout engineers, the refresh current transient is important for power grid design. A REFab command can draw several hundred milliamps of transient current over a few nanoseconds, causing voltage droop on VDD1 and VDD2. The power grid must have sufficient decoupling capacitance and low enough impedance to keep the voltage drop within the DRAM's operating margin. The placement of decoupling capacitors (MIM caps on-die, or embedded capacitors in the package) must be optimised to respond to these transients.

---

### Q8. What is the concept of rank in LPDDR, and how does dual-rank affect layout?

**Answer:**

A rank is a collection of DRAM arrays that respond to a single chip-select (CS) signal and share the same data bus. In a single-rank LPDDR configuration, one set of DRAM arrays is connected to each channel. In a dual-rank configuration, two independent sets of DRAM arrays share the same data bus but are selected by different CS signals. Only one rank can drive the data bus at any time, but the other rank can be performing internal operations (precharge, refresh) in parallel.

Dual-rank configurations double the memory capacity without increasing the data bus width. They are used in higher-end mobile devices and automotive applications that require more memory. The performance benefit comes from rank interleaving: the memory controller can issue commands to one rank while the other is busy with internal operations, hiding latency and improving effective bandwidth.

From a layout perspective, dual-rank LPDDR introduces additional complexity. The data bus is shared between ranks, so the IO cells and routing are unchanged for DQ/DQS. However, the CA bus, CK, and CS signals may need to be routed to both ranks, potentially requiring additional package routing or through-mold vias in a PoP stack. The additional CS signal requires an extra bump and IO cell.

The signal integrity implications of dual-rank are significant. When one rank is driving and the other is in high-impedance state, the unterminated input of the idle rank presents a capacitive stub on the data bus. This stub can cause reflections and degrade signal integrity, particularly at high data rates. The ODT settings become more complex in dual-rank: the active rank needs write termination, while the idle rank may need non-target termination (NT-ODT) to absorb reflections. The layout must ensure clean routing between the SoC and both ranks, minimising stub lengths and managing the additional capacitive loading.

---

### Q9. How does the burst length affect data throughput and PHY design?

**Answer:**

The burst length (BL) defines the number of data bits transferred per DQ pin for a single read or write command. In LPDDR4, the standard burst length is BL16, transferring 16 bits per DQ pin over 8 clock cycles (double data rate: 2 bits per cycle). In LPDDR5, both BL16 and BL32 are supported, with BL32 transferring 32 bits per pin over 16 clock cycles.

The burst length directly affects data throughput. For a single read command on an 8-bit LPDDR5 channel at 6400 MT/s with BL16: 8 pins x 16 bits = 128 bits = 16 bytes transferred in 8 clock cycles (2.5ns at 3200 MHz CK). With BL32, a single command delivers 32 bytes in 5ns. The choice of burst length affects the minimum access granularity and the efficiency of the interface for different workload patterns.

For PHY design, the burst length determines the depth of the serialiser/deserialiser and the FIFO buffers. A BL16 operation with 16n prefetch means the serialiser handles 16 bits per DQ pin, requiring a 16:1 multiplexer on the write side and a 1:16 demultiplexer on the read side. BL32 extends this to 32:1 and 1:32, or alternatively uses two consecutive BL16 operations internally.

The FIFO depth between the memory controller domain and the IO domain must accommodate the burst length plus margin for timing uncertainty. Typical FIFO depth is 4-8 entries of the burst width. The layout of these FIFOs affects the data path latency and must be optimised for both area and timing.

The burst length also affects the switching pattern on the DQ bus. Longer bursts create more sustained switching activity, which increases the cumulative IR drop on the VDDQ supply and the thermal load on the IO cells. The power grid must be designed for the worst-case sustained switching scenario (all DQ bits toggling for the full burst duration), which occurs during BL32 operations with a checkerboard data pattern.

---

### Q10. What is the relationship between the memory controller and the PHY, and where is the boundary in the layout?

**Answer:**

The memory controller and the PHY are the two main blocks that implement the LPDDR interface on the SoC side. The memory controller handles the high-level protocol: command scheduling, refresh management, bank state tracking, address mapping, reordering, and quality-of-service arbitration. The PHY handles the physical signaling: serialisation/deserialisation, timing alignment, impedance calibration, IO driving/receiving, and training.

The boundary between the controller and PHY is typically defined by a standard interface such as DFI (DDR PHY Interface), developed by the DFI consortium. The DFI interface consists of wide parallel data buses (carrying the prefetched data), command/address signals, and control signals for training and calibration. The DFI interface operates at a fraction of the IO data rate, typically at the memory controller clock frequency (1/4 or 1/8 of the data rate).

In the layout, the memory controller is placed in the core logic area and is synthesised and placed-and-routed using standard digital design flows. The PHY is a mixed-signal block that includes both digital logic (training FSMs, FIFOs, serialisers) and analog/custom circuits (IO cells, DLL/PLL, ZQ calibration, VREF generators). The PHY is typically implemented as a hard macro or a combination of hard IO cells with soft digital logic.

The physical boundary between the controller and PHY is critical for timing closure. The DFI interface runs at a high clock frequency (800 MHz or more for LPDDR5 at 6400 MT/s with 1:8 ratio), so the routing between the controller and PHY must meet setup and hold timing constraints. The layout engineer must minimise the wire delay on the DFI bus by placing the controller close to the PHY. In some designs, the DFI interface crosses voltage domains (the controller may operate at a lower core voltage), requiring level shifters at the boundary.

---

See also:
- [LPDDR Evolution and Standards](lpddr_evolution_and_standards.md)
- [LPDDR Signaling and Timing](lpddr_signaling_and_timing.md)
- [PHY Architecture](../02_physical_interface/phy_architecture.md)
- [Floorplanning for LPDDR](../03_layout_fundamentals/floorplanning_for_lpddr.md)
