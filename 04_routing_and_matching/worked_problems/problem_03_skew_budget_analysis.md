# Worked Problem: Skew Budget Analysis

## Problem Statement

Create a complete skew budget for an LPDDR5 interface operating at 6400 MT/s for the read path (data from DRAM to SoC). Determine whether the design has positive margin, and identify the top contributors to timing closure risk.

Given parameters:
- Data rate: 6400 MT/s (UI = 156.25 ps)
- Receiver tDS (setup): 50 ps
- Receiver tDH (hold): 50 ps
- DRAM tDQS2DQ (max): 25 ps
- DRAM DQS jitter (pk-pk): 8 ps
- DRAM DCD (duty cycle distortion): 5 ps
- Package routing DQ-DQS skew: 8 ps
- SoC on-die DQ-DQS skew: 10 ps
- SoC PLL jitter contribution to DQS path: 5 ps
- VDDQ noise timing impact: 8 ps
- Crosstalk timing impact: 5 ps
- Temperature-induced drift (between calibrations): 3 ps

---

## Worked Solution

### Step 1: Define the timing budget

```
Total available window = 1 UI = 156.25 ps
Required window = tDS + tDH = 50 + 50 = 100 ps
Available margin = 156.25 - 100 = 56.25 ps
```

### Step 2: List all jitter and skew contributors

| Contributor | Value (ps) | Category | Controllable by layout? |
|---|---|---|---|
| DRAM tDQS2DQ | 25 | DRAM device | No |
| DRAM DQS jitter | 8 | DRAM device | No |
| DRAM DCD | 5 | DRAM device | No |
| Package DQ-DQS skew | 8 | Package | Partially |
| SoC on-die DQ-DQS skew | 10 | Layout | Yes |
| SoC PLL jitter | 5 | Circuit + Layout | Partially |
| VDDQ noise timing | 8 | Layout | Yes |
| Crosstalk timing | 5 | Layout | Yes |
| Temperature drift | 3 | Environment | No |
| **Total** | **77** | | |

### Step 3: Calculate remaining margin

```
Remaining margin = Available margin - Total contributors
                 = 56.25 - 77
                 = -20.75 ps (NEGATIVE)
```

The budget does not close with this simple addition.

### Step 4: Apply statistical analysis (RSS for uncorrelated contributors)

Not all contributors are worst-case simultaneously. Statistically independent contributors can be combined using root-sum-square (RSS):

Correlated (add linearly): DRAM tDQS2DQ, DRAM DCD
Uncorrelated (RSS): DRAM DQS jitter, Package skew, SoC on-die skew, PLL jitter, VDDQ noise, Crosstalk, Temperature drift

```
Linear sum = 25 + 5 = 30 ps

RSS contributors = sqrt(8^2 + 8^2 + 10^2 + 5^2 + 8^2 + 5^2 + 3^2)
                 = sqrt(64 + 64 + 100 + 25 + 64 + 25 + 9)
                 = sqrt(351)
                 = 18.7 ps

Total (statistical) = 30 + 18.7 = 48.7 ps
```

### Step 5: Recalculate margin with statistical analysis

```
Remaining margin = 56.25 - 48.7 = 7.55 ps
```

The budget closes with 7.55 ps of statistical margin. However, this is thin.

### Step 6: Identify top contributors and improvement opportunities

| Contributor | Value (ps) | Improvement target (ps) | Method |
|---|---|---|---|
| DRAM tDQS2DQ | 25 | 20 | Select lower-skew DRAM |
| SoC on-die skew | 10 | 6 | Tighter length matching |
| VDDQ noise timing | 8 | 5 | Better power grid |
| Package skew | 8 | 5 | Optimise package routing |
| Crosstalk timing | 5 | 3 | Add shielding |

With these improvements:
```
Linear: 20 + 5 = 25 ps
RSS: sqrt(8^2 + 5^2 + 6^2 + 5^2 + 5^2 + 3^2 + 3^2) = sqrt(169) = 13 ps
Total: 25 + 13 = 38 ps
Margin: 56.25 - 38 = 18.25 ps
```

This provides a comfortable 18.25 ps margin, approximately 3 sigma.

### Step 7: Key takeaways

1. Simple worst-case addition rarely closes timing for LPDDR5 -- statistical analysis is necessary.
2. The DRAM tDQS2DQ is the single largest contributor, but it is not layout-controllable.
3. Layout-controllable contributors (on-die skew, VDDQ noise, crosstalk) collectively represent 23 ps worst-case or approximately 14 ps RSS -- a substantial portion of the budget.
4. A 7.55 ps statistical margin is tight but acceptable if the analysis is validated with silicon measurements.

---

See also:
- [Length Matching and Skew](../length_matching_and_skew.md)
- [LPDDR Signaling and Timing](../../01_foundations/lpddr_signaling_and_timing.md)
- [SI for LPDDR](../../05_signal_and_power_integrity/si_for_lpddr.md)
