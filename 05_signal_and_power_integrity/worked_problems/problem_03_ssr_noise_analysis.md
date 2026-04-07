# Worked Problem: SSR Noise Analysis

## Problem Statement

Analyse the SSR (Simultaneous Switching Receiver) noise impact during an LPDDR5 read operation at 6400 MT/s. The SoC receiver has the following characteristics:

- 8 DQ receivers per byte lane, each with input capacitance Cin = 200 fF
- ODT resistance: 60 ohm per receiver
- VSS bond inductance per bump: 80 pH, with 5 VSS bumps for the byte lane
- VDDQ bond inductance per bump: 80 pH, with 4 VDDQ bumps
- Signal transition time at receiver: 100 ps
- VDDQ = 0.5V

During a read, the DRAM drives all 8 DQ pins, and the SoC receivers activate ODT. Calculate the SSR noise on VSS and VDDQ, and determine its impact on timing.

---

## Worked Solution

### Step 1: Calculate the receiver switching current

Each receiver with 60 ohm ODT draws current based on the input signal level:
```
I_receiver = VDDQ / (R_driver + R_ODT)
           = 0.5V / (40 + 60) = 5 mA per pin (when signal is at one rail)
```

For worst-case SSR (all 8 pins receive the same data value):
```
I_total = 8 x 5 mA = 40 mA (DC through ODT)
```

The current change during a data transition (all pins switch simultaneously):
```
di = 40 mA (from full ODT current to near zero, or vice versa)
dt = 100 ps (transition time)
di/dt = 40 mA / 100 ps = 4 x 10^8 A/s
```

### Step 2: Calculate VSS bounce

Effective VSS inductance (5 bumps in parallel):
```
L_VSS = 80 pH / 5 = 16 pH
```

VSS bounce:
```
V_VSS_bounce = L_VSS x di/dt = 16 x 10^-12 x 4 x 10^8 = 6.4 mV
```

### Step 3: Calculate VDDQ droop

Effective VDDQ inductance (4 bumps in parallel):
```
L_VDDQ = 80 pH / 4 = 20 pH
```

VDDQ droop:
```
V_VDDQ_droop = L_VDDQ x di/dt = 20 x 10^-12 x 4 x 10^8 = 8.0 mV
```

### Step 4: Calculate total supply noise

```
Total supply noise = V_VSS_bounce + V_VDDQ_droop = 6.4 + 8.0 = 14.4 mV
```

As a percentage of VDDQ: 14.4 / 500 = 2.9%

### Step 5: Calculate timing impact

Assuming a supply-to-timing sensitivity of 3 ps/mV (typical for LPDDR5 receivers):
```
Timing impact = 14.4 mV x 3 ps/mV = 43.2 ps
```

This is a SEVERE timing penalty -- 43.2 ps consumes 28% of the 156.25 ps UI.

### Step 6: Assess and mitigate

The 43.2 ps timing impact is too large. Mitigation options:

1. **Increase VSS/VDDQ bumps:** Adding 3 more VSS bumps (total 8) and 2 more VDDQ bumps (total 6):
   - L_VSS = 80/8 = 10 pH, L_VDDQ = 80/6 = 13.3 pH
   - V_VSS = 10 x 4e8 = 4.0 mV
   - V_VDDQ = 13.3 x 4e8 = 5.3 mV
   - Total: 9.3 mV, timing: 28 ps (improved but still significant)

2. **Add on-die decoupling:** 200 pF of MIM capacitance with 10 pH ESL:
   - Capacitor supplies current during the 100 ps transition
   - V_droop = I*dt/C = 40e-3 * 100e-12 / 200e-12 = 20 mV
   - But ESL limits: V_ESL = 10 pH x 4e8 = 4 mV
   - Effective: max(4, 20) = 4 mV from capacitor path (faster response than droop)

3. **Combined (bumps + decoupling):**
   - Package path: 9.3 mV at the bump
   - On-die MIM provides local current: reduces net noise to ~5-7 mV
   - Timing impact: 5 mV x 3 ps/mV = 15 ps (acceptable)

### Step 7: Key findings

- SSR noise during read operations is significant but approximately half the magnitude of SSO noise during writes (because ODT current is lower than driver current).
- The timing impact (3 ps/mV sensitivity) makes even modest supply noise a major concern.
- Combined mitigation (additional bumps + on-die decoupling) is needed to bring the timing impact below 15-20 ps.

---

See also:
- [EMI and Noise Mitigation](../emi_and_noise_mitigation.md)
- [Power Integrity for LPDDR](../power_integrity_for_lpddr.md)
