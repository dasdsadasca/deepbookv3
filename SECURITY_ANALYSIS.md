# DeepBookV3 Security Analysis

This document provides a summary of potential security vulnerabilities identified in the DeepBookV3 smart contracts. Detailed deep-dive reports for critical and high-severity issues are available in separate linked files. This summary will be updated as more analysis is performed.

## Calculation Vulnerabilities

This section details potential vulnerabilities related to arithmetic operations, such as overflows, underflows, precision loss, or division by zero, that could lead to incorrect state, loss of funds, or denial of service.

---

### **CALC-001: `Account::total_volume()` u128 Overflow**

*   **Module & Function**: `account.move` (within `pool` module) - `Account::total_volume()`
*   **Potential Issue**: The function calculates total volume by summing `self.base_volume_total` and `self.quote_volume_total_in_base_asset`, both of which are `u128`. If the sum exceeds `u128::MAX`, an overflow occurs.
*   **Affected Variables/State**: Return value of `total_volume()`.
*   **Severity**: **Low Severity / Not Practically Exploitable**
*   **Analysis & Exploit Path Confirmation**: While `base_volume_total + quote_volume_total_in_base_asset` can theoretically overflow `u128`, the required cumulative volume for a single account within a single epoch (e.g., `> 3.4e38` if assets are scaled by `1e9`) is practically unattainable with current blockchain transaction limits and typical asset scales. An overflow would cause a panic due to Move's default arithmetic checks.
*   **Impact (Theoretical)**: Panic during `total_volume()` call if it were to overflow, potentially affecting fee tier calculations or other logic that might use it, leading to transaction reversion.
*   **Mitigations**: The sheer size of `u128` is the primary mitigating factor against practical exploits. Standard panic on overflow prevents silent data corruption.

---

### **CALC-002: `Account` Volume Accumulation u128 Overflow**

*   **Module & Function**: `account.move` (within `pool` module) - `Account::process_maker_fill()`, `Account::add_taker_volume()`
*   **Potential Issue**: Increments (`+=`) to `self.base_volume_total` or `self.quote_volume_total_in_base_asset` (both `u128`) can overflow.
*   **Affected Variables/State**: `account.base_volume_total`, `account.quote_volume_total_in_base_asset`.
*   **Severity**: **Low Severity / Not Practically Exploitable**
*   **Analysis & Exploit Path Confirmation**: Similar to CALC-001, a single account accumulating enough trading volume (base or quote, scaled) within one epoch to overflow a `u128` field is practically infeasible.
*   **Impact (Theoretical)**: Panic during the volume update operation (e.g., in `process_maker_fill` or `add_taker_volume`), reverting the transaction that caused the overflow. This would prevent the trade from being recorded for that user if it hit the theoretical limit.
*   **Mitigations**: The size of `u128`. Standard panic on overflow.

---

### **CALC-003: `Account::remove_stake()` - Stake Summation `u64` Overflow Leading to Fund Freeze**

*   **Module & Function**: `deepbook::state::account::remove_stake`
*   **Potential Issue**: Sum `self.active_stake + self.inactive_stake` (`u64`) panics if it exceeds `u64::MAX`.
*   **Severity**: **Critical**
*   **Detailed Analysis**: Refer to the dedicated detailed report: `CALC-003_Stake_Sum_Overflow_Fund_Freeze.md` for a comprehensive analysis, exploit scenarios, and mitigation strategies.

---

### **CALC-004: `Balances::add_balances` (and helpers) u64 Overflow leading to DoS for Account operations**

*   **Module & Function**: `deepbook::balances::add_balances` (and helpers); called by various `deepbook::state::account` functions.
*   **Potential Issue**: Direct `u64` additions to `Balances` fields (used in `Account` for `settled_balances`, etc.) panic on overflow.
*   **Severity**: **High/Medium**
*   **Detailed Analysis**: Refer to the dedicated detailed report: `CALC-004_Balance_Overflow_DoS.md` for a comprehensive analysis, exploit scenarios, and mitigation strategies.

