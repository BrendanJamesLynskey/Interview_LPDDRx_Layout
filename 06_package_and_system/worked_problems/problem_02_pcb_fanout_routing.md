# Worked Problem: PCB Fanout Routing

## Problem Statement

Design the BGA fanout routing for an LPDDR5 SoC package on a PCB. The SoC BGA has:
- Body size: 14 mm x 14 mm
- Ball pitch: 0.5 mm
- Ball count: ~780 (28 x 28 grid minus corners)
- LPDDR signals on one side (7 rows x 28 columns = 196 balls)
- PCB stackup: 8 layers, FR-4, 1.2 mm total thickness
- Target impedance: 50 ohm single-ended

Determine the fanout strategy for the LPDDR signal region.

---

## Worked Solution

### Step 1: Analyse the ball field

The LPDDR region occupies 7 rows x 28 columns = 196 ball positions at 0.5 mm pitch.
- Width: 28 x 0.5 = 14 mm
- Depth: 7 x 0.5 = 3.5 mm

Of the 196 positions, approximately 96 are LPDDR signals and 100 are power/ground (based on ~1:1 signal-to-power ratio).

### Step 2: Determine routing layers

With 0.5 mm ball pitch, the space between ball pads is:
- Ball pad diameter: ~0.3 mm (300 um)
- Space between pads: 0.5 - 0.3 = 0.2 mm (200 um)
- Available routing width between pads: 200 um minus clearances (~50 um each side) = 100 um

At 100 um available width, one trace (75-100 um wide for 50 ohm) can pass between adjacent pads. This means the outer 2 rows can be fanned out on the top layer (Layer 1). The next 2 rows require Layer 3 (via through Layer 2 ground). The inner 3 rows require Layer 5 or deeper.

### Step 3: Design the fanout per row

```
Row 1 (outermost): Direct fanout on L1 (top copper)
  - Traces route outward from balls to the routing channels
  - No vias needed for signal escape
  
Row 2: Fanout on L1 between Row 1 balls
  - One trace passes between each pair of Row 1 balls
  - Traces route outward to routing channels

Row 3: Via to L3, fanout on L3
  - Via from ball pad through L2 (ground) to L3 (signal)
  - Route on L3 to escape the ball field

Row 4: Via to L3, fanout on L3
  - Shares L3 routing with Row 3

Row 5: Via to L5, fanout on L5
  - Via through L2, L3, L4 to L5 (signal)

Row 6: Via to L5, fanout on L5
  - Shares L5 routing with Row 5

Row 7 (innermost): Via to L5 or L7, fanout on deeper layer
```

### Step 4: Calculate via parasitics

Through-hole via from L1 to L3 (2 layers, ~0.3 mm depth):
- Via stub below L3: 1.2 - 0.3 = 0.9 mm
- Stub resonance: f = c / (4 * L * sqrt(Er)) = 3e8 / (4 * 0.9e-3 * sqrt(4.2)) = ~41 GHz
- Acceptable for LPDDR5 (well above 3.2 GHz Nyquist)

For LPDDR5X (4.27 GHz Nyquist), the 5th harmonic (21.3 GHz) is still below stub resonance. Acceptable.

### Step 5: Length matching in the fanout region

The fanout creates inherent length mismatches:
- Row 1 signal: 0.5 mm escape route
- Row 7 signal: 3.5 mm escape route (must traverse through 6 rows of balls)

Delta: 3.0 mm. At ~7 ps/mm PCB propagation: 21 ps of skew.

For intra-byte matching, all 8 DQ should be assigned to the same 2-3 rows to minimise the fanout-induced skew. DQS should be in the same row as the DQ bits it serves.

### Step 6: Recommendations

1. Assign all DQ/DQS signals within a byte to adjacent rows (maximum 2 rows apart) to limit fanout skew to less than 0.5 mm (3.5 ps).
2. Place CK/WCK in the same rows as their associated CA signals.
3. Use via-in-pad (if budget allows) to eliminate dog-bone stubs.
4. Consider back-drilling for LPDDR5X to reduce stub length on inner vias.
5. Add serpentine on the shorter routes (outer rows) to match the longer inner row routes.

---

See also:
- [PCB Routing for LPDDR](../pcb_routing_for_lpddr.md)
