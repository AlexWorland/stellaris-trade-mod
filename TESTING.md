# Surplus Trade Mod — Test Plan

> **40 tests across 7 categories**
> All tests are manual, using Stellaris console commands.

---

## Console Command Quick Reference

```
debugtooltip                                    # enable variable tooltips
effect set_variable = { which = X value = Y }   # set country variable
effect add_resource = { energy = -999999 }       # drain resource to 0
cash [amount]                                    # set energy
minerals [amount]                                # set minerals
food [amount]                                    # set food
alloys [amount]                                  # set alloys
resource consumer_goods [amount]                 # set consumer goods
resource trade_value [amount]                    # set trade value
event st_events.1                                # fire monthly pulse directly
add_time months 1                                # advance 1 month
```

## Conversion Rate Cheat Sheet

| Resource | Sell (per unit to TV) | Buy (TV per unit) | Buy Chunk | TV per Chunk |
|----------|----------------------|-------------------|-----------|-------------|
| Energy | 0.85 | 1.18 | 200 units | 236 TV |
| Minerals | 0.85 | 1.18 | 200 units | 236 TV |
| Food | 0.85 | 1.18 | 200 units | 236 TV |
| Consumer Goods | 1.70 | 2.35 | 100 units | 235 TV |
| Alloys | 3.40 | 4.71 | 50 units | 236 TV |

## Sell Chunk Progression (per sell effect call)

Graduated chunks fire top-down via non-exclusive `if` blocks. One sell effect call can remove up to 18,800 units. The monthly pulse calls `st_sell_all_surplus` 5 times = 94,000 max per resource/month.

| Chunk | Energy/Min/Food TV | CG TV | Alloys TV |
|-------|--------------------|-------|-----------|
| 10000 | 8500 | 17000 | 34000 |
| 5000 | 4250 | 8500 | 17000 |
| 2000 | 1700 | 3400 | 6800 |
| 1000 | 850 | 1700 | 3400 |
| 500 | 425 | 850 | 1700 |
| 200 | 170 | 340 | 680 |
| 100 | 85 | 170 | 340 |

---

## Standard Test Setup

Before each test, drain all resources and set variables:
```
effect set_variable = { which = st_system_enabled value = 1 }
effect add_resource = { energy = -999999 }
effect add_resource = { minerals = -999999 }
effect add_resource = { food = -999999 }
effect add_resource = { consumer_goods = -999999 }
effect add_resource = { alloys = -999999 }
effect add_resource = { trade_value = -999999 }
```

Then set resource-specific variables/stockpiles per the test preconditions.

---

## Category 1: Setup & Configuration (8 tests)

### 1.1 `enable_system_default_disabled`
**Precondition:** Fresh game, mod installed, no manual variable changes
**Action:** `debugtooltip`, hover empire flag
**Expected:** `st_system_enabled = 0`. Advance 1 month — no resource changes.

### 1.2 `enable_system_via_decision`
**Precondition:** `effect set_variable = { which = st_system_enabled value = 0 }`
**Action:** Click "Enable Surplus Trade" decision
**Expected:** `st_system_enabled = 1`

### 1.3 `disable_system_via_decision`
**Precondition:** `st_system_enabled = 1`, energy above max threshold
**Action:** Click "Disable Surplus Trade", advance 1 month
**Expected:** `st_system_enabled = 0`. Energy unchanged (no sell despite being above max).

### 1.4 `cycle_minimum_tier_energy`
**Precondition:** `st_energy_min = 0`
**Action:** Click "Energy: Set Minimum" four times
**Expected:** Values cycle: 0 -> 100 -> 200 -> 500 -> 0

### 1.5 `cycle_desired_tier_minerals`
**Precondition:** `st_minerals_desired = 200`
**Action:** Click "Minerals: Set Desired" five times
**Expected:** Values cycle: 200 -> 500 -> 1000 -> 2500 -> 5000 -> 200

### 1.6 `cycle_maximum_tier_alloys`
**Precondition:** `st_alloys_max = 1000`
**Action:** Click "Alloys: Set Maximum" five times
**Expected:** Values cycle: 1000 -> 2500 -> 5000 -> 10000 -> 99999 (unlimited) -> 1000

### 1.7 `enable_single_resource_only`
**Precondition:** Energy enabled, all others disabled. Energy=2000, max=1000. Minerals=2000 (disabled, would be above any threshold).
**Action:** Advance 1 month
**Expected:** Energy sells surplus (-> 1000, TV increases). Minerals unchanged at 2000.

