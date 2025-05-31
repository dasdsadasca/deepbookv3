---
# Detailed Security Report: CALC-004 - Balances u64 Overflow Leading to DoS

**Vulnerability ID**: CALC-004
**Severity**: High/Medium (depending on the specific operation blocked)
**Module**: `deepbook::balances`, `deepbook::state::account`
**Function**:
*   `deepbook::balances::add_balances(balances: &mut Balances, other: Balances)`
*   `deepbook::balances::add_base(balances: &mut Balances, amount: u64)`
*   `deepbook::balances::add_quote(balances: &mut Balances, amount: u64)`
*   `deepbook::balances::add_deep(balances: &mut Balances, amount: u64)`
*   These are called by various `deepbook::state::account` functions such as `process_maker_fill`, `add_settled_balances`, `add_owed_balances`, `claim_rebates`, `add_stake`.
**Impact Summary**: Panics due to `u64` overflow when updating `Balances` struct fields (used within `Account` for `settled_balances`, `owed_balances`, `unclaimed_rebates`) can cause Denial of Service for critical user operations like settling trades, claiming rebates, or staking/unstaking balance adjustments.

## 1. Vulnerability Explanation

The `Balances` struct in `balances.move` is used to track amounts of base, quote, and DEEP tokens. Its fields (`base`, `quote`, `deep`) are all `u64` integers. The `Account` struct in `account.move` uses instances of this `Balances` struct to manage various user-specific balances, such as `settled_balances` (funds ready for withdrawal), `owed_balances` (funds due to the user, e.g., from staking), and `unclaimed_rebates`.

When functions like `balances::add_balances` or its helpers (`add_base`, `add_quote`, `add_deep`) are called to increment these `u64` fields, the standard `+` operation is used. In Move, these arithmetic operations will panic if the mathematical result exceeds the maximum value representable by `u64` (approximately `1.844 * 10^19`).

This vulnerability occurs when a user's balance component (e.g., `settled_balances.base`) is already very close to `u64::MAX`, and a subsequent operation attempts to add even a small amount, causing the addition to panic. This reverts the entire transaction, leading to a Denial of Service (DoS) for the user attempting that operation.

## 2. Code Breakdown

**File**: `packages/deepbook/sources/state/balances.move`
```move
struct Balances has copy, drop, store {
    base: u64,
    quote: u64,
    deep: u64,
}

public(package) fun add_balances(balances: &mut Balances, other: Balances) {
    // Each of these additions can panic if the sum exceeds u64::MAX
    balances.base = balances.base + other.base;
    balances.quote = balances.quote + other.quote;
    balances.deep = balances.deep + other.deep;
}

public(package) fun add_deep(balances: &mut Balances, deep_amount: u64) {
    // This addition can panic if balances.deep + deep_amount exceeds u64::MAX
    balances.deep = balances.deep + deep_amount;
}
// Similar add_base and add_quote functions exist
```

**File**: `packages/deepbook/sources/state/account.move` (examples of calls)
```move
struct Account has key, store {
    // ...
    settled_balances: Balances,
    owed_balances: Balances,
    unclaimed_rebates: Balances,
    // ...
}

public(package) fun process_maker_fill(
    self: &mut Account,
    // ...
    settled_balances_update: Balances,
    // ...
) {
    // ...
    // Vulnerable call if self.settled_balances fields are near u64::MAX
    self.settled_balances.add_balances(settled_balances_update);
    // ...
}

public(package) fun claim_rebates(self: &mut Account, ctx: &TxContext) {
    // ...
    let rebates_to_claim = self.unclaimed_rebates;
    self.unclaimed_rebates = Balances::zero();
    // Vulnerable call if self.settled_balances fields are near u64::MAX
    // and rebate amounts are being added.
    self.settled_balances.add_balances(rebates_to_claim);
    // ...
}

public(package) fun add_stake(self: &mut Account, stake_amount: u64, /* ... */) {
    // ...
    // Vulnerable call if self.owed_balances.deep is near u64::MAX
    self.owed_balances.add_deep(stake_amount);
    // ...
}
```

## 3. Exploit Scenario / Validation Steps (Conceptual PoC for DoS)

**General Scenario**:
1.  A user's `Account` has a specific `Balances` field (e.g., `settled_balances.deep` or `owed_balances.base`) that, through numerous legitimate transactions (many small fills, large deposits that get settled, or large rebate accumulations), reaches a value very close to `u64::MAX`.
2.  A subsequent, unrelated or related, operation attempts to add a small additional amount to this specific balance component.
3.  The `balances::add_<asset>()` or `balances::add_balances()` function is called.
4.  The `balance.field = balance.field + amount_to_add` operation panics due to `u64` overflow.
5.  The entire transaction reverts.

