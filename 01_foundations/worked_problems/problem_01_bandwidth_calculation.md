# Worked Problem: Bandwidth Calculation

## Problem Statement

Calculate the peak theoretical bandwidth for the following LPDDR configurations and compare them:

1. LPDDR4X at 4267 MT/s with a 32-bit (2 x 16-bit channel) interface
2. LPDDR5 at 6400 MT/s with a 32-bit (4 x 8-bit channel) interface
3. LPDDR5X at 8533 MT/s with a 32-bit (4 x 8-bit channel) interface

Also determine the bandwidth per pin for each configuration.

---

## Worked Solution

### Step 1: Understand the bandwidth formula

Peak bandwidth is calculated as:

```
Bandwidth (bytes/s) = Data Rate (MT/s) x Bus Width (bits) / 8 (bits/byte)
```

The data rate in MT/s (megatransfers per second) already accounts for double data rate (DDR) operation, so no additional factor of 2 is needed.

### Step 2: Calculate LPDDR4X bandwidth

```
Bus width = 32 bits (2 channels x 16 bits each)
Data rate = 4267 MT/s

Bandwidth = 4267 x 10^6 x 32 / 8
           = 4267 x 10^6 x 4
           = 17,068 x 10^6 bytes/s
           = 17.07 GB/s
```

Bandwidth per pin:
```
Per-pin bandwidth = 4267 MT/s / 8 bits per byte = 533.4 MB/s per pin
Total signal pins (DQ only) = 32
Verification: 32 x 533.4 MB/s = 17,068 MB/s = 17.07 GB/s (matches)
```

### Step 3: Calculate LPDDR5 bandwidth

```
Bus width = 32 bits (4 channels x 8 bits each)
Data rate = 6400 MT/s

Bandwidth = 6400 x 10^6 x 32 / 8
           = 6400 x 10^6 x 4
           = 25,600 x 10^6 bytes/s
           = 25.6 GB/s
```

Bandwidth per pin:
```
Per-pin bandwidth = 6400 / 8 = 800 MB/s per pin
Verification: 32 x 800 = 25,600 MB/s = 25.6 GB/s (matches)
```

### Step 4: Calculate LPDDR5X bandwidth

```
Bus width = 32 bits (4 channels x 8 bits each)
Data rate = 8533 MT/s

Bandwidth = 8533 x 10^6 x 32 / 8
           = 8533 x 10^6 x 4
           = 34,132 x 10^6 bytes/s
           = 34.13 GB/s
```

Bandwidth per pin:
```
Per-pin bandwidth = 8533 / 8 = 1066.6 MB/s per pin
Verification: 32 x 1066.6 = 34,131 MB/s ~ 34.13 GB/s (matches)
```

### Step 5: Summary comparison

| Parameter | LPDDR4X | LPDDR5 | LPDDR5X |
|---|---|---|---|
| Data rate (MT/s) | 4267 | 6400 | 8533 |
| Bus width (bits) | 32 | 32 | 32 |
| Channels | 2 x 16-bit | 4 x 8-bit | 4 x 8-bit |
| Peak bandwidth (GB/s) | 17.07 | 25.6 | 34.13 |
| Bandwidth per pin (MB/s) | 533.4 | 800.0 | 1066.6 |
| Improvement over LPDDR4X | 1.0x | 1.50x | 2.00x |

### Step 6: Layout implications

The bandwidth increase from LPDDR4X to LPDDR5X is 2x, but the pin count for DQ signals remains at 32. The higher per-pin bandwidth means each signal toggles faster, requiring:

- Tighter impedance control (higher frequency content demands lower reflection coefficients)
- More aggressive decoupling (higher di/dt on VDDQ supply)
- Better length matching (the UI shrinks from 468ps at 4267 MT/s to 234ps at 8533 MT/s, halving the absolute timing budget)

The shift from 2 x 16-bit channels to 4 x 8-bit channels increases the number of independent control signal groups (CA, CK, WCK) by 2x, requiring more routing resources and careful floorplanning.

---

See also:
- [LPDDR Evolution and Standards](../lpddr_evolution_and_standards.md)
- [Worked Problem: Generation Comparison](problem_02_generation_comparison.md)
