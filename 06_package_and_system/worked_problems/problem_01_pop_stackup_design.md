# Worked Problem: PoP Stackup Design

## Problem Statement

Design a PoP stackup for an LPDDR5 interface with the following requirements:
- SoC die: 10 mm x 10 mm, 5nm process, flip-chip bump pitch 130 um
- SoC package: 14 mm x 14 mm, 6-layer organic substrate
- DRAM: x32 LPDDR5 (2 dies, each x16), stacked with wire bonds
- DRAM package: 14 mm x 14 mm, 2-layer substrate
- TMV pitch: 0.4 mm, TMV diameter: 150 um
- Target LPDDR5 data rate: 6400 MT/s

Determine the stackup dimensions, TMV allocation, and identify SI-critical elements.

---

## Worked Solution

### Step 1: Define the vertical stackup

```
Top
  |-- DRAM die 2 (wire-bonded to DRAM substrate)
  |-- DRAM die 1 (wire-bonded to DRAM substrate)
  |-- DRAM package substrate (2 layers, 0.2 mm thick)
  |-- Solder balls (DRAM-to-SoC, 0.3 mm height after reflow)
  |-- TMVs (through the mold compound, 0.3 mm height)
  |-- SoC package substrate top surface
  |-- SoC package substrate (6 layers, 0.4 mm thick)
  |-- Flip-chip bumps (SoC die to substrate, 0.08 mm height)
  |-- SoC die (0.1 mm thick, face down)
  |-- Mold compound (encapsulating die, 0.3 mm above die back)
Bottom
  |-- SoC package BGA balls (to PCB, 0.3 mm height)
```

Total PoP stack height:
- SoC package: 0.08 (bumps) + 0.1 (die) + 0.3 (mold) + 0.4 (substrate) = 0.88 mm
- Interconnect: 0.3 (TMV/solder) = 0.3 mm
- DRAM package: 0.2 (substrate) + 0.15 (die+wire bonds) x 2 = 0.5 mm
- Mold on DRAM: 0.2 mm
- Total: approximately 1.9 mm (typical for mobile PoP)

### Step 2: Allocate TMVs

TMVs are placed at the periphery of the SoC package (outside the die area). For a 14 mm package with 10 mm die, the TMV area is:
- Available perimeter ring: 14 mm x 14 mm minus 10 mm x 10 mm = 96 mm^2
- At 0.4 mm pitch: approximately (14/0.4) x 4 sides x 2 rows = ~280 TMV positions

TMV allocation for LPDDR x32 interface:
- Signal TMVs: ~96 (4 channels x 24 signals per channel)
- VDDQ TMVs: ~40 (providing low-inductance power delivery)
- VSS TMVs: ~60 (providing return current paths)
- VDD2/VDD1 TMVs: ~16
- Total LPDDR TMVs: ~212

Remaining TMVs: 280 - 212 = 68 for other interfaces and mechanical support.

### Step 3: Identify SI-critical elements

| Element | Parasitic | Impact on SI |
|---|---|---|
| SoC flip-chip bump | 30 pH L, 50 fF C | Small impedance discontinuity |
| SoC substrate routing | 2-4 mm, 50 ohm | Moderate loss, controlled impedance |
| SoC substrate via | 50 pH L, 100 fF C | Impedance discontinuity |
| TMV | 100-150 pH L, 80 fF C | **Major discontinuity** |
| TMV-DRAM solder joint | 30 pH L, 30 fF C | Small discontinuity |
| DRAM substrate routing | 1-2 mm, 50 ohm | Moderate loss |
| DRAM wire bond | 1-2 nH L, 50 fF C | **Major discontinuity and inductive** |

### Step 4: Critical findings

The two most SI-critical elements are the TMVs (100-150 pH inductance per via, creating impedance bumps) and the DRAM wire bonds (1-2 nH inductance, creating significant impedance discontinuity and bandwidth limitation).

The wire bonds are the dominant SI limiter. At 1 nH inductance, the impedance at 3.2 GHz is:
```
Z_wire = 2*pi*f*L = 6.28 * 3.2e9 * 1e-9 = 20 ohm
```

This 20 ohm inductive impedance in series with the 50-ohm trace creates a significant mismatch and limits the channel bandwidth.

### Step 5: Recommendations

1. Minimise wire bond length (shorter bonds = lower inductance).
2. Use flip-chip DRAM packaging if available (eliminates wire bonds).
3. Optimise TMV impedance by adjusting diameter and ground via placement.
4. Place VDDQ and VSS TMVs adjacent to every signal TMV for return current.
5. Simulate the complete stackup with 3D EM tools before committing to fabrication.

---

See also:
- [PoP and Package Design](../pop_and_package_design.md)
- [Bump and Ball Map Design](../../03_layout_fundamentals/bump_and_ball_map_design.md)
