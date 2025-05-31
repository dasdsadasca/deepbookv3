# DeepBookV3 Security Analysis

This document outlines potential security vulnerabilities identified in the DeepBookV3 smart contracts. It is a living document and will be updated as more analysis is performed.

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

### **CALC-003: `Account::remove_stake()` u64 Overflow in DEEP Settlement**

*   **Module & Function**: `account.move` (within `pool` module) - `Account::remove_stake()`
*   **Potential Issue**: When calculating the total DEEP to return to the user, the function sums `active_stake + inactive_stake`. Both `active_stake` and `inactive_stake` are `u64`. If this sum `active_stake + inactive_stake` overflows the `u64` limit, the `total_deep_to_return` variable passed to `balances::add_deep` will be a smaller, wrapped-around value.
*   **Affected Variables/State**: The `amount` parameter in the subsequent `balances::add_deep()` call, and ultimately the amount of DEEP tokens returned to the user's `BalanceManager`.
*   **Potential Impact**: The user would receive fewer DEEP tokens than they are entitled to upon unstaking, leading to a direct loss of user funds (staked DEEP).

---

### **CALC-004: `Balances::add_balances` u64 Overflow (Core Issue for Stake Accumulation)**

*   **Module & Function**: `deepbook::balances::add_balances` (and helpers `add_base`, `add_quote`, `add_deep`). Also impacts multiple functions in `deepbook::account` that call these (e.g., `process_maker_fill`, `add_settled_balances`, `add_owed_balances`, `claim_rebates`, `add_stake`, `remove_stake`).
*   **Potential Issue**: Direct `u64 + u64` addition for `base`, `quote`, or `deep` fields within the `Balances` struct can overflow if the sum exceeds `u64::MAX`. This is a general issue affecting many balance update operations.
*   **Affected Variables/State**: `Balances::base_total`, `Balances::quote_total`, `Balances::deep_total`. Also `AccountBalances` fields within `BalanceManager`, and `Account::active_stake`, `Account::inactive_stake`, `Account::settled_balances`, `Account::owed_balances`.
*   **Exploit Path Confirmation & Impact**:
    *   **General DoS**: If any balance component (e.g., `settled_balances.base` in `Account`) is close to `u64::MAX`, and a transaction attempts to add even a small amount (e.g., via `add_base`) that would cause an overflow, the transaction will panic (due to Move's default overflow checks on `+`). This can lead to a Denial of Service for the affected user for operations that try to update these balances (e.g., receiving fills, claiming rebates, staking). The user might be unable to interact further with the pool for that asset or operation.
    *   **CALC-003 (Fund Loss via `Account::remove_stake` - Specific instance of CALC-004's broader issue)**:
        *   **Exploit**:
            1. A user's `active_stake` and `inactive_stake` in `Account` struct are both `u64`.
            2. In `account::remove_stake()`, `stake_before = self.active_stake + self.inactive_stake;` is calculated. If `active_stake` is, for example, `u64::MAX - 100` and `inactive_stake` is `200`, their sum overflows `u64` and wraps around to a small value (e.g., `99`).
            3. This wrapped-around, small `stake_before` value is then passed to `self.settled_balances.add_deep(stake_before)`.
            4. The `Balances::add_deep` function itself might not overflow here if `settled_balances.deep` was small, but it's adding the *wrong, much smaller* amount to the user's settled DEEP balance.
        *   **Ultimate Impact (CALC-003)**: When the user subsequently withdraws their funds from `BalanceManager` (which reads from these `settled_balances`), the `Vault` will only transfer this small, wrapped-around DEEP amount from `settled_balances` to their `BalanceManager`. This results in a **direct and permanent loss of the majority of the user's staked DEEP tokens**. Any user whose total stake (`active + inactive`) approaches or exceeds `u64::MAX` is vulnerable when calling `remove_stake`.
        *   **Attacker Capability (CALC-003)**: This can be triggered by the user themselves when unstaking if their stake is sufficiently large. An attacker cannot directly cause this for another user unless they can influence the victim's stake amounts to reach overflow conditions prior to the victim calling `remove_stake`.
*   **Mitigations**:
    *   Move's panic on overflow is a default safety feature preventing silent corruption, but it leads to DoS in many cases.
    *   **Missing/Insufficient**: The `base`, `quote`, and `deep` fields within the `Balances` struct (and consequently fields like `active_stake`, `inactive_stake` in `Account` that are summed up before being added to `Balances`) should ideally be `u128` to significantly reduce the likelihood of overflow for token amounts. For sums like `active_stake + inactive_stake` before they are used with `Balances` functions (as in `remove_stake`), the sum should be performed into a `u128` temporary variable to prevent overflow before further processing or before being passed to a `Balances` function that expects a `u64` amount (if `Balances` fields remain `u64`).

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
*   **Affected Variables/State**: `DeepPrice::cumulative_base` (u64), `DeepPrice::cumulative_quote` (u64), and subsequently the calculated `deep_per_asset` oracle price.

*   **Issue 1: Overflow of `cumulative_base`/`cumulative_quote`**
    *   **Description**: `self.cumulative_base = self.cumulative_base + conversion_rate;` (similarly for quote) can overflow `u64`.
    *   **Verification**: Confirmed. `conversion_rate` can be a large `u64` (up to `~10^18` or `~0.9*10^19` based on `constants::max_price()`). Summing even 2-3 such large values, or ~18 values of `10^18`, will exceed `u64::MAX (~1.8*10^19)`. This is possible within the `MAX_DATA_POINTS` (100) window.
    *   **Exploit Path**:
        1.  Attacker manipulates a whitelisted `reference_pool` to make its `mid_price` such that the derived `conversion_rate` for `add_price_point` is very high (e.g., close to `u64::MAX / k` where `k` is a small integer like 2 or 3).
        2.  Attacker or any user calls `pool::add_deep_price_point` for the target pool repeatedly (respecting the 1-minute `MIN_DURATION_BETWEEN_DATA_POINTS_MS`).
        3.  After `k` such calls, `cumulative_base` (or `quote`) overflows and wraps to a small value.
        4.  `calculate_order_deep_price` then computes `deep_per_asset = small_wrapped_cumulative / asset_prices.length()`, resulting in an artificially very low oracle price.
    *   **Impact**: **High Severity**. Leads to near-zero DEEP fee calculations for trades in the target pool, causing loss of protocol revenue.
    *   **Mitigations**:
        *   `MIN_DURATION_BETWEEN_DATA_POINTS_MS` (1 minute) slows the attack but does not prevent it.
        *   `MAX_DATA_POINTS` (100) & `MAX_DATA_POINT_AGE_MS` (e.g., 1 day) mean old malicious data points will eventually be pruned, allowing the average to self-correct if manipulation stops.
        *   Reference pool whitelisting provides some trust but doesn't prevent manipulation of a whitelisted pool's market if the whitelisted pool itself is vulnerable or thinly traded.
        *   **Insufficient**: `cumulative_base` and `cumulative_quote` should be `u128` to make overflow from summing `MAX_DATA_POINTS` (100) `u64` values practically impossible.

*   **Issue 2: Underflow of `cumulative_base`/`cumulative_quote` during Pruning**
    *   **Description**: `self.cumulative_base = self.cumulative_base - asset_prices[0].conversion_rate;` (similarly for quote) can panic if `asset_prices[0].conversion_rate` is greater than the current `self.cumulative_base`.
    *   **Verification**: Confirmed. This can occur if a very large historical price point (`P_high`) is at the head of the `asset_prices` vector (due to be pruned by age or vector size limit) and the `cumulative_base` has become small due to subsequent additions of very small price points (`P_low`) or pruning of other initial large values.
    *   **Exploit Path**:
        1.  Attacker adds one or more `P_high` price points (large `conversion_rate`).
        2.  Attacker then adds multiple `P_low` price points (e.g., `conversion_rate` = 1) until `P_high` is at `asset_prices[0]`. The `cumulative_base` would be approximately `P_high + sum_of_some_P_lows - sum_of_any_other_pruned_values`.
        3.  Attacker triggers another `add_price_point` when `P_high` is eligible for pruning (either `asset_prices` is full at `MAX_DATA_POINTS`, or `P_high`'s timestamp is older than `MAX_DATA_POINT_AGE_MS`).
        4.  If current `cumulative_base` is less than `P_high` (plausible if `P_high` was very large and many subsequent prices were small, or other initial large values were already pruned), the subtraction `self.cumulative_base - P_high` panics.
    *   **Impact**: **High Severity**. Causes `pool::add_deep_price_point` to panic, leading to a Denial of Service for oracle updates for that asset (base or quote). This results in a stale oracle price and inaccurate fees.
    *   **Mitigations**:
        *   Move's default panic on underflow prevents silent corruption.
        *   **Insufficient**: Using `u128` for `cumulative_base` and `cumulative_quote` would make it much harder for a single old data point to be larger than the cumulative sum of up to 100 `u64` data points. A saturating subtraction (`saturating_sub`) would prevent the panic but could lead to `cumulative_base` becoming 0 if `P_high` is larger, which would significantly skew the average (though perhaps preferable to a DoS). A larger type for cumulative sums is the more robust fix.

*   **Note on LIV-001 (Pruning Loop Panic from Empty Vector Access)**: The `while` loop condition for pruning is `asset_prices.length() > MAX_DATA_POINTS || (asset_prices.length() > 0 && asset_prices[0].timestamp + MAX_DATA_POINT_AGE_MS < timestamp)`. The `asset_prices.length() > 0` check before `asset_prices[0]` access acts as a short-circuit guard. Therefore, the specific panic described in LIV-001 (accessing index 0 of an empty vector *within the loop condition itself*) is **prevented by this short-circuiting logic**. The primary remaining risk during pruning is the underflow described in "Issue 2" above. LIV-001 can be considered superseded/covered by this analysis of CALC-007.

*   **Overall Attacker Requirements for CALC-007**: Ability to significantly influence the `mid_price` of a whitelisted, registered reference pool over a period of minutes to hours to feed extreme `conversion_rate` values into the oracle.

---

### **CALC-008: `Account::update()` u64 Stake Overflow**

*   **Module & Function**: `account.move` (within `pool` module) - `Account::update()`
*   **Potential Issue**: During an epoch change, this function calculates `let total_stake = self.active_stake + self.inactive_stake;`. Both `active_stake` and `inactive_stake` are `u64`. This sum can overflow if a user has a very large amount of combined active and inactive stake.
*   **Affected Variables/State**: The local variable `total_stake`. This value is subsequently used to update `self.active_stake = total_stake;` and `self.inactive_stake = 0;`.
*   **Potential Impact**: If `total_stake` overflows, `self.active_stake` will be set to an incorrect, smaller wrapped-around value. This would underrepresent the user's actual stake, impacting their voting power in governance and potentially their eligibility for or amount of staking rewards or rebates that are stake-dependent.
*   **Relation to CALC-004/CALC-003**: This is a specific instance where a sum (`active_stake + inactive_stake`) overflows before being assigned. The core risk of `u64` for stake fields is covered by CALC-004, and the direct fund loss from such a sum overflowing during `remove_stake` is detailed in CALC-003. If this overflow in `update()` doesn't lead to a CALC-003 scenario elsewhere, the impact is primarily incorrect stake accounting affecting governance/rebates, or DoS if subsequent operations panic.

---

### **CALC-009: `Account::add_stake()` u64 Overflows**

*   **Module & Function**: `account.move` (within `pool` module) - `Account::add_stake()`
*   **Potential Issue**: This function has potential `u64` overflows:
    1.  `let total_stake = self.active_stake + self.inactive_stake;` can overflow.
    2.  `self.inactive_stake = self.inactive_stake + stake_amount;` can overflow.
    3.  The call `self.owed_balances.add_deep(stake_amount)` can overflow if `owed_balances.deep` is already large (this is an instance of CALC-004).
*   **Affected Variables/State**: `self.inactive_stake`, `self.owed_balances.deep_balance`.
*   **Potential Impact**:
    1.  Overflow of `total_stake`: Incorrect validation if used for pre-check against a max total stake.
    2.  Overflow of `self.inactive_stake`: Incorrect accounting of inactive stake.
    3.  Overflow in `add_deep`: Transaction panics (DoS for staking).
*   **Relation to CALC-004/CALC-003**: The fundamental issue is the use of `u64` for stake and balance fields (CALC-004). The direct fund loss risk from `active_stake + inactive_stake` overflowing during `remove_stake` is CALC-003. Here, an overflow in `add_deep` would cause a DoS. Overflows in `inactive_stake` or `total_stake` primarily lead to incorrect accounting unless they trigger a DoS or the CALC-003 scenario upon unstaking.

---

### **CALC-010: `governance::adjust_vote` u64 Overflow/Underflow for `proposal.votes`**

*   **Module & Function**: `deepbook::state::governance::adjust_vote`
*   **Affected Variables/State**: `Proposal::votes` (u64), `Governance::next_trade_params`.
*   **Potential Issue & Exploit Path Confirmation**:
    *   **Underflow**: `proposal.votes = proposal.votes - votes;` can panic if `votes` (current voting power of the user changing their vote, derived from `stake_to_voting_power(stake_amount)`) is greater than `proposal.votes` (current total votes for the proposal being un-voted).
        *   **Scenario**: User U with stake S1 (contributing `vp_old_contrib = stake_to_voting_power(S1)`) votes for Proposal P1. `P1.votes` becomes `vp_old_contrib + other_votes`. User U then significantly increases their stake to S2 (now `stake_amount` for `adjust_vote` call, new contribution `vp_new_contrib = stake_to_voting_power(S2)`). User U then decides to change their vote from P1 to P2. When removing the vote from P1, `adjust_vote` is called with `vote_type = UnVoted` and `stake_amount = S2`. The `votes` to subtract will be `vp_new_contrib`. The operation `P1.votes = (vp_old_contrib + other_votes) - vp_new_contrib` will panic if `vp_new_contrib > vp_old_contrib + other_votes`. This is likely if `other_votes` is small and `vp_new_contrib` (from S2) is substantially larger than `vp_old_contrib` (from S1).
        *   **Consequence**: A user who increases their stake *after* voting for a proposal might be unable to change their vote (remove it from the original proposal) if their new voting power is much larger than their old one and the proposal has few other votes.
    *   **Overflow**: `proposal.votes = proposal.votes + votes;` can panic if the sum exceeds `u64::MAX`. This can happen if a proposal receives a very large number of votes or votes from stakers with very high individual voting power.
*   **Ultimate Impact**:
    *   **DoS**: Panics can prevent users from changing votes or new votes from being registered if they trigger overflow/underflow. This could manipulate governance outcomes by preventing further voting changes or disincentivizing participation.
    *   Incorrect `next_trade_params`: If a winning proposal loses quorum due to an underflow panic preventing vote removal, or fails to achieve quorum due to an overflow panic preventing vote addition, the `next_trade_params` might not reflect the true desired outcome.
*   **Mitigations**:
    *   Move's default panic on overflow/underflow prevents silent corruption.
    *   **Missing/Insufficient**: `Proposal::votes` should ideally be `u128` to accommodate sums of many `u64` voting powers. For subtraction, the logic should ensure that the `votes` value subtracted corresponds to the actual voting power the user contributed to *that specific proposal at the time of their original vote*, rather than their current total voting power. This would require storing more granular vote information (e.g., a receipt per vote or tracking individual vote contributions). Alternatively, `saturating_sub` could prevent panic but might lead to `proposal.votes` becoming zero prematurely if a user with large current voting power unvotes a proposal they previously voted for with smaller power.

---

### **CALC-011: `governance::adjust_voting_power` u64 Overflow/Underflow for `self.voting_power`**

*   **Module & Function**: `deepbook::state::governance::adjust_voting_power`
*   **Affected Variables/State**: `Governance::voting_power` (u64), and consequently `Governance::quorum`.
*   **Potential Issue & Exploit Path Confirmation**:
    *   The calculation is `self.voting_power = self.voting_power + stake_to_voting_power(stake_after) - stake_to_voting_power(stake_before);`. Let `vp_current = self.voting_power`, `vp_new_contrib = stake_to_voting_power(stake_after)`, `vp_old_contrib = stake_to_voting_power(stake_before)`. The effective operation is `vp_current + (vp_new_contrib - vp_old_contrib)`.
    *   **Overflow**: Can occur if `vp_new_contrib > vp_old_contrib` and `vp_current + (vp_new_contrib - vp_old_contrib)` exceeds `u64::MAX`. This is possible if total voting power is already high and a user significantly increases their stake, leading to a large positive net change in voting power.
    *   **Underflow**: Can occur if `vp_old_contrib > vp_new_contrib` and `vp_old_contrib - vp_new_contrib > vp_current`. This means the amount of voting power being removed is greater than the current total voting power. This is highly likely if a user with a very large stake (whose `vp_old_contrib` forms a significant portion of `vp_current`) unstakes most or all of their funds (making `vp_new_contrib` small or zero). The subtraction `vp_current - (vp_old_contrib - vp_new_contrib)` would then underflow.
*   **Ultimate Impact**:
    *   **DoS**: Panics during `adjust_voting_power` (called by `Account::add_stake` and `Account::remove_stake` via `state.move`) would prevent users from staking or unstaking if their operation triggers the overflow/underflow. This is a significant issue as it can lock user funds or prevent participation.
    *   **Incorrect Quorum**: If `voting_power` became corrupted due to wrap-around (though panics prevent this by default), the `quorum` calculation (`voting_power / 2`) for subsequent epochs would be incorrect, undermining governance.
*   **Mitigations**:
    *   Move's default panic on overflow/underflow.
    *   **Missing/Insufficient**: `Governance::voting_power` should ideally be `u128` to prevent overflow/underflow from the sum/difference of many `u64` voting power contributions. The arithmetic should be performed using `u128` for intermediate calculations before any necessary casting. For instance, `(vp_current.as_u128() + vp_new_contrib.as_u128() - vp_old_contrib.as_u128()).as_u64()`, with appropriate checks for final casting if `voting_power` remains `u64`. Using `u128` for `Governance::voting_power` itself is the most robust solution.

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
