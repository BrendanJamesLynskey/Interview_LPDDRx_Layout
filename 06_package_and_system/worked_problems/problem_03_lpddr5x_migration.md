# Worked Problem: LPDDR5X Migration

## Problem Statement

An existing LPDDR5 PHY layout (validated at 6400 MT/s) must be migrated to LPDDR5X at 8533 MT/s on the same process node (5nm). Identify the layout modifications needed, prioritise them, and estimate the effort.

---

## Worked Solution

### Step 1: Analyse the timing budget change

| Parameter | LPDDR5 (6400) | LPDDR5X (8533) | Change |
|---|---|---|---|
| UI (ps) | 156.25 | 117.19 | -25% |
| tDS + tDH (ps) | 100 (est.) | 90 (est.) | -10% |
| Available margin (ps) | 56.25 | 27.19 | -52% |
| Nyquist freq (MHz) | 3200 | 4267 | +33% |
| WCK freq (MHz) | 3200 | 4267 | +33% |

The available margin drops by 52%, which is severe. Every timing contributor must be re-evaluated.

### Step 2: Identify layout areas requiring modification

**Priority 1 -- Power Grid (High Impact, Moderate Effort):**
- 33% higher di/dt requires more decoupling
- Action: Add 50% more MIM capacitance in the IO region
- Action: Widen critical VDDQ/VSS straps by 20-30%
- Estimated effort: 2-3 weeks

**Priority 2 -- DQ/DQS Length Matching (High Impact, Low Effort):**
- Tighten matching from +/- 80 um to +/- 50 um
- Action: Re-tune serpentines on all DQ/DQS routes
- Action: Verify via count matching (must be exact)
- Estimated effort: 1-2 weeks

**Priority 3 -- WCK Routing (High Impact, Moderate Effort):**
- WCK at 4267 MHz is 33% faster than before
- Action: Add shielding on WCK differential pair
- Action: Tighten WCK/WCK_n matching to +/- 3 um
- Action: Verify impedance continuity along WCK route
- Estimated effort: 1-2 weeks

**Priority 4 -- PLL Modification (Medium Impact, High Effort):**
- PLL must generate higher WCK frequency
- Action: Update PLL block (may require circuit redesign)
- Action: Re-verify PLL isolation and decoupling
- Estimated effort: 4-8 weeks (if redesign needed)

**Priority 5 -- Crosstalk Mitigation (Medium Impact, Low Effort):**
- Higher frequency increases coupling
- Action: Verify inter-byte spacing is adequate
- Action: Add shield traces if crosstalk simulation shows violations
- Estimated effort: 1 week

### Step 3: SI/PI re-verification plan

1. Extract all DQ/DQS/CA/CK/WCK routing parasitics at the new frequency
2. Re-run eye diagram simulation at 8533 MT/s
3. Re-run PDN impedance analysis (target Z < 100 mohm at 4.27 GHz)
4. Re-run crosstalk simulation with all aggressors active
5. Verify training convergence range (delay line range must cover the new timing window)

### Step 4: Effort estimate summary

| Task | Priority | Effort (weeks) |
|---|---|---|
| Power grid enhancement | 1 | 2-3 |
| DQ/DQS re-matching | 2 | 1-2 |
| WCK routing upgrade | 3 | 1-2 |
| PLL update | 4 | 4-8 |
| Crosstalk mitigation | 5 | 1 |
| SI/PI re-verification | -- | 2-3 |
| **Total** | | **11-19 weeks** |

### Step 5: Risk assessment

- **Highest risk:** PLL may not achieve 4267 MHz WCK with acceptable jitter on the existing design. If circuit redesign is needed, this becomes the critical path.
- **Medium risk:** Power grid may not meet the tighter impedance target without adding more VDDQ bumps, which requires package modification.
- **Lowest risk:** Routing modifications (matching, shielding) are incremental changes that are well understood.

---

See also:
- [LPDDR5X and Beyond](../lpddr5x_and_beyond.md)
- [PHY Architecture](../../02_physical_interface/phy_architecture.md)