### 1.8 `constraint_min_less_than_desired`
**Precondition:** `st_energy_min = 500`, `st_energy_desired = 500`
**Action:** Attempt to set desired below min via cycling
**Expected:** Mod enforces min <= desired (either prevents or clamps). Document actual behavior.

---

## Category 2: Upstream SELL (8 tests)

### 2.1 `sell_basic_energy_surplus`
**Precondition:** Energy=1500, max=1000, TV=0
**Action:** `event st_events.1`
**Expected:** Energy=1000. TV=425 (500 x 0.85).
**How it works:** The 500-unit graduated chunk fires at threshold >= 1500.

### 2.2 `sell_exact_conversion_rate_energy`
**Precondition:** Energy=2000, max=1000, TV=0
**Action:** `event st_events.1`
**Expected:** Energy=1000. TV=850 (1000 x 0.85). Verify exact value, no rounding error.
**How it works:** The 1000-unit graduated chunk fires at threshold >= 2000.

### 2.3 `sell_alloys_at_higher_rate`
**Precondition:** Alloys=1100, max=1000, TV=0
**Action:** `event st_events.1`
**Expected:** Alloys=1000. TV=340 (100 x 3.40). Confirms 4x energy rate.
**How it works:** The 100-unit graduated chunk fires at threshold >= 1100.

### 2.4 `sell_consumer_goods_at_double_rate`
**Precondition:** CG=1500, max=1000, TV=0
**Action:** `event st_events.1`
**Expected:** CG=1000. TV=850 (500 x 1.70). Confirms 2x energy rate.

### 2.5 `no_sell_when_at_max`
**Precondition:** Energy=1000 (exactly at max=1000), TV=0
**Action:** `event st_events.1`
**Expected:** Energy=1000. TV=0. Boundary: at-max does NOT trigger sell.

### 2.6 `no_sell_when_below_max`
**Precondition:** Energy=999, max=1000, TV=0
**Action:** `event st_events.1`
**Expected:** Energy=999. TV=0.

### 2.7 `sell_multiple_resources_simultaneously`
**Precondition:** Energy=1500 (max=1000), Minerals=1200 (max=1000), Alloys=1100 (max=1000), TV=0
**Action:** `event st_events.1`
**Expected:** All three at 1000. TV = 425 + 170 + 340 = 935.

### 2.8 `sell_extremely_high_stockpile`
**Precondition:** Energy=50000, max=1000, TV=0
**Action:** `event st_events.1`
**Expected:** Energy=1000. TV=41650 (49000 x 0.85). Confirms no sell cap.
**How it works:** 5 iterations of `st_sell_all_surplus`, each draining graduated chunks. Fully drains in ~4 iterations.

---

## Category 3: Downstream BUY (7 tests)

### 3.1 `buy_basic_energy_deficit`
**Precondition:** Energy=0, desired=1000, TV=500
**Action:** `event st_events.1`
**Expected:** 2 buy iterations (2 x 236 = 472 TV). Energy=400. TV=28.

### 3.2 `buy_exact_conversion_rate`
**Precondition:** Energy=0, desired=1000, TV=236 (exactly 1 chunk)
**Action:** `event st_events.1`
**Expected:** Energy=200. TV=0.

### 3.3 `buy_alloys_at_higher_cost`
**Precondition:** Alloys=0, desired=500, TV=500
**Action:** `event st_events.1`
**Expected:** 2 buy iterations (50 alloys each, 236 TV each). Alloys=100. TV=28.

### 3.4 `no_buy_when_at_desired`
**Precondition:** Energy=500 (exactly at desired=500), TV=1000
**Action:** `event st_events.1`
**Expected:** Energy=500. TV=1000. At-desired does NOT trigger buy.

### 3.5 `no_buy_when_trade_zero`
**Precondition:** Energy=0, desired=1000, TV=0
**Action:** `event st_events.1`
**Expected:** Energy=0. TV=0. No negatives.

### 3.6 `partial_buy_insufficient_trade`
**Precondition:** Energy=0, desired=1000, TV=100 (less than half chunk of 118)
**Action:** `event st_events.1`
**Expected:** Energy=0. TV=100. Insufficient TV for even a half chunk.

### 3.7 `buy_caps_at_five_iterations`
**Precondition:** Energy=0, desired=5000, TV=2000 (enough for 8 chunks but capped at 5)
**Action:** `event st_events.1`
**Expected:** Energy=1000 (5 x 200). TV=820 (2000 - 5 x 236).