---

### **CALC-005: `history::calculate_rebate_amount` Division by Zero**

*   **Module & Function**: `history.move` (within `pool` module) - `history::calculate_rebate_amount()`
*   **Potential Issue**: The function calculates rebates using `math::div_u128(..., volumes.historic_median, ...)`. If `volumes.historic_median` (which is a `u64` value representing the median trading volume) is zero, this could lead to a division by zero error.
*   **Affected Variables/State**: The rebate calculation process, `maker_rebate_percentage`.
*   **Potential Impact (if not mitigated)**: A division by zero would cause the transaction to fail. If this occurs during a critical process like end-of-epoch rebate distribution, it could prevent all users from receiving their rebates for that epoch, leading to a denial of service for rebate claims.
*   **Status**: **Mitigated**
*   **Mitigation Explanation**: The function includes a specific check: `if (volumes.historic_median > 0)`. If `volumes.historic_median` is 0, `maker_rebate_percentage` is explicitly set to 0. This avoids the division by zero and correctly results in zero rebates being calculated, preventing a panic and ensuring the function behaves as intended in a zero median volume scenario.

---

### **CALC-006: `history::add_volume` u128 Overflows**

*   **Module & Function**: `history.move` (within `pool` module) - `history::add_volume()`
*   **Potential Issue**: This function updates `u128` fields like `current_epoch_history.total_volume`, `current_epoch_history.total_staked_volume`, etc., by addition. These additions could theoretically overflow.
*   **Affected Variables/State**: `total_volume` and `total_staked_volume` fields within `EpochHistory` and `GlobalHistory` structs.
*   **Severity**: **Low Severity / Not Practically Exploitable**
*   **Analysis & Exploit Path Confirmation**: Accumulating `total_volume` or `total_staked_volume` for an entire pool to overflow `u128` within a single epoch (or even across many epochs for global history, though epochs reset `current_epoch_history`) is practically infeasible given transaction limits and realistic market volumes. The value `u128::MAX` is astronomically large.
*   **Impact (Theoretical)**: Panic during the `add_volume` call if an overflow were to occur, reverting the transaction. This could potentially disrupt the recording of historical data for that transaction, but the conditions for such an overflow are not realistic.
*   **Mitigations**: The size of `u128`. Standard panic on overflow.

---

### **CALC-007: `deep_price::add_price_point` - `cumulative_base/quote` `u64` Overflow/Underflow and Pruning Logic**

*   **Module & Function**: `deepbook::deep_price::add_price_point`
*   **Potential Issue**: `u64` `cumulative_base`/`quote` can overflow on addition of new `conversion_rate` or underflow on subtraction of old `conversion_rate` during pruning, leading to panics.
*   **Severity**: **High**
*   **Detailed Analysis**: Refer to the dedicated detailed report: `CALC-007_Oracle_Price_Manipulation.md` for a comprehensive analysis, exploit scenarios, and mitigation strategies.

---

### **CALC-008: `Account::update()` u64 Stake Overflow**

*   **Module & Function**: `account.move` (within `pool` module) - `Account::update()`
*   **Potential Issue**: During an epoch change, this function calculates `let total_stake = self.active_stake + self.inactive_stake;`. Both `active_stake` and `inactive_stake` are `u64`. This sum can overflow if a user has a very large amount of combined active and inactive stake.
*   **Affected Variables/State**: The local variable `total_stake`. This value is subsequently used to update `self.active_stake = total_stake;` and `self.inactive_stake = 0;`.
*   **Potential Impact**: If the sum `self.active_stake + self.inactive_stake` overflows `u64`, the transaction panics. This leads to a **DoS for epoch updates for this specific user's account**. Their stake cannot be rolled over from inactive to active. Incorrect accounting (due to wrap-around) is prevented by the panic.
*   **Relation to CALC-004/CALC-003**: This is a specific instance where a sum (`active_stake + inactive_stake`) overflows. The core risk of `u64` for stake fields is covered by CALC-004. Unlike CALC-003 (fund freeze in `remove_stake`), the immediate impact here is DoS for the `Account::update` operation for that user.
*   **Mitigations**: `Account::active_stake` and `Account::inactive_stake` should be `u128`, and the sum performed into a `u128` local variable.