**Example 1: DoS in `process_maker_fill`**
*   **Prerequisite**: A maker user ("MakerWhale") has `settled_balances.base = u64::MAX - 50` in their `Account` for a specific pool.
*   **Trigger**: A taker order matches one of MakerWhale's resting orders. The fill settlement logic calculates that MakerWhale should receive `100` units of the base asset.
*   `deepbook::state::process_create` (or similar fill processing function) calls `account::process_maker_fill` for MakerWhale's account.
*   Inside `process_maker_fill`, `self.settled_balances.add_balances(fill_settlement_for_maker)` is called. `fill_settlement_for_maker` contains `base = 100`.
*   The operation `self.settled_balances.base = (u64::MAX - 50) + 100` is attempted within `add_balances`.
*   **Panic**: This sum exceeds `u64::MAX`, causing a panic.
*   **Result**: The transaction processing the trade (and fill) reverts. MakerWhale's fill is not processed, their `settled_balances` are not updated, and they do not receive the `100` base tokens from this trade into their `settled_balances`. Any other effects of that transaction (e.g., for the taker) are also reverted. MakerWhale is effectively blocked from receiving further base asset settlements if their `settled_balances.base` is this high.

**Example 2: DoS in `claim_rebates`**
*   **Prerequisite**: User ("RebateWhale") has `settled_balances.deep = u64::MAX - 50`. They also have accumulated `unclaimed_rebates.deep = 100`.
*   **Trigger**: RebateWhale calls `pool::claim_rebates(...)`.
*   This calls `account::claim_rebates(...)`.
*   The line `self.settled_balances.add_balances(self.unclaimed_rebates)` is executed.
*   When adding the `deep` components, `self.settled_balances.deep = (u64::MAX - 50) + 100` is attempted.
*   **Panic**: This sum exceeds `u64::MAX`, causing a panic.
*   **Result**: RebateWhale cannot claim their rebates. The transaction reverts, and their `unclaimed_rebates` and `settled_balances` remain as they were.

**Attacker Capability**:
*   This is primarily a self-inflicted DoS for users who accumulate extremely large balances in one component of their `Account`'s `Balances` fields.
*   An attacker could potentially grief a user whose balances are known to be very close to `u64::MAX`. For instance, if VictimUser has `settled_balances.base = u64::MAX - 50`, Attacker could place a small taker order that matches VictimUser's maker order, intending to credit VictimUser with just enough (e.g., 51+ units) to cause their `process_maker_fill` to panic. This would prevent VictimUser from participating in that specific trade settlement.

## 4. Impact Assessment

*   **Severity**: **High** (if blocking critical operations like trade settlement or unstaking/claiming of significant value for a user) to **Medium** (if blocking less critical functions or only affecting users at extreme, unlikely balance levels).
*   **Integrity/Availability**: The panic on overflow is a crucial safety feature of Move that prevents silent state corruption (e.g., balances wrapping around to a small value). The state remains consistent by reverting the transaction.
*   **Denial of Service**: The primary impact is a Denial of Service for the specific user and the specific operation that triggers the overflow. This can prevent users from:
    *   Receiving proceeds from their trades into `settled_balances`.
    *   Claiming their accumulated `unclaimed_rebates`.
    *   Staking additional tokens if their `owed_balances.deep` (where new stake is temporarily held) is near maximum.
    *   Unstaking or processing order cancellations/modifications if these operations involve adding to a `settled_balances` field that is near maximum.
*   **Fund Loss**: This vulnerability (CALC-004 itself) does not directly cause fund loss because the transaction reverts, and no state is wrongly committed. However, it can make parts of the system unusable for users with very large balances, effectively "freezing" their ability to perform certain actions.
*   **Relation to CALC-003**: The critical fund freeze issue CALC-003 (`Account::remove_stake` overflow on `active_stake + inactive_stake`) occurs due to an overflow in a sum *before* `balances::add_deep` is called. CALC-004 can also affect `remove_stake` if `settled_balances.deep` is already near `u64::MAX` when the (correctly calculated) `stake_before` is added to it.

## 5. Recommended Mitigations

The most robust solution is to ensure that balance accumulations can handle the expected scale of token amounts without panicking.

1.  **Primary Mitigation: Use `u128` for Balance Fields**:
    *   Change the fields `base`, `quote`, and `deep` within the `deepbook::balances::Balances` struct from `u64` to `u128`.
    *   This change will propagate to all instances of `Balances` used in `Account` (i.e., `settled_balances`, `owed_balances`, `unclaimed_rebates`).
    *   Functions like `add_balances`, `add_base`, `add_quote`, `add_deep` will then operate on `u128` fields.
    *   This will make overflows from legitimate token accumulations for individual users or even sum totals practically impossible, thus preventing the DoS scenarios described.

2.  **Propagate `u128` Changes**:
    *   Ensure that all functions passing amounts to these `Balances` functions, or reading from them, are also using `u128` where appropriate to maintain consistency and prevent truncation or new overflow points. For example, amounts passed into `add_balances` or `add_deep` should be `u128` if they represent sums that could exceed `u64`.

Using `u128` for balance fields is the standard way to handle token ledgers in Sui and other Move-based systems to avoid these types of overflow-related DoS issues.
---
