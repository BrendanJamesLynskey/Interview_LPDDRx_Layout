# Worked Problem: Driver Sizing

## Problem Statement

Design the output driver for an LPDDR5 DQ pin with the following specifications:

- Target output impedance: 40 ohm (both pull-up and pull-down)
- VDDQ: 0.5V
- Process: 5nm FinFET
- NMOS effective Ron: 180 ohm-um (at VDDQ = 0.5V, typical corner)
- PMOS effective Ron: 360 ohm-um (at VDDQ = 0.5V, typical corner)
- Calibration range: plus or minus 30% of target impedance
- Calibration resolution: 5 binary-weighted legs

Calculate the transistor sizing, determine the total driver area, and estimate the peak output current.

---

## Worked Solution

### Step 1: Calculate nominal transistor widths

For the NMOS pull-down network (target 40 ohm):

```
W_nmos = Ron_spec / R_target = 180 ohm-um / 40 ohm = 4.5 um
```

For the PMOS pull-up network (target 40 ohm):

```
W_pmos = Ron_spec / R_target = 360 ohm-um / 40 ohm = 9.0 um
```

### Step 2: Design the binary-weighted leg structure

With 5 binary-weighted legs, the weights are 1x, 2x, 4x, 8x, 16x. The total weight when all legs are enabled is 1+2+4+8+16 = 31x.

The nominal operating point should be near the middle of the range to allow calibration in both directions. We set the nominal code to enable legs that sum to approximately 16x (half of 31x), so the total enabled width equals the target.

For NMOS pull-down:
```
Unit width (1x leg) = W_nmos / 16 = 4.5 / 16 = 0.281 um
```

Total width of all legs:
```
W_total_nmos = 31 x 0.281 = 8.72 um
```

For PMOS pull-up:
```
Unit width (1x leg) = W_pmos / 16 = 9.0 / 16 = 0.5625 um
```

Total width of all legs:
```
W_total_pmos = 31 x 0.5625 = 17.44 um
```

### Step 3: Verify the calibration range

Minimum impedance (all legs enabled, code = 31):
```
R_min_nmos = 180 / 8.72 = 20.6 ohm
R_min_pmos = 360 / 17.44 = 20.6 ohm
```

Maximum impedance (only 1x leg enabled, code = 1):
```
R_max_nmos = 180 / 0.281 = 640 ohm
R_max_pmos = 360 / 0.5625 = 640 ohm
```

Impedance at nominal code (16x enabled):
```
R_nom_nmos = 180 / (16 x 0.281) = 180 / 4.5 = 40 ohm (matches target)
R_nom_pmos = 360 / (16 x 0.5625) = 360 / 9.0 = 40 ohm (matches target)
```

Range around nominal: from code 11 (28 ohm) to code 23 (56 ohm), which is -30% to +40% of 40 ohm. This exceeds the plus or minus 30% requirement.

### Step 4: Estimate the driver area

In a 5nm FinFET process, each fin is approximately 5nm wide with a pitch of approximately 25-30 nm. For the transistor widths calculated:

NMOS total: 8.72 um = approximately 290 fins (at 30 nm pitch)
PMOS total: 17.44 um = approximately 580 fins

Assuming a standard cell height of approximately 200 nm per fin row with 4 fins per row:
- NMOS rows: 290/4 = ~73 rows
- PMOS rows: 580/4 = ~145 rows

With typical layout overhead (contacts, metal routing, spacing), each row is approximately 100 nm tall:
- NMOS area: ~73 rows x 0.1 um x 5 um (length) = ~37 um^2
- PMOS area: ~145 rows x 0.1 um x 5 um = ~73 um^2
- Total driver area: ~110 um^2

Including pre-drivers, control logic, and routing overhead (3-4x multiplier):
**Estimated driver area per DQ pin: approximately 400-500 um^2**

### Step 5: Calculate peak output current

When driving a logic low (NMOS on, PMOS off) into a 40-ohm ODT to VDDQ:
```
I_peak = VDDQ / (R_driver + R_ODT) = 0.5V / (40 + 40) = 6.25 mA
```

When driving a logic high (PMOS on, NMOS off) into a 40-ohm ODT to VSS:
```
I_peak = VDDQ / (R_driver + R_ODT) = 0.5V / (40 + 40) = 6.25 mA
```

For 8 DQ pins switching simultaneously (worst case):
```
I_total_peak = 8 x 6.25 = 50 mA per byte lane
```

For all 4 channels (32 DQ pins):
```
I_total = 32 x 6.25 = 200 mA peak switching current
```

### Step 6: Layout recommendations

1. The PMOS pull-up is approximately 2x wider than the NMOS pull-down due to the lower PMOS mobility, requiring more layout area for the pull-up network.
2. The 5-bit binary-weighted architecture requires careful unit cell matching for the smaller legs (1x, 2x) to ensure calibration accuracy.
3. The 200 mA peak switching current requires a VDDQ power grid with less than 125 mohm resistance from the nearest decoupling to the driver (to keep IR drop under 25 mV, which is 5% of VDDQ).

---

See also:
- [IO Cell Design](../io_cell_design.md)
- [Power Grid for Memory IO](../../03_layout_fundamentals/power_grid_for_memory_io.md)