---

## Category 4: Gap Priority (8 tests)

### 4.1 `larger_gap_filled_first`
**Precondition:** Energy=200 (gap=800, Medium band), Minerals=0 (gap=1000, Large band). Both desired=1000. TV=472 (2 chunks).
**Action:** `event st_events.1`
**Expected:** Minerals gets both chunks (Large > Medium). Minerals=400. Energy=200. TV=0.

### 4.2 `tiebreaker_alloys_over_minerals`
**Precondition:** Alloys=0 (gap=500), Minerals=0 (gap=500). Both desired=500. Same gap band. TV=472 (2 chunks).
**Action:** `event st_events.1`
**Expected:** Alloys gets both (tiebreaker: Alloys > Minerals). Alloys=100. Minerals=0. TV=0.

### 4.3 `tiebreaker_consumer_goods_over_food`
**Precondition:** CG=0 (gap=500), Food=0 (gap=500). Both desired=500. TV=472.
**Action:** `event st_events.1`
**Expected:** CG gets priority (tiebreaker: CG > Food).

### 4.4 `critical_band_monopolizes_iterations`
**Precondition:** Energy=0 (gap=5000, Critical), Minerals=0 (gap=500, Medium). Energy desired=5000, Minerals desired=500. TV=1180 (5 chunks).
**Action:** `event st_events.1`
**Expected:** Energy gets all 5 iterations (Critical > Medium). Energy=1000. Minerals=0. TV=0.

### 4.5 `gap_recalculation_between_iterations`
**Precondition:** Energy=400 (gap=2100, Critical), Minerals=500 (gap=2000, Large). Both desired=2500. TV=2000.
**Action:** `event st_events.1`
**Expected:** Iteration 1: Energy (Critical band, gap=2100). After buy, Energy=600, gap=1900 (drops to Large). Iteration 2+: both in Large band, tiebreaker applies. Trace all 5 iterations.

### 4.6 `small_gap_starved_by_large_gap`
**Precondition:** Energy=0 (gap=5000, Critical), Food=100 (gap=500, Medium). TV=1180 (5 chunks).
**Action:** `event st_events.1`
**Expected:** Energy gets all 5. Food gets nothing.

### 4.7 `three_resources_priority_order`
**Precondition:** Alloys=1000 (gap=2000), Minerals=900 (gap=2100), Food=0 (gap=3000). All desired=3000. TV=2000.
**Action:** `event st_events.1`
**Expected:** All three in Critical band. Tiebreaker: Minerals > Food > (Alloys at boundary). Trace 5 iterations.

### 4.8 `gap_band_boundary_test`
**Precondition:** Set desired=2500, stockpile=500. Gap is exactly 2000.
**Action:** `event st_events.1`
**Expected:** Documents whether gap=2000 triggers Critical (>2000) or Large. The trigger uses `value < 500` for desired=2500 Critical band, so stockpile=500 does NOT trigger Critical (500 is not < 500). This means gap=2000 falls in Large, not Critical.

---

## Category 5: Minimum Floor (5 tests)

### 5.1 `sell_to_max_respects_min`
**Precondition:** Energy=1500, min=800, desired=900, max=1000
**Action:** `event st_events.1`
**Expected:** Energy=1000 (max). Min (800) not breached since max > min. TV=425.

### 5.2 `invalid_config_min_above_max`
**Precondition:** Via console: `st_energy_min = 500`, `st_energy_max = 300`. Energy=1000.
**Action:** `event st_events.1`
**Expected:** Document actual behavior — min enforcement is not implemented in the sell logic (sell checks max tier only). The runtime does NOT clamp min vs max.

### 5.3 `buy_does_not_overshoot_desired`
**Precondition:** Energy=800, desired=950 (gap=150, less than chunk size 200). TV=1000.
**Action:** `event st_events.1`
**Expected:** 950 is not a valid desired tier (tiers are 200/500/1000/2500/5000). Set desired=1000 instead. With desired=1000, gap=200, which is in the "Any deficit" band. Buy cycle buys a 200-chunk, Energy goes to 1000. If gap < smallest chunk, the half-chunk (100) may still overshoot — document actual behavior.

### 5.4 `min_zero_works`
**Precondition:** Energy=2000, min=0, max=1000
**Action:** `event st_events.1`
**Expected:** Energy=1000. Min=0 doesn't interfere.

