# Worked Problem: Bump Assignment

## Problem Statement

Design the bump assignment for one LPDDR5 channel (8-bit data, single rank) with the following signal list:
- DQ[7:0]: 8 data signals
- DQS, DQS_n: 1 differential strobe pair
- WCK, WCK_n: 1 differential write clock pair
- CA[6:0]: 7 command/address signals
- CK, CK_n: 1 differential system clock pair
- CS_n: 1 chip select
- VDDQ power bumps
- VSS ground bumps
- ZQ: 1 calibration pad (shared, but allocated here)

Constraints: 0.4 mm bump pitch, maximum 6 rows deep, signal-to-power ratio approximately 1:1. Determine the bump arrangement.

---

## Worked Solution

### Step 1: Count signal bumps

| Signal Group | Count |
|---|---|
| DQ[7:0] | 8 |
| DQS/DQS_n | 2 |
| WCK/WCK_n | 2 |
| CA[6:0] | 7 |
| CK/CK_n | 2 |
| CS_n | 1 |
| ZQ | 1 |
| **Total signal bumps** | **23** |

### Step 2: Determine power/ground bumps

With a 1:1 signal-to-power ratio: 23 power/ground bumps needed.
Split approximately 60% VSS and 40% VDDQ: 14 VSS + 9 VDDQ = 23 power/ground bumps.

Total bumps: 23 + 23 = 46 bumps.

### Step 3: Determine grid dimensions

At 0.4 mm pitch with 6 rows maximum:
- Columns needed: ceil(46 / 6) = 8 columns
- Grid: 8 columns x 6 rows = 48 positions (2 spare)

Physical dimensions: 8 x 0.4 mm = 3.2 mm wide, 6 x 0.4 mm = 2.4 mm deep.

### Step 4: Assign bumps to grid positions

Principles: DQ byte signals grouped together, each signal bump flanked by VSS for return current, differential pairs on adjacent positions, power bumps distributed uniformly.

```
Column:    1      2      3      4      5      6      7      8
Row 1:   VSS    DQ0    VSS    DQ2    VSS    DQ4    VSS    DQ6
Row 2:   VDDQ   DQ1    VDDQ   DQ3    VDDQ   DQ5    VDDQ   DQ7
Row 3:   VSS    DQS    DQS_n  VSS    WCK    WCK_n  VSS    ZQ
Row 4:   VDDQ   CA0    CA1    VSS    CA2    CA3    VDDQ   VSS
Row 5:   VSS    CA4    CA5    VDDQ   CA6    CS_n   VSS    VDDQ
Row 6:   VDDQ   CK     CK_n   VSS    VSS    VDDQ   VSS    VSS
```

### Step 5: Verify the assignment

Signal bumps: DQ0-DQ7 (8) + DQS/DQS_n (2) + WCK/WCK_n (2) + CA0-CA6 (7) + CK/CK_n (2) + CS_n (1) + ZQ (1) = 23. Correct.

VDDQ bumps: Row 2 (4) + Row 4 (2) + Row 5 (2) + Row 6 (2) = 10
VSS bumps: Row 1 (4) + Row 3 (3) + Row 4 (1) + Row 5 (2) + Row 6 (3) + spare = 13 + 2 spare = 15

Total power/ground: 10 + 15 = 25. Signal-to-power ratio = 23:25 ~ 1:1.1. Acceptable.

### Step 6: Verify signal integrity considerations

- Every DQ signal has at least one adjacent VSS bump for return current path
- DQS differential pair is on adjacent bumps (Row 3, columns 2-3)
- WCK differential pair is on adjacent bumps (Row 3, columns 5-6)
- CK differential pair is on adjacent bumps (Row 6, columns 2-3)
- CA signals are grouped in Rows 4-5, close to CK in Row 6
- DQ signals are in Rows 1-2, close to DQS in Row 3

### Step 7: Routing implications

The outermost rows (1-2) contain DQ signals -- these have the shortest escape routes from the bump field to the IO cells, which is desirable for the highest-speed signals. The CK and CA signals are in the inner rows (4-6), which have longer escape routes but are lower frequency (CK rate vs data rate), making this acceptable.

---

See also:
- [Bump and Ball Map Design](../bump_and_ball_map_design.md)
- [PoP and Package Design](../../06_package_and_system/pop_and_package_design.md)
