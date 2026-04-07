# Worked Problem: Timing Margin Analysis

## Problem Statement

For an LPDDR5 interface operating at 6400 MT/s, calculate the read-side timing margin at the SoC receiver. The following parameters are given:

- Data rate: 6400 MT/s
- tDS (setup time): 55 ps
- tDH (hold time): 55 ps
- DQS jitter (total, peak-to-peak): 15 ps
- VDDQ-induced timing shift: 10 ps
- Crosstalk-induced jitter: 8 ps
- SoC on-die DQ-to-DQS skew (from layout): 12 ps
- Package DQ-to-DQS skew: 5 ps
- DRAM tDQS2DQ contribution: 20 ps

Determine whether the design has positive timing margin and identify which contributor should be reduced if the margin is insufficient.

---

## Worked Solution

### Step 1: Calculate the Unit Interval (UI)

```
UI = 1 / Data Rate = 1 / (6400 x 10^6) = 156.25 ps
```

This is the total time available for one data bit. The data eye must fit within this window.

### Step 2: Understand the timing budget structure

For a centre-aligned read (DQS is centre-aligned with DQ at the DRAM, meaning DQS edge is in the middle of the DQ eye), the available window at the receiver is:

```
Available window = 1 UI = 156.25 ps
```

From this window, we subtract all timing degradation sources to find the remaining margin.

### Step 3: List all timing deductions

| Source | Value (ps) | Category |
|---|---|---|
| tDS (setup time) | 55 | Receiver requirement |
| tDH (hold time) | 55 | Receiver requirement |
| DQS jitter (pk-pk) | 15 | Clock uncertainty |
| VDDQ-induced timing shift | 10 | Power integrity |
| Crosstalk-induced jitter | 8 | Signal integrity |
| SoC on-die DQ-DQS skew | 12 | Layout |
| Package DQ-DQS skew | 5 | Package |
| DRAM tDQS2DQ | 20 | DRAM device |

### Step 4: Calculate total timing deduction

The setup and hold times define the minimum required eye opening. The remaining sources reduce the effective eye opening by shifting the DQS sampling point or distorting the DQ signal.

```
Total deduction = tDS + tDH + DQS_jitter + VDDQ_shift + Xtalk_jitter 
                  + SoC_skew + Pkg_skew + DRAM_tDQS2DQ

Total deduction = 55 + 55 + 15 + 10 + 8 + 12 + 5 + 20
                = 180 ps
```

### Step 5: Calculate timing margin

```
Timing margin = UI - Total deduction
              = 156.25 - 180
              = -23.75 ps
```

The timing margin is NEGATIVE, meaning the design does not close timing. The system cannot reliably operate at 6400 MT/s with these parameters.

### Step 6: Identify the critical contributors

Ranking the controllable contributors by magnitude:

1. Receiver setup + hold (110 ps combined) -- This is a circuit design parameter, not directly controllable by layout. Better receiver design can reduce this.
2. DRAM tDQS2DQ (20 ps) -- This is a DRAM device parameter. Selecting a lower-skew DRAM vendor can help.
3. DQS jitter (15 ps) -- Reduce by improving PLL design, power supply filtering, and DQS routing quality.
4. SoC on-die DQ-DQS skew (12 ps) -- This is directly controllable by layout. Tighter length matching can reduce this to 5-8 ps.
5. VDDQ-induced timing shift (10 ps) -- Improve power grid and decoupling to reduce VDDQ ripple.
6. Crosstalk-induced jitter (8 ps) -- Improve signal spacing or add shielding in layout.
7. Package DQ-DQS skew (5 ps) -- Optimise package routing.

### Step 7: Determine required improvements

To achieve a positive margin of at least 10 ps (minimum recommended guard band):

```
Required total deduction < 156.25 - 10 = 146.25 ps
Current total deduction = 180 ps
Required reduction = 180 - 146.25 = 33.75 ps
```

A realistic improvement plan:

| Source | Current (ps) | Target (ps) | Savings (ps) |
|---|---|---|---|
| tDS + tDH | 110 | 95 | 15 |
| SoC on-die skew | 12 | 6 | 6 |
| VDDQ timing shift | 10 | 5 | 5 |
| Crosstalk jitter | 8 | 3 | 5 |
| DQS jitter | 15 | 10 | 5 |
| **Total savings** | | | **36** |

This brings the total deduction to 144 ps, yielding a margin of 12.25 ps -- barely sufficient. This analysis demonstrates why LPDDR5 at 6400 MT/s requires aggressive optimisation across all contributors, with layout playing a critical role in managing skew, crosstalk, and power integrity.

### Step 8: Key takeaways for layout engineers

- The layout-controllable contributors (on-die skew, crosstalk, VDDQ ripple impact) collectively account for 30 ps in this example -- a substantial portion of the total budget.
- Reducing SoC on-die DQ-DQS skew from 12 ps to 6 ps requires length matching within approximately plus or minus 40 micrometres (at ~150 ps/mm propagation velocity), which is achievable with careful routing.
- Crosstalk reduction from 8 ps to 3 ps may require adding shield traces between DQ signals or increasing inter-signal spacing.
- VDDQ improvement requires more decoupling capacitance and lower-impedance power grid routing near the IO cells.

---

See also:
- [LPDDR Signaling and Timing](../lpddr_signaling_and_timing.md)
- [SI for LPDDR](../../05_signal_and_power_integrity/si_for_lpddr.md)
- [Length Matching and Skew](../../04_routing_and_matching/length_matching_and_skew.md)
