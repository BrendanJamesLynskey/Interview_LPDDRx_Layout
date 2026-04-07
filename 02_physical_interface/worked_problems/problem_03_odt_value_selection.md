# Worked Problem: ODT Value Selection

## Problem Statement

An LPDDR5 interface operates at 6400 MT/s with a PoP interconnect that has the following channel characteristics:

- Transmission line impedance (Zo): 45 ohm
- One-way propagation delay: 150 ps
- Transmission line loss at Nyquist (3.2 GHz): 1.5 dB
- VDDQ: 0.5V
- DRAM driver impedance: 40 ohm
- SoC receiver sensitivity: 80 mV minimum eye height

Evaluate three ODT configurations for read operations (SoC-side ODT) and recommend the optimal setting:
- Option A: 40 ohm ODT
- Option B: 60 ohm ODT
- Option C: 120 ohm ODT

---

## Worked Solution

### Step 1: Calculate received voltage swing for each ODT value

The received voltage at the SoC (after voltage division between the DRAM driver impedance and SoC ODT) is:

```
V_received = VDDQ x R_ODT / (R_driver + R_ODT)
```

Option A (40 ohm ODT):
```
V_received = 0.5 x 40 / (40 + 40) = 0.5 x 0.5 = 250 mV
```

Option B (60 ohm ODT):
```
V_received = 0.5 x 60 / (40 + 60) = 0.5 x 0.6 = 300 mV
```

Option C (120 ohm ODT):
```
V_received = 0.5 x 120 / (40 + 120) = 0.5 x 0.75 = 375 mV
```

### Step 2: Calculate reflection coefficients

At the SoC receiver (ODT end):
```
Gamma_receiver = (R_ODT - Zo) / (R_ODT + Zo)
```

Option A: Gamma = (40 - 45) / (40 + 45) = -5/85 = -0.059 (near perfect match)
Option B: Gamma = (60 - 45) / (60 + 45) = 15/105 = 0.143
Option C: Gamma = (120 - 45) / (120 + 45) = 75/165 = 0.455

At the DRAM driver end:
```
Gamma_driver = (R_driver - Zo) / (R_driver + Zo) = (40 - 45) / (40 + 45) = -0.059
```

### Step 3: Calculate reflection-induced voltage at the receiver

The first reflection from the receiver arrives back at the driver after one round trip (2 x 150 ps = 300 ps). At 6400 MT/s, 1 UI = 156.25 ps, so the reflection arrives approximately 2 UI after the initial signal. This means the reflection from bit N affects bit N+2.

Reflected voltage at receiver (first reflection returning):
```
V_reflection = V_incident x Gamma_receiver x Gamma_driver x (channel_loss_factor)^2
```

Channel loss factor (round trip, 2 x 1.5 dB = 3.0 dB):
```
Loss_factor = 10^(-3.0/20) = 0.708
```

Option A: V_refl = 250 x (-0.059) x (-0.059) x 0.708 = 0.6 mV (negligible)
Option B: V_refl = 300 x 0.143 x (-0.059) x 0.708 = -1.8 mV (small)
Option C: V_refl = 375 x 0.455 x (-0.059) x 0.708 = -7.1 mV (noticeable)

### Step 4: Account for channel loss on the signal

The received signal after one-way channel loss (1.5 dB):
```
Loss_factor_one_way = 10^(-1.5/20) = 0.841
```

Option A: V_at_receiver = 250 x 0.841 = 210 mV
Option B: V_at_receiver = 300 x 0.841 = 252 mV
Option C: V_at_receiver = 375 x 0.841 = 315 mV

### Step 5: Calculate effective eye height

Effective eye height = V_at_receiver - ISI penalty - reflection penalty - noise margin deduction

Assume ISI penalty (from channel loss on data pattern) = 30 mV for all options (pattern-dependent loss).
Assume VDDQ noise = 15 mV (3% of 500 mV).
Assume crosstalk = 10 mV.

| Parameter | Option A (40R) | Option B (60R) | Option C (120R) |
|---|---|---|---|
| V_at_receiver (mV) | 210 | 252 | 315 |
| ISI penalty (mV) | 30 | 30 | 30 |
| Reflection penalty (mV) | 0.6 | 1.8 | 7.1 |
| VDDQ noise (mV) | 15 | 15 | 15 |
| Crosstalk (mV) | 10 | 10 | 10 |
| **Effective eye height (mV)** | **154.4** | **195.2** | **252.9** |
| Margin above 80 mV threshold | 74.4 | 115.2 | 172.9 |

### Step 6: Calculate ODT power consumption

DC power per pin (average for random data):
```
P_ODT = VDDQ^2 / (4 x R_ODT)
```

Option A: P = 0.25 / 160 = 1.56 mW per pin, 12.5 mW per byte
Option B: P = 0.25 / 240 = 1.04 mW per pin, 8.3 mW per byte
Option C: P = 0.25 / 480 = 0.52 mW per pin, 4.2 mW per byte

### Step 7: Recommendation

| Criteria | Option A (40R) | Option B (60R) | Option C (120R) |
|---|---|---|---|
| Eye height margin | Adequate | Good | Excellent |
| Reflection quality | Excellent | Good | Poor |
| Power consumption | Highest | Moderate | Lowest |
| **Recommendation** | Conservative | **Optimal** | Risky |

**Option B (60 ohm ODT) is recommended.** It provides good eye height (195 mV, well above the 80 mV threshold), moderate reflections (1.8 mV, negligible), and moderate power consumption (8.3 mW per byte). Option A wastes power for margin that is not needed. Option C, while showing the best eye height in this simple analysis, has larger reflections that could worsen in a more complex channel with additional discontinuities.

In practice, the final ODT selection should be validated with full-channel SI simulation including package and PoP parasitics, and fine-tuned during silicon bring-up using the eye scanning capability of the PHY.

---

See also:
- [Termination and ODT](../termination_and_odt.md)
- [SI for LPDDR](../../05_signal_and_power_integrity/si_for_lpddr.md)
