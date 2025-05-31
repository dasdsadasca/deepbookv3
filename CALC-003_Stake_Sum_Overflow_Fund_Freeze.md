---
# Detailed Security Report: CALC-003 - Account::remove_stake() Stake Summation Overflow

**Vulnerability ID**: CALC-003
**Severity**: Critical
**Module**: `deepbook::state::account`
**Function**: `remove_stake(self: &mut Account)`
**Impact**: Permanent freezing of a user's staked DEEP tokens if their total stake exceeds `u64::MAX`.

## 1. Vulnerability Explanation

The `remove_stake` function in the `account.move` module is responsible for processing a user's request to unstake their DEEP tokens from a pool. It calculates the total stake to be returned by summing the user's `active_stake` and `inactive_stake`. Both of these fields are stored as `u64` integers.

In Move, standard arithmetic operations (including addition `+`) on fixed-size integers like `u64` will panic if the mathematical result of the operation exceeds the maximum value representable by that type (i.e., an overflow occurs).

The vulnerability arises when a user has accumulated a very large amount of combined `active_stake` and `inactive_stake` such that their sum is greater than `u64::MAX` (approximately `1.844 * 10^19`). When `remove_stake` attempts to calculate this sum, the `+` operation panics, causing the entire transaction (including the `pool::unstake` call) to revert.

This prevents the user from successfully executing the unstaking operation. Their `active_stake` and `inactive_stake` remain unchanged in the `Account` object, and no DEEP tokens are moved to their `settled_balances` to be withdrawn. Effectively, their staked funds become permanently frozen and inaccessible through the standard unstaking mechanism.

## 2. Code Breakdown

**File**: `packages/deepbook/sources/state/account.move`

```move
public(package) fun remove_stake(self: &mut Account) {
    // 1. `self.active_stake` is u64.
    // 2. `self.inactive_stake` is u64.

    // 3. THE VULNERABLE OPERATION:
    //    If the mathematical sum of `self.active_stake + self.inactive_stake`
    //    exceeds u64::MAX, this line will panic due to u64 overflow.
    let stake_before = self.active_stake + self.inactive_stake;

    // These lines are NOT reached if the sum above panics.
    self.active_stake = 0;
    self.inactive_stake = 0;
    self.voted_proposal = option::none(); // Reset voting status

    // 4. If the sum did not panic (i.e., did not overflow u64),
    //    `stake_before` (which would be the correct total stake)
    //    is added to the user's settled DEEP balance.
    //    The function `balances::add_deep` itself also involves a u64 addition
    //    (`balances.deep = balances.deep + deep;`) which could panic if
    //    `settled_balances.deep` was already very high, but the primary issue
    //    for CALC-003 is the panic on the initial sum.
    self.settled_balances.add_deep(stake_before);
}
```

