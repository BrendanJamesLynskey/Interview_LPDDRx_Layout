# Worked Problem: PHY Placement Strategy

## Problem Statement

An SoC die measures 10 mm x 10 mm. The following blocks must be placed:
- LPDDR5 PHY: 4-channel x32 interface, requires 4 mm of die edge, 0.5 mm deep
- Memory controller: 1.5 mm x 0.8 mm
- GPU: 4 mm x 3 mm (high power, 3W)
- CPU cluster: 3 mm x 2 mm (high power, 2W)
- Display interface: 2 mm of die edge
- Camera interface: 1.5 mm of die edge
- PCIe/USB: 2 mm of die edge

The DRAM PoP package is located above the top edge of the die. Determine the optimal placement of the LPDDR PHY and memory controller, and justify the decision.

---

## Worked Solution

### Step 1: Identify the primary constraint

The DRAM PoP is above the top edge of the die, so the LPDDR PHY must be placed along the top die edge. The PHY requires 4 mm of the 10 mm top edge, leaving 6 mm for other uses.

### Step 2: Allocate die edges

Top edge (10 mm): LPDDR PHY (4 mm) + remaining (6 mm)
Left edge (10 mm): Available for other IO
Right edge (10 mm): Available for other IO
Bottom edge (10 mm): Available for other IO

The other IO blocks need: Display (2 mm) + Camera (1.5 mm) + PCIe/USB (2 mm) = 5.5 mm total.

Place these on the remaining edges:
- Bottom edge: PCIe/USB (2 mm) + Display (2 mm) = 4 mm used
- Left edge: Camera (1.5 mm) = 1.5 mm used

### Step 3: Place the LPDDR PHY

Centre the PHY on the top edge for symmetric bump map alignment:
- PHY occupies top edge from x=3 mm to x=7 mm (centred)
- PHY depth: from y=9.5 mm to y=10 mm (top 0.5 mm of the die)

### Step 4: Place the memory controller

The controller (1.5 mm x 0.8 mm) must be directly behind the PHY to minimise DFI routing:
- Controller at x=4.25 mm to x=5.75 mm, y=8.7 mm to y=9.5 mm
- This centres the controller behind the PHY with minimal DFI bus length (~0 mm, directly abutting)

### Step 5: Place the GPU and CPU with thermal consideration

The GPU (4 mm x 3 mm, 3W) and CPU (3 mm x 2 mm, 2W) are high-power blocks. Placing them adjacent to the PHY would create a thermal hotspot. Instead:

- GPU: Bottom-left area, x=0 to x=4 mm, y=0 to y=3 mm
- CPU: Bottom-right area, x=5 mm to x=8 mm, y=0 to y=2 mm

This separates the high-power blocks from the LPDDR PHY (which also dissipates 0.5-1W) and distributes heat across the die.

### Step 6: Verify the floorplan

```
 10mm |-LPDDR PHY (4mm)--|
      |____________________|
  9.5 |   | Mem Ctrl |     |
      |                    |
      |   [System          |
      |    Interconnect    |
      |    & misc logic]   |
      |                    |
   3  |GPU     |           |
      |        |   CPU     |
   0  |________|___________|
      0        5          10
```

### Step 7: Check thermal distribution

- LPDDR PHY: top centre, ~0.8W
- GPU: bottom-left, 3W
- CPU: bottom-right, 2W
- No two high-power blocks are adjacent
- Maximum thermal gradient is diagonal (PHY to GPU), ~14 mm apart

### Step 8: Check DFI routing

- DFI bus width: ~640 signals
- DFI routing distance: ~0 mm (controller directly behind PHY)
- DFI clock frequency: 800 MHz
- Maximum wire delay at 0 mm: negligible -- timing closure assured

### Step 9: Key decisions and rationale

1. **PHY centred on top edge:** Minimises asymmetry in bump routing to the PoP DRAM, and leaves space at the corners for power/ground bumps.
2. **Controller directly behind PHY:** Eliminates DFI routing delay as a timing concern.
3. **GPU and CPU separated from PHY:** Prevents thermal hotspot that could degrade LPDDR performance.
4. **GPU and CPU on opposite sides:** Distributes heat evenly across the die.

---

See also:
- [Floorplanning for LPDDR](../floorplanning_for_lpddr.md)