### 5.5 `min_equals_desired`
**Precondition:** `st_energy_min = 500`, `st_energy_desired = 500`, `st_energy_max = 2500`. Energy=3000.
**Action:** `event st_events.1`
**Expected:** Energy=2500 (sells to max). No buy triggered (already at desired=500, stockpile 2500 > 500). TV=425.

---

## Category 6: Edge Cases (8 tests)

### 6.1 `all_at_desired_nothing_happens`
**Precondition:** All 5 resources at exactly their desired values. TV=1000.
**Action:** `event st_events.1`
**Expected:** All unchanged. TV unchanged. Zero mod activity.

### 6.2 `all_at_max_boundary`
**Precondition:** All 5 resources at exactly their max values. TV=0.
**Action:** `event st_events.1`
**Expected:** No sell triggered (at max, not above). TV=0.

### 6.3 `system_disabled_no_conversions`
**Precondition:** `st_system_enabled = 0`. Energy=5000 (above max). TV=0.
**Action:** `event st_events.1`
**Expected:** Energy=5000. TV=0. System disabled = no activity.

### 6.4 `resource_at_desired_plus_one`
**Precondition:** Energy=501, desired=500. TV=1000.
**Action:** `event st_events.1`
**Expected:** No buy (501 > 500). Energy=501. TV=1000.

### 6.5 `resource_at_max_plus_one`
**Precondition:** Energy=1001, max=1000. TV=0.
**Action:** `event st_events.1`
**Expected:** Sell triggered — but smallest graduated chunk (100) requires stockpile >= 1100. Since 1001 < 1100, no sell occurs. Energy remains 1001. Document this as a known limitation of the graduated chunk approach (excess < 100 is not sold until it accumulates further).

### 6.6 `trade_zero_multiple_deficits`
**Precondition:** All 5 resources at 0. All desired > 0. TV=0.
**Action:** `event st_events.1`
**Expected:** All remain at 0. TV=0. No negatives. No infinite loop.

### 6.7 `single_resource_enabled_others_disabled`
**Precondition:** Only energy enabled (`st_energy_enabled = 1`, all others `= 0`). Minerals above its (disabled) max, food below its (disabled) desired.
**Action:** `event st_events.1`
**Expected:** Only energy conversions occur. Minerals and food unchanged.

### 6.8 `no_negative_resources_ever`
**Precondition:** Energy=0, all others at 0, TV=0. All enabled with desired > 0.
**Action:** `event st_events.1` three times
**Expected:** No resource goes negative. TV never goes negative.

---

## Category 7: Full Integration (4 tests)

### 7.1 `full_cycle_sell_energy_buy_minerals`
**Precondition:** Energy=2000 (max=1000). Minerals=0 (desired=1000). TV=0.
**Action:** `event st_events.1`
**Expected:**
- Sell: 1000 energy -> 850 TV. Energy=1000.
- Buy: 3 chunks of minerals (3 x 236 = 708 TV). Minerals=600. TV=142.

### 7.2 `multi_month_convergence`
**Precondition:** Energy=5000 (max=2000). Minerals=0 (desired=2000). TV=0.
**Action:** Advance 5 months, record state each month
**Expected:** Minerals steadily increases toward 2000. Document per-month values.

### 7.3 `all_five_resources_different_configs`
**Precondition:**
- Energy=2000, max=1000 (sells 1000, +850 TV)
- Minerals=200, desired=800 (deficit 600, Medium band)
- Food=1000, max=500 (sells 500, +425 TV)
- CG=0, desired=600 (deficit 600, Medium band — but 600 is not a valid tier, use desired=500 or 1000)
- Alloys=3000, max=2000 (sells 1000, +3400 TV)
- TV=0

**Action:** `event st_events.1`
**Expected:**
- Total TV from sells: 850 + 425 + 3400 = 4675
- Buy phase: 5 iterations distributed by gap priority among minerals and CG
- Trace each iteration

### 7.4 `steady_state_convergence`
**Precondition:** Continuous energy surplus (production pushes above max each month). Minerals deficit.
**Action:** Advance 10+ months
**Expected:** Minerals converges to desired and stabilizes. System reaches equilibrium.

---

## Installation for Testing

1. Copy the mod folder contents to: `~/Documents/Paradox Interactive/Stellaris/mod/surplus_trade/`
2. Copy `surplus_trade.mod` to: `~/Documents/Paradox Interactive/Stellaris/mod/`
3. Launch Stellaris, enable the mod in the launcher
4. Start a new game (required for `on_game_start_country` to fire)
5. Open console with `~`, run `debugtooltip`
6. Verify `st_system_enabled = 0` on empire tooltip
