# Worked Problem: IO Power Grid Design

## Problem Statement

Design the VDDQ power grid for one LPDDR5 byte lane (8 DQ + 1 DQS) with the following requirements:

- VDDQ = 0.5V, maximum IR drop = 15 mV (3%)
- Peak current per byte lane: 120 mA (8 DQ switching + ODT)
- IO cell block dimensions: 300 um wide x 80 um deep
- Available metal layers for power: M10 (0.4 um thick), M11 (0.8 um thick), M12 (2.0 um thick)
- Metal sheet resistance: M10 = 25 mohm/sq, M11 = 15 mohm/sq, M12 = 6 mohm/sq
- 4 VDDQ bumps and 5 VSS bumps available for this byte lane

Determine the power strap widths, pitch, and layer assignment to meet the IR drop target.

---

## Worked Solution

### Step 1: Calculate the resistance budget

```
R_budget = V_drop_max / I_peak = 15 mV / 120 mA = 125 mohm
```

This 125 mohm budget must cover the entire path from bump to IO cell, including the via stack and metal routing.

### Step 2: Estimate the via stack resistance

Typical via resistances (per via):
- Bump to M12: ~5 mohm
- M12 to M11 via: ~10 mohm
- M11 to M10 via: ~10 mohm
- M10 to IO cell internal: ~15 mohm

Total via stack: ~40 mohm per path.

With 4 VDDQ bumps in parallel, the effective via resistance is 40/4 = 10 mohm.

Remaining budget for metal routing: 125 - 10 = 115 mohm.

### Step 3: Design M12 (top metal) power straps

M12 is the thickest layer (2.0 um, 6 mohm/sq) and carries the most current. Use M12 for the main VDDQ buses running horizontally across the byte lane.

Target: 2 VDDQ straps on M12, spanning the full 300 um width.
Strap width: 10 um each.

Resistance of one M12 strap across 300 um:
```
R_M12 = Rsh x L / W = 6 mohm/sq x 300/10 = 180 mohm per strap
```

Two straps in parallel: 90 mohm.

But the current distribution is not uniform -- bumps are at specific locations. Assume bumps are evenly distributed, so the effective length from bump to the farthest IO cell is approximately 150 um (half the width).

```
R_M12_eff = 6 x 150/10 = 90 mohm per strap, two in parallel = 45 mohm
```

### Step 4: Design M11 (intermediate) power straps

M11 runs vertically (orthogonal to M12), distributing current from the M12 horizontal straps to the IO cells below.

Target: 4 VDDQ straps on M11, running the 80 um depth of the IO cell block.
Strap width: 4 um each.

```
R_M11 = 15 x 80/4 = 300 mohm per strap, 4 in parallel = 75 mohm
```

But the effective length from M12 connection to the IO cell is approximately 40 um (half depth):
```
R_M11_eff = 15 x 40/4 = 150 mohm per strap, 4 in parallel = 37.5 mohm
```

### Step 5: Calculate total IR drop

```
R_total = R_via_stack + R_M12_eff + R_M11_eff
        = 10 + 45 + 37.5
        = 92.5 mohm
```

```
V_drop = R_total x I_peak = 92.5 mohm x 120 mA = 11.1 mV
```

This is within the 15 mV budget with 3.9 mV margin. Acceptable.

### Step 6: Add M10 local distribution

M10 provides fine-grained distribution within the IO cells:
- 8 VDDQ straps on M10, 2 um wide, running 30 um within each IO cell
- R_M10 = 25 x 30/2 = 375 mohm per strap, 8 in parallel = 47 mohm
- Effective R_M10 = ~24 mohm (half-length approximation)

Revised total: 10 + 45 + 37.5 + 24 = 116.5 mohm
V_drop = 116.5 x 120 mA = 14.0 mV (within 15 mV budget, 1 mV margin)

### Step 7: Repeat for VSS

The VSS grid mirrors the VDDQ grid but with 5 bumps (one more). The additional bump reduces the via stack resistance, providing slightly better IR drop for VSS. The VSS straps are interleaved with VDDQ straps on each metal layer.

### Step 8: Final design summary

| Layer | Direction | VDDQ Straps | Width | VSS Straps | Width |
|---|---|---|---|---|---|
| M12 | Horizontal | 2 | 10 um | 3 | 10 um |
| M11 | Vertical | 4 | 4 um | 4 | 4 um |
| M10 | Horizontal | 8 | 2 um | 8 | 2 um |

Total metal area used for power: approximately 15% of available routing resources on these layers. The remaining 85% is available for signal routing.

---

See also:
- [Power Grid for Memory IO](../power_grid_for_memory_io.md)
- [Power Integrity for LPDDR](../../05_signal_and_power_integrity/power_integrity_for_lpddr.md)