---

### **CALC-009: `Account::add_stake()` u64 Overflows**

*   **Module & Function**: `account.move` (within `pool` module) - `Account::add_stake()`
*   **Potential Issue**: This function has potential `u64` overflows:
    1.  `let total_stake = self.active_stake + self.inactive_stake;` can panic if the sum exceeds `u64::MAX`.
    2.  `self.inactive_stake = self.inactive_stake + stake_amount;` can panic if this sum exceeds `u64::MAX`.
    3.  The call `self.owed_balances.add_deep(stake_amount)` can panic if `owed_balances.deep + stake_amount` exceeds `u64::MAX` (this is an instance of CALC-004).
*   **Affected Variables/State**: `self.inactive_stake`, `self.owed_balances.deep_balance`, user's ability to stake.
*   **Potential Impact**: Panics if sums like `active_stake + inactive_stake` or `inactive_stake + stake_amount` overflow `u64`. This leads to **DoS for new staking operations (CALC-009) or for epoch updates for the user (related to CALC-008 logic if sum overflows there)**. Incorrect accounting due to wrap-around is prevented by the panic.
*   **Mitigations**: Stake fields (`active_stake`, `inactive_stake`) and intermediate sums should use `u128`. `Balances::deep` (in `owed_balances`) should also be `u128` (see CALC-004).

---

### **CALC-010: `governance::adjust_vote` u64 Overflow/Underflow for `proposal.votes`**

*   **Module & Function**: `deepbook::state::governance::adjust_vote`
*   **Potential Issue**: `u64` `Proposal::votes` field can overflow on addition or underflow on subtraction during vote casting/changing, leading to panics.
*   **Severity**: **Medium-High**
*   **Detailed Analysis**: Refer to the dedicated detailed report: `CALC-010_011_Governance_Overflow_DoS.md` for a comprehensive analysis, exploit scenarios, and mitigation strategies.

---

### **CALC-011: `governance::adjust_voting_power` u64 Overflow/Underflow for `self.voting_power`**

*   **Module & Function**: `deepbook::state::governance::adjust_voting_power`
*   **Potential Issue**: `u64` `Governance::voting_power` field can overflow or underflow during updates based on user stake changes, leading to panics.
*   **Severity**: **High**
*   **Detailed Analysis**: Refer to the dedicated detailed report: `CALC-010_011_Governance_Overflow_DoS.md` for a comprehensive analysis, exploit scenarios, and mitigation strategies.

---

### **CALC-012: `history::calculate_rebate_amount` u64 Burn Tracking Overflow**

*   **Module & Function**: `history.move` (within `pool` module) - `history::calculate_rebate_amount()`
*   **Potential Issue**: The function calculates `let total_burn = balance_to_burn + maker_burn;`. Both `balance_to_burn` and `maker_burn` are `u64`. This sum can overflow if the combined burn amounts are very large.
*   **Affected Variables/State**: The local variable `total_burn`, which is subsequently added to `current_epoch_history.total_deep_burned`.
*   **Potential Impact**: If `total_burn` overflows, the amount added to `current_epoch_history.total_deep_burned` will be smaller than the actual amount burned during that calculation. This leads to an under-reporting of total DEEP burned in the epoch history, affecting the accuracy of burn tracking metrics. While not a direct loss of user funds, it corrupts economic data of the pool.

---

### **CALC-013: `history::update_historic_median` u128 Median Sum Overflow**

