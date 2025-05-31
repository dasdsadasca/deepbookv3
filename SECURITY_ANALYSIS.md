# DeepBookV3 Security Analysis

This document outlines potential security vulnerabilities identified in the DeepBookV3 smart contracts. It is a living document and will be updated as more analysis is performed.

## Calculation Vulnerabilities

This section details potential vulnerabilities related to arithmetic operations, such as overflows, underflows, precision loss, or division by zero, that could lead to incorrect state, loss of funds, or denial of service.

---

### **CALC-001: `Account::total_volume()` u128 Overflow**

*   **Module & Function**: `account.move` (within `pool` module) - `Account::total_volume()`
*   **Potential Issue**: The function calculates total volume by summing `self.base_volume_total` and `self.quote_volume_total_in_base_asset`, both of which are `u128`. If the sum `base_volume_total + quote_volume_total_in_base_asset` exceeds the maximum value for a `u128` integer, an arithmetic overflow will occur.
*   **Affected Variables/State**: The return value of `total_volume()`. This function is used in `history::calculate_rebate_amount` to determine user rebates.
*   **Potential Impact**: If an overflow occurs, the returned `total_volume` will be a much smaller, wrapped-around value. This would lead to an incorrect (likely much lower) rebate calculation for the user, potentially causing a loss of earned rebates.

---

### **CALC-002: `Account` Volume Accumulation u128 Overflow**

*   **Module & Function**: `account.move` (within `pool` module) - `Account::process_maker_fill()`, `Account::add_taker_volume()`
*   **Potential Issue**:
    *   In `process_maker_fill`, `self.base_volume_total` and `self.quote_volume_total_in_base_asset` are incremented.
    *   In `add_taker_volume`, `self.base_volume_total` and `self.quote_volume_total_in_base_asset` are incremented.
    These increments (`+=`) on `u128` typed fields can overflow if a user's accumulated volume becomes excessively large.
*   **Affected Variables/State**: `account.base_volume_total`, `account.quote_volume_total_in_base_asset` within the `Account` struct.
*   **Potential Impact**: An overflow would cause the user's recorded trading volume to wrap around to a much smaller value. This would impact any logic that relies on accurate total volume, such as fee tier calculations, rebate distributions, and potentially governance power if volume is a factor. Users might receive incorrect fee rates or lower rebates than deserved.

---

### **CALC-003: `Account::remove_stake()` u64 Overflow in DEEP Settlement**

*   **Module & Function**: `account.move` (within `pool` module) - `Account::remove_stake()`
*   **Potential Issue**: When calculating the total DEEP to return to the user, the function sums `active_stake + inactive_stake`. Both `active_stake` and `inactive_stake` are `u64`. If this sum `active_stake + inactive_stake` overflows the `u64` limit, the `total_deep_to_return` variable passed to `balances::add_deep` will be a smaller, wrapped-around value.
*   **Affected Variables/State**: The `amount` parameter in the subsequent `balances::add_deep()` call, and ultimately the amount of DEEP tokens returned to the user's `BalanceManager`.
*   **Potential Impact**: The user would receive fewer DEEP tokens than they are entitled to upon unstaking, leading to a direct loss of user funds (staked DEEP).

---

### **CALC-004: `Balances::add_balances` u64 Overflow**

*   **Module & Function**: `balances.move` (within `pool` module) - `Balances::add_balances()`, and by extension `add_base()`, `add_quote()`, `add_deep()`.
*   **Potential Issue**: The `add_balances` function, and the specific asset addition functions it calls (`add_base`, `add_quote`, `add_deep`), perform addition on `u64` fields (e.g., `self.base_total += amount`). If the `amount` being added to an existing total (e.g., `self.base_total`) causes the sum to exceed the `u64` maximum, an overflow will occur.
*   **Affected Variables/State**: `balances.base_total`, `balances.quote_total`, `balances.deep_total` within the `Balances` struct, which tracks the pool's overall asset totals. Also affects `account_balances.base_balance`, `account_balances.quote_balance`, `account_balances.deep_balance` within the `AccountBalances` struct, which tracks a user's balances within the `BalanceManager`.
*   **Potential Impact**:
    *   **Pool Accounting**: If `Pool::Balances` overflows, the pool's internal accounting of its total assets becomes incorrect. This could disrupt pool operations or lead to inconsistencies.
    *   **User Balances**: If `BalanceManager::AccountBalances` overflows during deposit or settlement, the user's recorded balance will be incorrect (lower than actual), potentially leading to a loss of funds for the user when they attempt to withdraw or use these balances.

---

### **CALC-005: `history::calculate_rebate_amount` Division by Zero**

