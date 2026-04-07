# Worked Problem: Byte Lane Routing

## Problem Statement

Route one LPDDR5 byte lane (DQ[7:0] + DQS/DQS_n) from the IO cells to the bump pads. The routing region is 300 um wide and 200 um long (from IO cells at the bottom to bumps at the top). The following constraints apply:

- Signal trace width: 1.0 um, spacing: 1.0 um (2 um pitch)
- DQS differential pair: 1.0 um width, 1.5 um intra-pair spacing, 2.0 um spacing to adjacent signals
- Routing layer: M8 (propagation velocity: 1.5 x 10^8 m/s, ~6.67 ps/um)
- Length matching target: all DQ within plus or minus 50 um of DQS length
- Maximum route length: 300 um (to keep delay reasonable)
- DQS placed centrally in the byte group

Determine the routing arrangement and calculate the expected intra-byte skew.

---

## Worked Solution

### Step 1: Determine the signal arrangement

Place DQS pair centrally, with 4 DQ on each side:

```
Left to right (at IO cell row):
DQ0 | DQ1 | DQ2 | DQ3 | DQS | DQS_n | DQ4 | DQ5 | DQ6 | DQ7
```

Total width: 8 DQ x 2 um pitch + 2 DQS x (1.0 + 1.5 + 2.0 um) = 16 + 9 = 25 um signal width.

With guard bands on each side (5 um each): 25 + 10 = 35 um. This fits within the 300 um routing region with ample margin.

### Step 2: Route all signals straight from IO cells to bumps

If the bumps are directly above the IO cells (ideal case), all signals route straight for 200 um:

```
DQ[0-7] route length: 200 um each
DQS route length: 200 um
DQS_n route length: 200 um
```

All lengths are equal -- no serpentine needed. Skew = 0.

### Step 3: Account for bump offset (realistic case)

In practice, the bumps are on a 400 um grid and may not align perfectly with the IO cells. Assume the bumps are offset as follows:

| Signal | IO cell X | Bump X | X offset | Route length (straight + jog) |
|---|---|---|---|---|
| DQ0 | 10 um | 0 um | -10 um | 200 + 10 = ~201 um |
| DQ1 | 12 um | 0 um | -12 um | 200 + 12 = ~202 um |
| DQ2 | 14 um | 20 um | +6 um | 200 + 6 = ~201 um |
| DQ3 | 16 um | 20 um | +4 um | 200 + 4 = ~200 um |
| DQS | 19 um | 20 um | +1 um | 200 + 1 = ~200 um (reference) |
| DQS_n | 21.5 um | 22 um | +0.5 um | 200 + 0.5 = ~200 um |
| DQ4 | 24 um | 40 um | +16 um | 200 + 16 = ~203 um |
| DQ5 | 26 um | 40 um | +14 um | 200 + 14 = ~202 um |
| DQ6 | 28 um | 60 um | +32 um | 200 + 32 = ~206 um |
| DQ7 | 30 um | 60 um | +30 um | 200 + 30 = ~205 um |

### Step 4: Calculate deviations and add serpentine

Reference length (DQS): 200 um. Deviations:

| Signal | Length | Delta from DQS | Serpentine needed |
|---|---|---|---|
| DQ0 | 201 | +1 | 0 (within tolerance) |
| DQ1 | 202 | +2 | 0 (within tolerance) |
| DQ2 | 201 | +1 | 0 |
| DQ3 | 200 | 0 | 0 |
| DQ4 | 203 | +3 | 0 |
| DQ5 | 202 | +2 | 0 |
| DQ6 | 206 | +6 | 0 (within 50 um) |
| DQ7 | 205 | +5 | 0 |

All deviations are within the plus or minus 50 um tolerance. No serpentine needed in this case.

### Step 5: Calculate expected skew

Maximum deviation: DQ6 at +6 um from DQS.
Delay per um: 6.67 ps/um.
Maximum skew: 6 x 6.67 = 40 ps.

This 40 ps is within the typical SoC on-die skew allocation of 50-60 ps. After per-bit deskew training, the residual skew would be less than 5 ps (one deskew step).

---

See also:
- [DQ DQS Routing](../dq_dqs_routing.md)
- [Length Matching and Skew](../length_matching_and_skew.md)
