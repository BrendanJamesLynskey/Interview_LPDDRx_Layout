# Worked Problem: Decoupling Strategy

## Problem Statement

Design the decoupling strategy for one LPDDR5 byte lane with the following requirements:

- VDDQ = 0.5V, noise budget = 20 mV (4%)
- Peak switching current: 100 mA (8 DQ pins, worst case all-same-direction)
- Transition time: 80 ps
- Package bump inductance: 80 pH per bump, 4 VDDQ bumps available
- On-die power grid inductance (bump to nearest IO cell): 20 pH
- Available decoupling types:
  - MIM capacitor: 15 fF/um^2, ESR = 20 mohm, ESL = 10 pH
  - MOM capacitor: 3 fF/um^2, ESR = 100 mohm, ESL = 5 pH
  - MOS capacitor: 8 fF/um^2, ESR = 200 mohm, ESL = 50 pH
- Available area for decoupling: 50,000 um^2 (50 um x 1000 um strip over the IO cells)

Determine the capacitor types, values, area allocation, and placement.

---

## Worked Solution

### Step 1: Calculate the target PDN impedance

```
Z_target = V_noise / I_peak = 20 mV / 100 mA = 200 mohm
```

This impedance must be maintained from DC to beyond the data rate (3.2 GHz Nyquist).

### Step 2: Calculate the effective package inductance

4 VDDQ bumps in parallel: L_bump = 80 pH / 4 = 20 pH
Plus on-die grid inductance: L_total = 20 + 20 = 40 pH

At the resonant frequency with on-die capacitance, the inductive impedance is:
```
Z_L = 2 * pi * f * L
At 1 GHz: Z_L = 6.28 * 10^9 * 40 * 10^-12 = 251 mohm
```

This exceeds Z_target at 1 GHz, so decoupling must reduce the impedance below 200 mohm at this frequency.

### Step 3: Determine the required on-die capacitance

For the impedance at 1 GHz to be below 200 mohm, the capacitive reactance must create a parallel impedance with the inductive path that is below 200 mohm.

The capacitive impedance to equal Z_target at 1 GHz:
```
Z_C = 1 / (2 * pi * f * C)
200 mohm = 1 / (6.28 * 10^9 * C)
C = 1 / (6.28 * 10^9 * 0.2) = 796 pF
```

However, this ignores the ESR and ESL of the capacitors, which limit effectiveness at high frequencies.

### Step 4: Design the capacitor mix

MIM capacitors (primary, high-frequency decoupling):
- Area: 30,000 um^2 (60% of available area)
- Capacitance: 30,000 x 15 = 450,000 fF = 450 pF
- Self-resonant frequency: f_SRF = 1 / (2*pi*sqrt(10e-12 * 450e-12)) = 2.37 GHz
- Effective up to ~2 GHz

MOM capacitors (supplementary, very high-frequency):
- Area: 10,000 um^2 (20% of available area)
- Capacitance: 10,000 x 3 = 30,000 fF = 30 pF
- Self-resonant frequency: f_SRF = 1 / (2*pi*sqrt(5e-12 * 30e-12)) = 13 GHz
- Effective up to ~10 GHz

MOS capacitors (low-frequency bulk):
- Area: 10,000 um^2 (20% of available area)
- Capacitance: 10,000 x 8 = 80,000 fF = 80 pF
- Self-resonant frequency: f_SRF = 1 / (2*pi*sqrt(50e-12 * 80e-12)) = 2.52 GHz
- But high ESR (200 mohm) provides good damping

### Step 5: Verify the PDN impedance

Total on-die capacitance: 450 + 30 + 80 = 560 pF

Resonant frequency of package inductance with on-die capacitance:
```
f_res = 1 / (2*pi*sqrt(40e-12 * 560e-12)) = 1.07 GHz
```

At resonance, the impedance is limited by the ESR of the capacitors (parallel combination):
- MIM ESR contribution: 20 mohm (dominant due to largest C)
- MOS ESR provides damping: 200 mohm in parallel with MIM
- Effective ESR at resonance: ~19 mohm

This is well below Z_target = 200 mohm. The resonance peak is heavily damped.

### Step 6: Verify transient response

First current spike: C must supply 100 mA for 80 ps before the package responds.
```
V_droop = I * dt / C = 100e-3 * 80e-12 / 560e-12 = 14.3 mV
```

This is within the 20 mV budget. With MOM and MIM capacitors responding within a few picoseconds (low ESL), the actual droop will be even less.

### Step 7: Placement recommendation

1. **MIM capacitors** (450 pF): Place directly over the IO cells, between M7 and M8 (or the designated MIM layer pair). Distribute evenly across the 1000 um width.
2. **MOM capacitors** (30 pF): Interdigitated metal fingers on M5/M6, in the IO cell region. Distributed uniformly.
3. **MOS capacitors** (80 pF): NMOS devices in the guard ring area surrounding the IO cells, connected gate-to-VDDQ and source/drain-to-VSS.

---

See also:
- [Power Integrity for LPDDR](../power_integrity_for_lpddr.md)
- [Power Grid for Memory IO](../../03_layout_fundamentals/power_grid_for_memory_io.md)
