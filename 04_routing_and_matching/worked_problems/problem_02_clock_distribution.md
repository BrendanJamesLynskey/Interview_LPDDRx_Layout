# Worked Problem: Clock Distribution

## Problem Statement

Design the CK clock distribution from a centrally placed PLL to 4 byte lanes arranged in a row along the die edge. The byte lanes are at positions X = 0, 1.0, 2.0, and 3.0 mm from the left edge. The PLL is at X = 1.5 mm (centred). The CK output from the PLL must reach each byte lane's CA IO cell with less than 10 ps of skew.

Given: propagation velocity on M10 = 1.2 x 10^8 m/s (~8.3 ps/mm), buffer delay = 30 ps per stage.

---

## Worked Solution

### Step 1: Calculate distances from PLL to each byte lane

| Byte Lane | X position (mm) | Distance from PLL (mm) | Distance (um) |
|---|---|---|---|
| BL0 | 0.0 | 1.5 | 1500 |
| BL1 | 1.0 | 0.5 | 500 |
| BL2 | 2.0 | 0.5 | 500 |
| BL3 | 3.0 | 1.5 | 1500 |

### Step 2: Calculate wire delays

| Byte Lane | Wire distance (um) | Wire delay (ps) |
|---|---|---|
| BL0 | 1500 | 1.5 mm x 8.3 = 12.45 ps |
| BL1 | 500 | 0.5 mm x 8.3 = 4.15 ps |
| BL2 | 500 | 0.5 mm x 8.3 = 4.15 ps |
| BL3 | 1500 | 1.5 mm x 8.3 = 12.45 ps |

Without any matching: max skew = 12.45 - 4.15 = 8.3 ps — inside the 10 ps target, but with almost no margin for buffer and process variation.

### Step 3: Design an H-tree distribution

An H-tree provides natural balancing:

```
PLL (1.5 mm)
  |
  +--- Left branch (to X=0.75 mm, midpoint of BL0 and BL1)
  |      |
  |      +--- BL0 (X=0.0, distance 750 um from midpoint)
  |      +--- BL1 (X=1.0, distance 250 um from midpoint)
  |
  +--- Right branch (to X=2.25 mm, midpoint of BL2 and BL3)
         |
         +--- BL2 (X=2.0, distance 250 um from midpoint)
         +--- BL3 (X=3.0, distance 750 um from midpoint)
```

### Step 4: Calculate H-tree delays with equalization

Root to left/right midpoint: 750 um each (PLL at 1.5 to midpoints at 0.75 and 2.25). Symmetric.

Left midpoint to BL0: 750 um
Left midpoint to BL1: 250 um -- mismatch of 500 um.

To equalize, add 500 um of serpentine to the BL1 branch. Similarly, add 500 um of serpentine to the BL2 branch.

After equalization:

| Path | Physical distance (um) | Wire delay (ps) | Buffer stages | Total delay (ps) |
|---|---|---|---|---|
| PLL to BL0 | 750 + 750 = 1500 | 12.45 | 2 | 12.45 + 60 = 72.45 |
| PLL to BL1 | 750 + 250 + 500(serp) = 1500 | 12.45 | 2 | 72.45 |
| PLL to BL2 | 750 + 250 + 500(serp) = 1500 | 12.45 | 2 | 72.45 |
| PLL to BL3 | 750 + 750 = 1500 | 12.45 | 2 | 72.45 |

### Step 5: Verify skew

All paths have identical wire length (1500 um) and buffer count (2 stages). The residual skew comes from process variation in metal width and buffer delay:

- Wire delay variation: plus or minus 3% of 12.45 ps = plus or minus 0.37 ps (and systematic variation largely cancels by H-tree symmetry)
- Buffer delay variation: plus or minus 5% per stage = plus or minus 1.5 ps per stage, 2 stages = plus or minus 2.1 ps (RSS)

Total estimated skew: sqrt(0.37^2 + 2.1^2) = approximately 2.1 ps. This meets the 10 ps target; buffer mismatch, not wire length, dominates.

### Step 6: Refine with tighter physical matching

To hold margin against the 10 ps target, the usual measures are:
1. Use same metal layer (M10) for all branches -- eliminates layer-to-layer variation
2. Route all branches in the same metal environment (same neighboring density) -- reduces systematic variation
3. Use matched buffer cells from the same standard cell library with common-centroid placement
4. Target residual random variation of plus or minus 5 ps per branch

With these measures, the achievable skew is approximately 5-8 ps, meeting the 10 ps requirement.

---

See also:
- [CA CK Routing](../ca_ck_routing.md)
- [PHY Architecture](../../02_physical_interface/phy_architecture.md)