*   **Module & Function**: `history.move` (within `pool` module) - `history::calculate_rebate_amount()`
*   **Potential Issue**: The function calculates rebates using `math::div_u128(..., volumes.historic_median, ...)`. If `volumes.historic_median` (which is a `u64` value representing the median trading volume) is zero, this will lead to a division by zero error.
*   **Affected Variables/State**: The rebate calculation process.
*   **Potential Impact**: A division by zero would cause the transaction to fail. If this occurs during a critical process like end-of-epoch rebate distribution, it could prevent all users from receiving their rebates for that epoch, leading to a denial of service for rebate claims. This might happen if a pool has very little or no trading activity for a period, causing the median volume to be calculated as zero.

---

### **CALC-006: `history::add_volume` u128 Overflows**

*   **Module & Function**: `history.move` (within `pool` module) - `history::add_volume()`
*   **Potential Issue**: This function updates several `u128` fields by addition:
    *   `current_epoch_history.total_volume += volume`
    *   `current_epoch_history.total_staked_volume += staked_volume`
    *   `global_history.total_volume += volume`
    *   `global_history.total_staked_volume += staked_volume`
    Any of these additions could overflow if the accumulated totals become excessively large.
*   **Affected Variables/State**: `total_volume` and `total_staked_volume` fields within `EpochHistory` and `GlobalHistory` structs.
*   **Potential Impact**: Overflowing these historical accumulators would lead to incorrect historical data. This could impact future calculations that rely on this history, such as average volumes, rebate calculations if they use global history, or any analytics/dashboarding features. The immediate impact might be less direct than fund loss but corrupts important long-term pool metrics.

---

### **CALC-007: `deep_price::add_price_point` Overflows/Underflows**