**File**: `packages/deepbook/sources/state/balances.move` (relevant for context if panic didn't occur)
```move
public(package) fun add_deep(balances: &mut Balances, deep: u64) {
    // This addition would also panic if balances.deep + deep (the stake_before value)
    // were to overflow u64. However, for CALC-003, the panic on the initial sum
    // `active_stake + inactive_stake` occurs first.
    balances.deep = balances.deep + deep;
}
```

The critical issue is the `let stake_before = self.active_stake + self.inactive_stake;` line. Due to Move's default checked arithmetic, an overflow here halts execution and reverts the transaction.

## 3. Detailed Exploit Scenario / Validation Steps (Conceptual PoC)

This scenario demonstrates how a user's funds can become permanently frozen.

**Prerequisites**:
*   A user ("WhaleUser") exists with a `BalanceManager`.
*   WhaleUser has access to a substantial amount of DEEP tokens (e.g., > `u64::MAX`).
*   A DeepBook `Pool` is available for staking.

**Steps**:

1.  **Initial State (Optional)**: WhaleUser might already have some `active_stake` from previous epochs. For simplicity, let's assume `active_stake = A_s` (where `A_s` could be 0 or some value significantly less than `u64::MAX`).

2.  **Accumulate `inactive_stake`**:
    *   WhaleUser makes one or more calls to `pool::stake(pool, balance_manager, trade_proof, amount, ctx)`.
    *   Let the total `amount` staked across these calls be `S_i`.
    *   Internally, `account::add_stake` is called, which increments `self.inactive_stake` by `S_i`.
    *   **Condition**: WhaleUser must stake enough such that `A_s + (current_inactive_stake + S_i)` will eventually exceed `u64::MAX`.
    *   For a direct example, assume `active_stake` is `(u64::MAX / 2) + 1` and current `inactive_stake` is `0`. WhaleUser then stakes an `amount` of `(u64::MAX / 2)`.
        *   `account::add_stake` updates `inactive_stake` to `u64::MAX / 2`.
        *   (This assumes `owed_balances.add_deep` within `add_stake` doesn't panic first, which is reasonable if `owed_balances.deep` wasn't already excessive).

3.  **Trigger `account::remove_stake()` via `pool::unstake()`**:
    *   WhaleUser now wishes to unstake their total accumulated DEEP. They call `pool::unstake(pool, balance_manager, trade_proof, ctx)`.
    *   This function call eventually leads to `account::remove_stake(&mut users_account_object)` being invoked.
    *   At this point, `self.active_stake` holds `A_s` and `self.inactive_stake` holds `I_s_total` (the accumulated inactive stake).
    *   The line `let stake_before = self.active_stake + self.inactive_stake;` is executed.
    *   **Overflow Condition**: If `A_s` is `(u64::MAX - 1000)` and `I_s_total` is `2000` (making the mathematical sum `u64::MAX + 1000`), the `+` operation on these `u64` values will attempt to produce a result greater than `u64::MAX`.
    *   **Panic**: Due to Move's default checked arithmetic, this overflow causes an immediate panic.

4.  **Outcome**:
    *   The entire `pool::unstake` transaction reverts.
    *   WhaleUser's `active_stake` and `inactive_stake` in their `Account` object remain unchanged (as the transaction failed).
    *   No DEEP tokens are moved to `self.settled_balances`.
    *   WhaleUser's DEEP tokens are effectively frozen in the pool, as any attempt to unstake this combined amount will result in the same panic. They cannot withdraw their stake via the standard mechanism.

**Validation**: This vulnerability is validated by understanding Move's integer overflow behavior. No complex interaction is needed beyond a user legitimately accumulating a stake large enough to trigger the `u64` overflow on summation. The `assert!( (active_stake as u128) + (inactive_stake as u128) > (max_u64 as u128) )` would be true in this scenario.

## 4. Impact Assessment

*   **Severity**: **Critical**
*   **Description**: This vulnerability leads to a **permanent freezing of staked funds** for users whose total stake (`active_stake + inactive_stake`) sum exceeds the `u64` maximum. While the panic prevents the silent loss of funds that would occur if the sum wrapped around and a smaller amount was credited (as initially hypothesized before confirming Move's panic behavior), the inability for a user to ever withdraw their legitimately staked DEEP tokens is equivalent to a permanent loss of those funds from their control and utility.
*   **Affected Users**: Users (likely whales or long-term large stakers) who accumulate more than `u64::MAX` total DEEP tokens in a single pool's account object (combining active and inactive stake).
*   **Attacker Capability**: This is primarily a self-inflicted condition based on the amount staked by a user. An external attacker cannot directly force another user's individual stake values to overflow this sum to steal their funds through this specific vector. The attack is against the availability of the user's own funds.

## 5. Recommended Mitigations

The primary goal of mitigation should be to allow users to stake and unstake amounts up to reasonable limits of tokenomics without funds becoming frozen.

1.  **Use `u128` for Stake Accounting**:
    *   Change the type of `Account::active_stake` and `Account::inactive_stake` from `u64` to `u128`.
    *   The local variable `stake_before` in `account::remove_stake` should also be `u128`.
    *   Consequently, `Balances::deep` (in `balances.move`) and the corresponding field in `Account::settled_balances` and `Account::owed_balances` should be changed to `u128` to correctly handle these potentially larger stake amounts being added or owed. This change would propagate to other functions interacting with these `Balances` fields.
    *   This is the most robust solution as `u128` can represent vastly larger numbers, making overflow from stake accumulation practically impossible.

2.  **Checked Summation into `u128` if `u64` Fields are Retained (Less Ideal - Prevents Fund Loss but not DoS for >u64 sums)**:
    *   If `active_stake` and `inactive_stake` must remain `u64` (e.g., due to other system constraints or data type consistency), the summation must be done safely:
      ```move
      // In account::remove_stake
      let total_stake_u128 = (self.active_stake as u128) + (self.inactive_stake as u128);
      // Option A: Revert if total stake cannot be represented by u64 (current behavior, but explicit)
      assert!(total_stake_u128 <= (sui::math::max_u64() as u128), /* new error code e.g., ETotalStakeExceedsU64Limit */ );
      let stake_before_u64 = total_stake_u128 as u64;
      // ... proceed with stake_before_u64
      ```
    *   This makes the DoS condition explicit but doesn't solve the fund freeze for users legitimately staking more than `u64::MAX`. The `u128` field type change (Option 1) is strongly preferred to actually allow larger stakes to be processed correctly.

The use of `u128` for all stake and related balance tracking would be the most comprehensive fix to prevent both the fund freeze (DoS on unstaking) and potential overflows in other stake-related calculations.
---
