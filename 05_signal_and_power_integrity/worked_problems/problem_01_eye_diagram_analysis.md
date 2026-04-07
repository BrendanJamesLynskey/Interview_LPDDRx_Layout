# Worked Problem: Eye Diagram Analysis

## Problem Statement

An LPDDR5 interface at 6400 MT/s has the following simulated eye diagram parameters at the SoC receiver (read path):

- Ideal eye height (no degradation): 250 mV (VDDQ/2 with matched termination)
- ISI penalty (from channel loss and reflections): 80 mV
- Crosstalk penalty (from 2 adjacent aggressors): 25 mV
- VDDQ noise (from PI simulation): 20 mV
- Ideal eye width: 156.25 ps (1 UI)
- DQS jitter (peak-to-peak): 12 ps
- DQ-DQS skew (from layout): 8 ps
- Crosstalk-induced jitter: 5 ps
- Supply-induced timing shift: 6 ps
- Receiver tDS: 50 ps, tDH: 50 ps
- Receiver minimum input voltage: 40 mV

Determine whether the eye meets specifications and calculate the voltage and timing margins.

---

## Worked Solution

### Step 1: Calculate the eye height

```
Eye height = Ideal height - ISI penalty - Crosstalk penalty - VDDQ noise
           = 250 - 80 - 25 - 20
           = 125 mV
```

Compare against receiver minimum: 125 mV > 40 mV. Passes with 85 mV margin.

### Step 2: Calculate the eye width

```
Eye width = Ideal width - DQS jitter - DQ-DQS skew - Crosstalk jitter - Supply timing shift
          = 156.25 - 12 - 8 - 5 - 6
          = 125.25 ps
```

Compare against receiver requirement: tDS + tDH = 50 + 50 = 100 ps.
Eye width 125.25 ps > 100 ps. Passes with 25.25 ps timing margin.

### Step 3: Calculate the margins as percentages

```
Voltage margin = (Eye height - Minimum) / Ideal height
               = (125 - 40) / 250
               = 34% of ideal
```

```
Timing margin = (Eye width - Required window) / UI
              = (125.25 - 100) / 156.25
              = 16.2% of UI
```

### Step 4: Identify the critical margin

The timing margin (16.2%) is tighter than the voltage margin (34%), indicating that timing is the limiting factor. This is typical for LPDDR5.

### Step 5: Sensitivity analysis

Which parameter has the most impact on timing margin?

| Parameter | Reduction by 50% | New timing margin |
|---|---|---|
| DQS jitter: 12 -> 6 ps | +6 ps | 31.25 ps (20%) |
| DQ-DQS skew: 8 -> 4 ps | +4 ps | 29.25 ps (18.7%) |
| Supply timing: 6 -> 3 ps | +3 ps | 28.25 ps (18.1%) |
| Crosstalk jitter: 5 -> 2.5 ps | +2.5 ps | 27.75 ps (17.8%) |

DQS jitter has the most impact. However, reducing DQ-DQS skew (layout controllable) provides the second-largest improvement.

### Step 6: Assessment

The design passes both voltage and timing specifications, but the timing margin is thin (25 ps or 16%). This leaves limited headroom for PVT variation, process corners not captured in the nominal simulation, and aging effects over the product lifetime.

Recommendation: tighten the DQ-DQS length matching (reducing layout skew from 8 ps to 4 ps) and improve the power grid (reducing supply timing shift from 6 ps to 3 ps) to increase the timing margin to approximately 32 ps (20%).

---

See also:
- [SI for LPDDR](../si_for_lpddr.md)
- [LPDDR Signaling and Timing](../../01_foundations/lpddr_signaling_and_timing.md)