*   **Module & Function**: `deep_price.move` (within `pool` module) - `deep_price::add_price_point()`
*   **Potential Issue**:
    *   **Overflow**: The lines `state.cumulative_base_price += conversion_rate_base_per_quote;` and `state.cumulative_quote_price += conversion_rate_quote_per_base;` involve adding a `u64` `conversion_rate` to a `u64` cumulative price. This can overflow if the cumulative price becomes very large.
    *   **Underflow**: The lines `state.cumulative_base_price -= old_rate_base;` and `state.cumulative_quote_price -= old_rate_quote;` (when removing an old price point to maintain a window) involve subtracting a `u64` `old_rate` from a `u64` cumulative price. If `old_rate` is larger than the current `cumulative_price` (which shouldn't happen in normal operation but could due to other bugs or extreme data), this would underflow.
*   **Affected Variables/State**: `state.cumulative_base_price` and `state.cumulative_quote_price` within the `PriceOracleState` struct.
*   **Potential Impact**:
    *   **Overflow**: Would lead to an incorrect, wrapped-around cumulative price, making the oracle's time-weighted average price (TWAP) calculations completely wrong. This could be exploited by other protocols relying on this oracle for pricing.
    *   **Underflow**: Would also lead to a wildly incorrect cumulative price and thus an erroneous TWAP. This could similarly be exploited or cause malfunctions in systems relying on the oracle.
---

### **CALC-008: `Account::update()` u64 Stake Overflow**

*   **Module & Function**: `account.move` (within `pool` module) - `Account::update()`
*   **Potential Issue**: During an epoch change, this function calculates `let total_stake = self.active_stake + self.inactive_stake;`. Both `active_stake` and `inactive_stake` are `u64`. This sum can overflow if a user has a very large amount of combined active and inactive stake.
*   **Affected Variables/State**: The local variable `total_stake`. This value is subsequently used to update `self.active_stake = total_stake;` and `self.inactive_stake = 0;`.
*   **Potential Impact**: If `total_stake` overflows, `self.active_stake` will be set to an incorrect, smaller wrapped-around value. This would underrepresent the user's actual stake, impacting their voting power in governance and potentially their eligibility for or amount of staking rewards or rebates that are stake-dependent.

---

### **CALC-009: `Account::add_stake()` u64 Overflows**

*   **Module & Function**: `account.move` (within `pool` module) - `Account::add_stake()`
*   **Potential Issue**: This function has two potential `u64` overflows:
    1.  `let total_stake = self.active_stake + self.inactive_stake;` can overflow, as described in CALC-008.
    2.  `self.inactive_stake = self.inactive_stake + stake_amount;` can overflow if a user adds a large `stake_amount` to their existing `inactive_stake`.
*   **Affected Variables/State**:
    1.  The local `total_stake` variable, which might be used in subsequent logic within the function not shown in the provided snippet (e.g., checks against a maximum total stake).
    2.  `self.inactive_stake`.
*   **Potential Impact**:
    1.  If `total_stake` overflows and is used for validation (e.g., `assert!(total_stake + stake_amount <= MAX_USER_STAKE)`), the validation might pass incorrectly, potentially allowing a user to stake more than a theoretical limit if not for the overflow.
    2.  If `self.inactive_stake` overflows, the user's recorded inactive stake will be incorrect (lower), potentially leading to issues when this stake is meant to become active or is withdrawn. This could result in a loss of a portion of the staked amount or incorrect accounting for staking rewards.

---

### **CALC-010: `governance::adjust_vote` u64 Overflow/Underflow**

*   **Module & Function**: `governance.move` (within `pool` module) - `governance::adjust_vote()`
*   **Potential Issue**:
    *   **Underflow**: `proposal.votes_for -= votes_to_remove` or `proposal.votes_against -= votes_to_remove` can underflow if `votes_to_remove` (derived from user's previous vote power) is greater than the current `proposal.votes_for` or `proposal.votes_against`. This shouldn't happen in normal logic if a user is only removing their own previous vote, but could occur if state is inconsistent or if `votes_to_remove` is miscalculated.
    *   **Overflow**: `proposal.votes_for += votes_to_add` or `proposal.votes_against += votes_to_add` can overflow if the total votes for or against a proposal become extremely large.
*   **Affected Variables/State**: `proposal.votes_for`, `proposal.votes_against` within the `Proposal` struct.
*   **Potential Impact**:
    *   **Underflow**: Would cause a panic, potentially reverting the transaction. This could prevent users from changing their votes or unvoting if the proposal's vote counts are already low (possibly due to other users unvoting).
    *   **Overflow**: Would lead to incorrect vote counts for a proposal (wrapped to a smaller value). This could change the outcome of a governance vote, potentially allowing a proposal to pass or fail incorrectly.

---

### **CALC-011: `governance::adjust_voting_power` u64 Overflow/Underflow**

*   **Module & Function**: `governance.move` (within `pool` module) - `governance::adjust_voting_power()`
*   **Potential Issue**: This function modifies `self.voting_power` (a `u64`) by adding or subtracting `stake_amount`.
    *   `self.voting_power -= stake_amount` can underflow if `stake_amount` is greater than `self.voting_power`. This could happen if a user unstakes more than their current voting power reflects (e.g., due to partial unstaking or if voting power was already zero).
    *   `self.voting_power += stake_amount` can overflow if a user stakes a very large amount or if their accumulated voting power is already close to the `u64` limit.
*   **Affected Variables/State**: `governance_state.voting_power` within the `GovernanceState` struct (assuming `self` refers to `GovernanceState`).
*   **Potential Impact**:
    *   **Underflow**: Would cause a panic, reverting staking or unstaking operations. Could prevent users from unstaking if their voting power is somehow less than the amount they are trying to unstake.
    *   **Overflow**: Would result in the user having an incorrect (much lower) voting power, diminishing their influence in governance votes.

---

### **CALC-012: `history::calculate_rebate_amount` u64 Burn Tracking Overflow**

*   **Module & Function**: `history.move` (within `pool` module) - `history::calculate_rebate_amount()`
*   **Potential Issue**: The function calculates `let total_burn = balance_to_burn + maker_burn;`. Both `balance_to_burn` and `maker_burn` are `u64`. This sum can overflow if the combined burn amounts are very large.
*   **Affected Variables/State**: The local variable `total_burn`, which is subsequently added to `current_epoch_history.total_deep_burned`.
*   **Potential Impact**: If `total_burn` overflows, the amount added to `current_epoch_history.total_deep_burned` will be smaller than the actual amount burned during that calculation. This leads to an under-reporting of total DEEP burned in the epoch history, affecting the accuracy of burn tracking metrics. While not a direct loss of user funds, it corrupts economic data of the pool.

---

### **CALC-013: `history::update_historic_median` u128 Median Sum Overflow**

*   **Module & Function**: `history.move` (within `pool` module) - `history::update_historic_median()` calls `math::median<u128>()`
*   **Potential Issue**: The `math::median` function, when calculating the median for an even number of elements, involves summing two central elements: `(v[mid - 1] + v[mid]) / 2`. If `v` is a vector of `u128` (in this case, `historic_volumes` from the last N epochs), the sum `v[mid - 1] + v[mid]` can overflow the `u128` limit if two consecutive historic epoch volumes are exceptionally large.
*   **Affected Variables/State**: The calculation of `historic_median` within the `EpochHistory` struct.
*   **Potential Impact**: An overflow would lead to an incorrect (much smaller, wrapped-around) sum before the division by 2, resulting in a significantly wrong `historic_median` value. Since the `historic_median` is used in `calculate_rebate_amount` (see CALC-005), an incorrect median could lead to users receiving vastly incorrect rebate amounts (either too high, potentially draining rebate funds, or too low).

---
