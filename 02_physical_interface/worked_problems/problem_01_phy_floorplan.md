# Worked Problem: PHY Floorplan

## Problem Statement

You are tasked with creating an initial floorplan for a 4-channel LPDDR5 PHY (x32 interface) on a 5nm SoC. The PHY must interface with a memory controller located centrally on the die. The DRAM PoP package is mounted on the top of the SoC package, with the ball map oriented along the top edge of the die. The following constraints apply:

- Each byte lane (8 DQ + 1 DQS) IO cell block is 300 um wide x 80 um deep
- Each CA lane (7 CA + CK + WCK + CS) IO cell block is 400 um wide x 80 um deep
- The PLL block is 150 um x 150 um and requires 50 um keep-out on all sides
- The ZQ calibration block is 100 um x 80 um
- Bump pitch is 0.4 mm (400 um)
- The memory controller interface (DFI) is 200 um wide and located at the bottom of the PHY area

Determine the total PHY width, depth, and placement strategy.

---

## Worked Solution

### Step 1: Identify all blocks per channel

Each LPDDR5 channel (8-bit) requires:
- 1 byte lane IO cell block: 300 um x 80 um
- 1 CA lane IO cell block: 400 um x 80 um
- 1 serialiser/deserialiser block (estimated): 300 um x 100 um
- 1 FIFO block (estimated): 200 um x 80 um
- 1 per-channel training logic: 150 um x 60 um

For 4 channels total:
- 4 byte lane blocks
- 4 CA lane blocks
- 4 serialiser blocks
- 4 FIFO blocks
- 4 training blocks

Shared blocks:
- 1 PLL: 150 um x 150 um (+ 50 um keep-out = 250 um x 250 um effective)
- 1 ZQ calibration: 100 um x 80 um
- 1 DFI interface block: 200 um x 100 um (estimated)

### Step 2: Determine the die-edge layout width

The IO cells must align with the bump map. With 0.4 mm (400 um) bump pitch, we need to allocate bumps for:

Per channel: 8 DQ + 2 DQS (differential) + 2 WCK (differential) + 7 CA + 2 CK (differential) + 1 CS = 22 signal bumps. Plus approximately 22 power/ground bumps (roughly 1:1 signal to power ratio).

Total bumps per channel: ~44 bumps
4 channels: ~176 bumps

At 400 um pitch in a grid, if arranged in 6 rows: 176/6 = ~30 columns.
Total width: 30 x 400 um = 12,000 um = 12 mm

This is too wide. In practice, the bump map is optimised with higher density. Let us assume the JEDEC ball map fits within an 8 mm x 4 mm area for the LPDDR interface region.

### Step 3: Arrange the channel blocks along the die edge

A practical arrangement groups channels in pairs:

```
|<--- Channel 0+1 (left) --->|<-- PLL -->|<--- Channel 2+3 (right) --->|
|  CA0 | Byte0 | CA1 | Byte1 |   PLL     | CA2 | Byte2 | CA3 | Byte3  |
|  400 |  300  | 400 |  300  | 250(eff)  | 400 |  300  | 400 |  300   |
```

Total width = 4 x 400 + 4 x 300 + 250 = 1600 + 1200 + 250 = 3050 um

Add spacing between blocks (50 um each, 9 gaps): 450 um
Add ZQ block (100 um) at one end: 100 um

**Total PHY width along die edge: approximately 3600 um (3.6 mm)**

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

2. **PLL centrally placed:** The PLL is placed at the centre of the PHY width, between channels 1 and 2. This minimises clock distribution skew to the outermost channels. The PLL keep-out zone provides noise isolation.

3. **Serialisers behind IO cells:** Each channel's serialiser/deserialiser is placed directly behind its IO cells, minimising the routing distance for the serialised data.

4. **FIFOs in the middle tier:** FIFOs are placed behind the serialisers, bridging between the high-speed IO domain and the slower DFI domain.

5. **ZQ at the edge:** The ZQ calibration block is placed at one end of the PHY, near a dedicated ZQ bump, with code distribution routing running across the full width.

6. **DFI at the bottom:** The DFI interface faces the core logic area, providing a clean interface to the memory controller.

### Step 6: Verify bump alignment

With 3.6 mm PHY width and 400 um bump pitch, approximately 9 bump columns fit across the PHY width. With 6 rows of bumps, that is 54 bumps per channel pair, or 27 per channel. This is sufficient for the estimated 22 signal + 22 power bumps if the rows extend deeper (more rows per channel).

**Final PHY dimensions: approximately 3.6 mm wide x 0.45 mm deep, total area approximately 1.6 mm^2 for the 4-channel PHY.**

---

See also:
- [PHY Architecture](../phy_architecture.md)
- [Floorplanning for LPDDR](../../03_layout_fundamentals/floorplanning_for_lpddr.md)