*   **Module & Function**: `history.move` (within `pool` module) - `history::update_historic_median()` calls `math::median<u128>()`.
*   **Potential Issue**: The `math::median` function, when calculating the median for an even number of elements, involves summing two central elements: `(v[mid - 1] + v[mid]) / 2`. If `v` is a vector of `u128` (historic epoch total volumes), this sum could theoretically overflow `u128`.
*   **Affected Variables/State**: The calculation of `historic_median` within `GlobalHistory` (as `update_historic_median` is called on `global_history`).
*   **Severity**: **Low Severity / Not Practically Exploitable**
*   **Analysis & Exploit Path Confirmation**: For the sum `sorted_v[n / 2 - 1] + sorted_v[n / 2]` to overflow `u128`, both `sorted_v[n / 2 - 1]` and `sorted_v[n / 2]` (which are individual epoch total volumes) would need to be exceedingly large, specifically each approaching `u128::MAX / 2`. As established in CALC-006, it's practically impossible for the total volume of a single epoch to reach such magnitudes. Therefore, the sum of two such epoch volumes overflowing `u128` is also practically impossible.
*   **Impact (Theoretical)**: If an overflow were to occur, it would cause a panic during the median calculation at epoch turnover (when `update_historic_median` is called). This could halt updates to `global_history.historic_median`, potentially affecting any logic that relies on it (e.g., if future rebate calculations were to use this global median).
*   **Mitigations**: The primary mitigation is the sheer size of `u128`, making the prerequisite individual epoch volumes practically unattainable. A theoretically safer summation method like `a/2 + b/2` (with appropriate scaling if precision is needed) is possible but likely an over-optimization given the practical infeasibility. Standard panic on overflow.

---

### **CALC-014: `deep_price::calculate_order_deep_price` Division by Zero**

*   **Module & Function**: `deepbook::deep_price::calculate_order_deep_price`
*   **Potential Issue**: Potential division by zero in the calculation `deep_per_asset = cumulative_asset / asset_length` if `asset_length` could be zero.
*   **Affected Variables/State**: The return value `deep_per_asset`.
*   **Potential Impact (if not mitigated)**: A division by zero would cause a panic, leading to a Denial of Service (DoS) for any operation that tries to calculate the DEEP price oracle (e.g., fee calculations).
*   **Status**: **Mitigated by existing checks**
*   **Analysis & Mitigation Explanation**:
    *   The function first asserts `assert!(self.last_insert_timestamp(true) > 0 || self.last_insert_timestamp(false) > 0, Errors::ENoDataPoints)`, which ensures that at least one of the price vectors (`base_prices` or `quote_prices`) has had data inserted at some point and is non-empty.
    *   The logic to determine which asset's cumulative price to use is: `let is_base_conversion = self.last_insert_timestamp(false) == 0 || (self.last_insert_timestamp(true) > 0 && self.last_insert_timestamp(true) > self.last_insert_timestamp(false));` (Note: The actual code uses this logic to determine which price to use if both exist, ensuring it picks one that has data).
    *   If `is_base_conversion` is true (meaning `base_prices` will be used):
        *   `cumulative_asset = self.cumulative_base();`
        *   `asset_length = self.base_prices.length();`
        *   The `ENoDataPoints` assert ensures `base_prices` is non-empty if it's selected (especially if `quote_prices` is empty or its last timestamp is older). The `add_price_point` logic ensures that if `base_prices` has entries, its length is `> 0` (it prunes to `MAX_DATA_POINTS` but doesn't empty a list that was full). Thus, `asset_length > 0`.
    *   If `is_base_conversion` is false (meaning `quote_prices` will be used):
        *   `cumulative_asset = self.cumulative_quote();`
        *   `asset_length = self.quote_prices.length();`
        *   Similarly, the `ENoDataPoints` assert ensures `quote_prices` is non-empty if it's selected (because `last_insert_timestamp(false)` would be `> 0`). Thus, `asset_length > 0`.
    *   The `ENoDataPoints` assert, combined with the logic for choosing which price vector's data to use, effectively ensures that `asset_length` will be greater than zero because an empty vector would not be chosen if a non-empty one exists, and the assert guarantees at least one is non-empty.

---
