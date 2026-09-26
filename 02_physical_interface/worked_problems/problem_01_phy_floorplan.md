# Worked Problem: PHY Floorplan

## Problem Statement

You are tasked with creating an initial floorplan for an LPDDR5 PHY with an x32 interface (2 channels × 16 bits) on a 5nm SoC. The PHY must interface with a memory controller located centrally on the die. The DRAM PoP package is mounted on the top of the SoC package, with the ball map oriented along the top edge of the die. The following constraints apply:

- Each byte lane (8 DQ + DMI + RDQS pair + WCK pair) IO cell block is 300 um wide x 80 um deep
- Each CA lane (7 CA + CK pair + CS) IO cell block is 400 um wide x 80 um deep
- The PLL block is 150 um x 150 um and requires 50 um keep-out on all sides
- The ZQ calibration block is 100 um x 80 um
- Bump pitch is 0.4 mm (400 um)
- The memory controller interface (DFI) is 200 um wide and located at the bottom of the PHY area

Determine the total PHY width, depth, and placement strategy.

---

## Worked Solution

### Step 1: Identify all blocks per channel

Each LPDDR5 channel is 16 bits wide, so it has two byte lanes that share one CA bus. Each channel requires:
- 2 byte lane IO cell blocks: 300 um x 80 um each
- 1 CA lane IO cell block: 400 um x 80 um
- 2 serialiser/deserialiser blocks, one behind each byte lane (estimated): 300 um x 100 um
- 1 FIFO block (estimated): 200 um x 80 um
- 1 per-channel training logic: 150 um x 60 um

For 2 channels total:
- 4 byte lane blocks
- 2 CA lane blocks
- 4 serialiser blocks
- 2 FIFO blocks
- 2 training blocks

Shared blocks:
- 1 PLL: 150 um x 150 um (+ 50 um keep-out = 250 um x 250 um effective)
- 1 ZQ calibration: 100 um x 80 um
- 1 DFI interface block: 200 um x 100 um (estimated)

### Step 2: Determine the die-edge layout width

The IO cells must align with the bump map. With 0.4 mm (400 um) bump pitch, we need to allocate bumps for:

Per channel: 16 DQ + 2 DMI + 4 RDQS (two differential pairs) + 4 WCK (two differential pairs) + 7 CA + 2 CK (differential) + 1 CS = 36 signal bumps. Plus approximately 36 power/ground bumps (roughly 1:1 signal to power ratio).

Total bumps per channel: ~72 bumps
2 channels: ~144 bumps

At 400 um pitch in a grid, if arranged in 6 rows: 144/6 = 24 columns.
Total width: 24 x 400 um = 9,600 um = 9.6 mm

This is too wide. In practice, the bump map is optimised with higher density. Let us assume the JEDEC ball map fits within an 8 mm x 4 mm area for the LPDDR interface region (20 x 10 = 200 sites at 0.4 mm, enough for 144).

### Step 3: Arrange the channel blocks along the die edge

A practical arrangement puts each channel's CA block between its two byte lanes, so CK/CA reach both bytes of the channel over similar distances:

```
|<------ Channel A (left) ------>|<-- PLL -->|<------ Channel B (right) ----->|
| Byte0A | CA_A | Byte1A |   PLL     | Byte0B | CA_B | Byte1B |
|  300   | 400  |  300   | 250(eff)  |  300   | 400  |  300   |
```

Total width = 4 x 300 + 2 x 400 + 250 = 1200 + 800 + 250 = 2250 um

Add spacing between blocks (50 um each, 7 gaps including the ZQ block): 350 um
Add ZQ block (100 um) at one end: 100 um

**Total PHY width along die edge: 2250 + 350 + 100 = 2700 um (2.7 mm)**

### Step 4: Determine the PHY depth

The PHY depth is layered from die edge inward:

```
Layer 1 (die edge): IO cells           = 80 um
Layer 2: Serialisers/Deserialisers     = 100 um
Layer 3: FIFOs and training logic      = 80 um
Layer 4: DFI interface                 = 100 um
Layer 5: PLL (extends through layers 1-3) = accounted for above
Spacing between layers:                 = 3 x 30 um = 90 um
```

**Total PHY depth: approximately 450 um**

### Step 5: Placement strategy

1. **IO cells at die edge:** All byte lane and CA lane IO cells are placed along the top die edge, aligned with the corresponding bump columns.

2. **PLL centrally placed:** The PLL is placed at the centre of the PHY width, between channels A and B. This minimises clock distribution skew to the outermost byte lanes. The PLL keep-out zone provides noise isolation.

3. **Serialisers behind IO cells:** Each byte lane's serialiser/deserialiser is placed directly behind its IO cells, minimising the routing distance for the serialised data.

4. **FIFOs in the middle tier:** FIFOs are placed behind the serialisers, bridging between the high-speed IO domain and the slower DFI domain.

5. **ZQ at the edge:** The ZQ calibration block is placed at one end of the PHY, near a dedicated ZQ bump, with code distribution routing running across the full width.

6. **DFI at the bottom:** The DFI interface faces the core logic area, providing a clean interface to the memory controller.

### Step 6: Verify bump alignment

With a 2.7 mm PHY width and 400 um bump pitch, 6 bump columns fit across the PHY (2700 / 400 = 6.75). With 6 rows that is 36 bump sites over the whole PHY, but the x32 interface needs about 144 bumps (Step 2). Only 36 / 144 = 25 % of them can sit directly over the IO cells.

At 0.4 mm pitch the interface is therefore **bump-limited, not IO-cell-limited**. There are two ways to resolve it:
- Fan the bump field out beyond the PHY footprint through the package redistribution layers, into the 8 mm x 4 mm region assumed in Step 2. The cost is longer, less matched escape routes.
- Use a finer die-bump pitch. To fit all 144 bumps in 6 rows across the 2.7 mm width takes 24 columns, which is a pitch of 2700 / 24 ≈ 112 um.

**Final PHY dimensions: approximately 2.7 mm wide x 0.45 mm deep, total area approximately 1.2 mm^2 for the 2-channel (x32) PHY.**

---

See also:
- [PHY Architecture](../phy_architecture.md)
- [Floorplanning for LPDDR](../../03_layout_fundamentals/floorplanning_for_lpddr.md)
