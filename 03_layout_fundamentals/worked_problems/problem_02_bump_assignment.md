# Worked Problem: Bump Assignment

## Problem Statement

Design the bump assignment for one LPDDR5 channel (16-bit data, single rank; an x32 interface uses two of these) with the following signal list:
- DQ[15:0]: 16 data signals (byte 0 = DQ[7:0], byte 1 = DQ[15:8])
- DMI[1:0]: 1 data-mask/inversion signal per byte
- RDQS0_t/c, RDQS1_t/c: 1 differential read strobe pair per byte
- WCK0_t/c, WCK1_t/c: 1 differential write clock pair per byte
- CA[6:0]: 7 command/address signals
- CK_t/c: 1 differential system clock pair
- CS: 1 chip select
- VDDQ power bumps
- VSS ground bumps
- ZQ: 1 calibration pad (shared, but allocated here)

(LPDDR5 replaces LPDDR4's bidirectional DQS with a write clock, WCK, and a read strobe, RDQS, for each byte.)

Constraints: 0.4 mm bump pitch, maximum 6 rows deep, signal-to-power ratio approximately 1:1. Determine the bump arrangement.

---

## Worked Solution

### Step 1: Count signal bumps

| Signal Group | Count |
|---|---|
| DQ[15:0] | 16 |
| DMI[1:0] | 2 |
| RDQS0_t/c, RDQS1_t/c | 4 |
| WCK0_t/c, WCK1_t/c | 4 |
| CA[6:0] | 7 |
| CK_t/c | 2 |
| CS | 1 |
| ZQ | 1 |
| **Total signal bumps** | **37** |

### Step 2: Determine power/ground bumps

With a 1:1 signal-to-power ratio: 37 power/ground bumps needed.
Split approximately 60% VSS and 40% VDDQ: 22 VSS + 15 VDDQ = 37 power/ground bumps.

Total bumps: 37 + 37 = 74 bumps.

### Step 3: Determine grid dimensions

At 0.4 mm pitch with 6 rows maximum:
- Columns needed: ceil(74 / 6) = 13 columns
- Grid: 13 columns x 6 rows = 78 positions (4 spare)

Physical dimensions: 13 x 0.4 mm = 5.2 mm wide, 6 x 0.4 mm = 2.4 mm deep. The full x32 interface uses two such fields: 26 columns, or 10.4 mm of bump-field width.

### Step 4: Assign bumps to grid positions

Principles:
- Each byte's signals are grouped together: byte 0 in columns 1-6 and byte 1 in columns 8-13, with a VSS/VDDQ column (7) between them.
- Each signal bump is flanked by power or ground for its return current.
- Differential pairs sit on adjacent positions.
- CA, CS and CK are centred between the two bytes they serve.
- Power bumps are distributed uniformly.

```
Column:  1     2       3       4    5      6      7     8      9      10   11      12      13
Row 1:   VSS   DQ0     DQ1     VSS  DQ2    DQ3    VSS   DQ8    DQ9    VSS  DQ10    DQ11    VSS
Row 2:   DQ4   VDDQ    DQ5     DQ6  VDDQ   DQ7    VDDQ  DQ12   VDDQ   DQ13 DQ14    VDDQ    DQ15
Row 3:   DMI0  RDQS0_t RDQS0_c VSS  WCK0_t WCK0_c VSS   WCK1_t WCK1_c VSS  RDQS1_t RDQS1_c DMI1
Row 4:   VDDQ  VSS     VDDQ    VSS  CA0    CA1    VSS   CA2    CA3    VSS  VDDQ    VSS     VDDQ
Row 5:   VSS   VDDQ    VSS     VDDQ CA4    CA5    VDDQ  CA6    CS     VDDQ VSS     VDDQ    VSS
Row 6:   VDDQ  VSS     VDDQ    VSS  VSS    CK_t   CK_c  VSS    ZQ     VSS  VDDQ    VSS     VDDQ
```

### Step 5: Verify the assignment

Signal bumps: DQ0-DQ15 (16) + DMI0-1 (2) + RDQS pairs (4) + WCK pairs (4) + CA0-CA6 (7) + CK_t/c (2) + CS (1) + ZQ (1) = 37. Correct.

VDDQ bumps: Row 2 (5) + Row 4 (4) + Row 5 (5) + Row 6 (4) = 18
VSS bumps: Row 1 (5) + Row 3 (3) + Row 4 (5) + Row 5 (4) + Row 6 (6) = 23 (the 4 spare positions are used as 1 extra VSS and 3 extra VDDQ)

Total power/ground: 18 + 23 = 41. Signal-to-power ratio = 37:41 ~ 1:1.1. Acceptable.

### Step 6: Verify signal integrity considerations

- Every signal bump has at least one orthogonally adjacent power/ground bump for its return current. DQ5, DQ7, DQ12, DQ14, DMI0, DMI1, CA5 and CS see VSS only diagonally, with VDDQ alongside.
- RDQS0 and RDQS1 differential pairs are on adjacent bumps (Row 3, columns 2-3 and 11-12)
- WCK0 and WCK1 differential pairs are on adjacent bumps (Row 3, columns 5-6 and 8-9)
- CK differential pair is on adjacent bumps (Row 6, columns 6-7), centred under the CA group
- CA signals are grouped in Rows 4-5 at the centre of the field, close to CK in Row 6, and equidistant from the two bytes that share them
- Each byte's DQ signals are in Rows 1-2, directly above their own RDQS/WCK pairs in Row 3

### Step 7: Routing implications

The outermost rows (1-2) contain the DQ signals. These have the shortest escape routes from the bump field to the IO cells, which is what the highest-speed signals need. Each byte's escape stays within its own six columns, so the byte lane is a self-contained skew group matched to its own RDQS/WCK. The CK and CA signals are in the inner rows (4-6). Their escape routes are longer, but they run at the lower CK rate rather than the data rate, so this is acceptable. Placing them at the centre also keeps the CA/CK-to-byte distance similar for both bytes of the channel.

---

See also:
- [Bump and Ball Map Design](../bump_and_ball_map_design.md)
- [PoP and Package Design](../../06_package_and_system/pop_and_package_design.md)
