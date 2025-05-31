---
# Detailed Security Report: CALC-010 & CALC-011 - Governance Logic Overflows/Underflows

The integrity of the governance mechanism is crucial for the long-term stability and adaptability of the DeepBookV3 protocol. This report details two vulnerabilities, CALC-010 and CALC-011, within the `deepbook::state::governance` module that arise from `u64` integer overflows/underflows. These can lead to Denial of Service (DoS) for governance operations (staking, unstaking, voting) and potentially impact the accuracy of governance outcomes, such as quorum calculations and proposal voting results.

---

## CALC-011: `governance::adjust_voting_power` - Total Voting Power Overflow/Underflow

**Vulnerability ID**: CALC-011
**Severity**: High
**Module**: `deepbook::state::governance`
**Function**: `adjust_voting_power(self: &mut Governance, stake_before: u64, stake_after: u64)`
**Impact Summary**: Panics due to `u64` overflow/underflow when updating `Governance::voting_power` can cause DoS for staking/unstaking operations (preventing users from changing their stake) and lead to incorrect quorum calculations if `voting_power` updates are missed.

### 1. Vulnerability Explanation

The `Governance` struct stores `voting_power`, a `u64` field representing the total sum of voting power from all staked DEEP tokens in the pool. The `adjust_voting_power` function updates this total whenever a user's stake changes (on staking or unstaking).

The function `stake_to_voting_power(stake_amount: u64): u64` converts a user's `u64` stake amount into a `u64` voting power. While `stake_to_voting_power` itself is presumed safe from internal overflow for valid stake amounts (as it typically involves scaling and possibly square root, designed to output a `u64`), the `adjust_voting_power` function performs additions and subtractions with the pool's total `voting_power` that can overflow or underflow.

The core calculation is `self.voting_power = self.voting_power + stake_to_voting_power(stake_after) - stake_to_voting_power(stake_before);`. Let `vp_current = self.voting_power`, `vp_new_contrib = stake_to_voting_power(stake_after)`, and `vp_old_contrib = stake_to_voting_power(stake_before)`. The operation can be seen as `vp_current + (vp_new_contrib - vp_old_contrib)`.
*   **Overflow**: If `vp_new_contrib > vp_old_contrib` (user increased stake or new staker), the net addition to `vp_current` can exceed `u64::MAX`.
*   **Underflow**: If `vp_old_contrib > vp_new_contrib` (user decreased stake or unstaked all), the net subtraction from `vp_current` can be larger than `vp_current` itself, leading to an attempt to make `voting_power` negative.

In Move, standard arithmetic operations panic on overflow/underflow, reverting the transaction.

### 2. Code Breakdown

**File**: `packages/deepbook/sources/state/governance.move`
```move
struct Governance has store {
    voting_power: u64, // Total voting power in the pool
    // ... other fields like proposals, quorum, next_trade_params
}

public(package) fun stake_to_voting_power(stake: u64): u64 {
    // Example: could be linear (stake / C) or sqrt(stake * C)
    // For this analysis, we assume it returns a valid u64.
    // The vulnerability is in how its result is used in adjust_voting_power.
    // if stake < MIN_STAKE_FOR_VOTING_POWER { return 0 }
    // else { (stake as u128 * FLOAT_SCALING / TOKEN_DECIMAL_SCALING_FACTOR) as u64 } (Illustrative)
    // Actual implementation: math::sqrt_u128(math::mul(stake, constants::float_scaling(), constants::token_e8()))
    math::sqrt_u128(math::mul(stake, constants::float_scaling(), constants::token_e8())) // returns u64
}

public(friend) fun adjust_voting_power(
    self: &mut Governance,
    stake_before: u64, // User's stake amount before the current operation
    stake_after: u64   // User's stake amount after the current operation
) {
    let voting_power_before = stake_to_voting_power(stake_before);
    let voting_power_after = stake_to_voting_power(stake_after);

    if (voting_power_after > voting_power_before) {
        // User's voting power increased (e.g., new stake, added stake)
        // OVERFLOW VULNERABILITY HERE:
        self.voting_power = self.voting_power + (voting_power_after - voting_power_before);
    } else if (voting_power_before > voting_power_after) {
        // User's voting power decreased (e.g., removed stake, partial unstake)
        // UNDERFLOW VULNERABILITY HERE:
        self.voting_power = self.voting_power - (voting_power_before - voting_power_after);
    };
    // If voting_power_after == voting_power_before, no change.
}
```

### 3. Overflow/Underflow Scenarios

**Overflow Scenario**:
1.  **Prerequisite**: The pool's total `Governance::voting_power` (`vp_current`) is already very high, e.g., `u64::MAX - 10^12`. This could be due to many users staking large amounts.
2.  **Trigger**: A user performs an action that significantly increases their stake. For example, a new whale stakes a large amount (`stake_after` is large, `stake_before` was 0), or an existing large staker adds more stake.
    *   `vp_old_contrib = stake_to_voting_power(stake_before)` (e.g., 0 if new staker).
    *   `vp_new_contrib = stake_to_voting_power(stake_after)` (e.g., `2 * 10^12`).
3.  **Calculation**: `adjust_voting_power` attempts `self.voting_power = vp_current + (vp_new_contrib - vp_old_contrib)`.
    *   This becomes `(u64::MAX - 10^12) + (2 * 10^12 - 0)`. The mathematical sum `u64::MAX + 10^12` exceeds `u64::MAX`.
4.  **Panic**: The addition panics due to overflow. The transaction (e.g., `pool::stake`) reverts.

**Underflow Scenario**:
1.  **Prerequisite**: `Governance::voting_power` (`vp_current`) is positive, but a single large staker ("WhaleStaker") accounts for a majority of it. E.g., `vp_current = 10^13`. WhaleStaker's `stake_before` contributes `vp_old_contrib = 8 * 10^{12}` to this total.
2.  **Trigger**: WhaleStaker calls `pool::unstake` to remove all their stake. So, `stake_after = 0`, making `vp_new_contrib = stake_to_voting_power(0) = 0`.
3.  **Calculation**: `adjust_voting_power` attempts `self.voting_power = vp_current - (vp_old_contrib - vp_new_contrib)`.
    *   This becomes `10^13 - (8 * 10^{12} - 0)`. The amount to subtract is `8 * 10^{12}`.
    *   The new `self.voting_power` would be `2 * 10^{12}`, which is fine.
    *   **More Extreme Case**: Suppose `vp_current = 10^{13}` and WhaleStaker's `vp_old_contrib` was `10^{13}` (they were the only staker). They unstake all. `vp_new_contrib = 0`.
        *   `self.voting_power = 10^{13} - (10^{13} - 0) = 0`. This is fine.
    *   **Actual Underflow Path**: The code is `self.voting_power = self.voting_power - (voting_power_before - voting_power_after);`. If `(voting_power_before - voting_power_after)` (the delta) is greater than `self.voting_power` at that moment, it underflows.
        *   Example: `self.voting_power = 500`. User A unstakes, `vp_before = 1000`, `vp_after = 0`. Delta is `1000`. `500 - 1000` panics. This implies `self.voting_power` was not correctly reflecting the sum of all individuals' voting powers if one individual's removal delta can exceed the total. This scenario points to potential state inconsistency or that `self.voting_power` could become small due to other large unstakes before this specific user's unstake is processed.

### 4. Impact Assessment (CALC-011)

*   **Severity**: **High**
*   **Denial of Service (DoS)**:
    *   If an overflow is triggered when a user tries to stake, their `pool::stake` transaction will revert. They will be unable to stake.
    *   If an underflow is triggered when a user tries to unstake, their `pool::unstake` transaction will revert. Their funds remain staked and inaccessible via the normal unstake mechanism, effectively a **temporary fund freeze**.
*   **Incorrect Quorum Calculation**:
    *   If `adjust_voting_power` panics, `Governance::voting_power` is not updated.
    *   The `quorum` for proposal voting is calculated as `self.voting_power / 2` (potentially using `math::mul` with `constants::half()`).
    *   If `voting_power` becomes stale due to repeated panics in `adjust_voting_power`, the quorum will not reflect the true current total voting power. This could make it too easy for proposals to pass (if actual total VP has decreased but `quorum` is based on an old higher VP) or too hard (if actual total VP has increased but `quorum` is based on an old lower VP). This undermines the integrity of the governance process.

### 5. Recommended Mitigations (CALC-011)

1.  **Primary: Use `u128` for `Governance::voting_power`**:
    *   Change the type of `Governance::voting_power` from `u64` to `u128`.
    *   The return type of `stake_to_voting_power` should also be changed to `u128`.
    *   All arithmetic operations within `adjust_voting_power` (addition and subtraction) should then be performed using `u128` operands. This makes overflows/underflows from summing individual voting power contributions practically impossible.
2.  **Quorum Calculation Adjustment**:
    *   If `voting_power` becomes `u128`, the quorum calculation `self.voting_power / 2` will also be `u128`. Ensure that `math::mul` or other functions used for calculating quorum (e.g., with `constants::half()`) are compatible with `u128` inputs for `voting_power` and produce the correct scaled `u64` or `u128` result as needed by `Proposal::min_stake_required_to_propose` or other quorum checks.

---

## CALC-010: `governance::adjust_vote` - Proposal Vote Count Overflow/Underflow

**Vulnerability ID**: CALC-010
**Severity**: Medium-High
**Module**: `deepbook::state::governance`
**Function**: `adjust_vote(self: &mut Governance, account: &Account, from_proposal_id: Option<u64>, to_proposal_id: Option<u64>, stake_amount: u64, current_timestamp_ms: u64)`
**Impact Summary**: Panics from `u64` overflow/underflow in `Proposal::votes` when users cast or change votes can cause DoS for voting operations. This can prevent users from participating in governance or changing their vote, potentially skewing governance outcomes.

### 1. Vulnerability Explanation

The `Proposal` struct stores `votes` (for or against, depending on the proposal type, but generally a single `u64` counter per proposal option) as a `u64` field. When a user votes, their `stake_amount` is converted to voting power (`votes_to_add_or_remove = stake_to_voting_power(stake_amount)`), which is also a `u64`.

The `adjust_vote` function updates a proposal's `votes` by either adding (`proposal.votes = proposal.votes + votes_to_add_or_remove`) or subtracting (`proposal.votes = proposal.votes - votes_to_add_or_remove`).
*   **Overflow**: If many users vote for a proposal, or a few users with very high voting power vote, the `proposal.votes + new_votes` sum can exceed `u64::MAX`.
*   **Underflow**: When a user changes their vote, their voting power is subtracted from the old proposal and added to the new one. If the `votes_to_add_or_remove` (calculated from the user's *current* `stake_amount`) is greater than the `proposal.votes` it's being subtracted from, an underflow will occur.

Both scenarios cause a panic, reverting the voting transaction.

### 2. Code Breakdown

**File**: `packages/deepbook/sources/state/governance.move`
```move
struct Proposal has store {
    // ...
    votes: u64, // Total votes for this proposal/option
    // ...
}

public(friend) fun adjust_vote(
    self: &mut Governance,
    account: &Account, // User's account, contains their stake
    from_proposal_id: Option<u64>, // Proposal to unvote (if changing vote)
    to_proposal_id: Option<u64>,   // Proposal to vote for
    stake_amount: u64, // User's current total stake used for voting power
    current_timestamp_ms: u64
) {
    let votes_to_add_or_remove = stake_to_voting_power(stake_amount);

    if (option::is_some(&from_proposal_id)) {
        let proposal_id = option::destroy_some(from_proposal_id);
        let proposal = vector::borrow_mut(&mut self.proposals, proposal_id);
        // Ensure proposal is active, etc. (checks omitted for brevity)

        // UNDERFLOW VULNERABILITY HERE:
        // If votes_to_add_or_remove > proposal.votes
        proposal.votes = proposal.votes - votes_to_add_or_remove;
    };

    if (option::is_some(&to_proposal_id)) {
        let proposal_id = option::destroy_some(to_proposal_id);
        let proposal = vector::borrow_mut(&mut self.proposals, proposal_id);
        // Ensure proposal is active, etc.

        // OVERFLOW VULNERABILITY HERE:
        // If proposal.votes + votes_to_add_or_remove > u64::MAX
        proposal.votes = proposal.votes + votes_to_add_or_remove;
    };
}
```

### 3. Overflow/Underflow Scenarios

**Underflow Scenario (Changing Vote with Increased Stake)**:
1.  **Initial Vote**: User U has an initial stake `S1`. They call `pool::vote` for Proposal P1. `adjust_vote` is called, and `votes_contributed_P1 = stake_to_voting_power(S1)` is added to `P1.votes`. So, `P1.votes = VP1_initial + OtherVotes`. User's `account.voted_proposal` is set to `Some(P1_id)`.
2.  **Increase Stake**: User U later significantly increases their total stake in the pool to `S2` (where `S2 > S1`). Their voting power is now `VP2 = stake_to_voting_power(S2)`.
3.  **Change Vote**: User U decides to change their vote from Proposal P1 to Proposal P2. They call `pool::vote` again.
    *   `adjust_vote` is called with `from_proposal_id = Some(P1_id)`, `to_proposal_id = Some(P2_id)`, and importantly, `stake_amount = S2` (their current total stake).
    *   The `votes_to_add_or_remove` is calculated as `VP2 = stake_to_voting_power(S2)`.
4.  **Subtraction from P1**: The code attempts `P1.votes = P1.votes - VP2`.
    *   This becomes `P1.votes = (VP1_initial + OtherVotes) - VP2`.
    *   **Panic Condition**: If `VP2` (derived from the user's *current, larger* stake `S2`) is greater than `VP1_initial + OtherVotes` (the total votes P1 had, including the user's *original, smaller* contribution), this subtraction will panic due to underflow. This is likely if `OtherVotes` is small.
5.  **Result**: The transaction reverts. User U is unable to change their vote from P1 to P2. Their vote remains on P1.

**Overflow Scenario (Popular Proposal)**:
1.  **Prerequisite**: Proposal P1 is very popular, or a few "whale" stakers with massive voting power have already voted for it. `P1.votes` is close to `u64::MAX` (e.g., `u64::MAX - 1000`).
2.  **Trigger**: Another user (User X) with voting power `VP_X` (e.g., `2000`) attempts to vote for Proposal P1.
3.  **Calculation**: `adjust_vote` attempts `P1.votes = P1.votes + VP_X`.
    *   This becomes `(u64::MAX - 1000) + 2000`. The mathematical sum exceeds `u64::MAX`.
4.  **Panic**: The addition panics due to overflow. The transaction reverts.
5.  **Result**: User X is unable to cast their vote for Proposal P1.

### 4. Impact Assessment (CALC-010)

*   **Severity**: **Medium-High**
*   **Denial of Service (DoS)**:
    *   The underflow scenario can prevent users who have increased their stake from changing their previous votes. This is problematic as it doesn't correctly reflect their current preferences with their current voting power.
    *   The overflow scenario can prevent users from voting for an already popular proposal, effectively capping the number of expressible votes or total voting power for a proposal.
*   **Skewed Governance Outcomes**:
    *   If users are blocked from changing votes or casting new votes, the final vote counts for proposals may not accurately reflect the collective will of the stakers.
    *   A proposal might incorrectly appear to pass or fail if voting is impeded. For example, if a user wants to remove their vote from a proposal that is near the quorum threshold, but cannot due to the underflow issue, that proposal might pass when it shouldn't have. Conversely, if users cannot add votes to a proposal due to overflow, it might fail when it should have passed.
    *   This could lead to `Governance::next_trade_params` (or other governable parameters) being set based on incomplete or "stuck" voting states, or reverting to current parameters if a winning proposal effectively loses its true quorum.

### 5. Recommended Mitigations (CALC-010)

1.  **Primary: Use `u128` for `Proposal::votes`**:
    *   Change the type of `Proposal::votes` from `u64` to `u128`.
    *   Since `stake_to_voting_power` returns `u64` (or would be changed to `u128` per CALC-011 mitigation), additions and subtractions to `Proposal::votes` would then be `u128` arithmetic, making overflows practically impossible.

2.  **Address Underflow Logic for Vote Changes**:
    *   **Ideal Solution**: When a user changes a vote, the system should subtract the *exact amount of voting power they initially contributed to the `from_proposal_id`*, not their current total voting power. This requires storing more state, e.g., a record of `(user_id, proposal_id, voting_power_at_time_of_vote)` for each vote cast. When changing a vote, this specific record would be found and its `voting_power_at_time_of_vote` subtracted. This is complex.
    *   **Alternative (if `Proposal::votes` is `u128` and `stake_to_voting_power` also `u128`)**: If the user's current `stake_to_voting_power` is used for subtraction, an underflow on a `u128` `Proposal::votes` is extremely unlikely. However, it's still logically more correct to subtract what was originally added.
    *   **Saturating Subtraction (Less Ideal)**: Using `proposal.votes = proposal.votes.saturating_sub(votes_to_add_or_remove)` would prevent the panic. However, if a user's current voting power (`votes_to_add_or_remove`) is much larger than their original contribution and also larger than the proposal's current total votes, this could inaccurately reduce `proposal.votes` to zero. This might be preferable to a DoS but still leads to incorrect accounting.

The most robust solution involves changing `Proposal::votes` to `u128` and ideally refining the logic for vote changes to subtract the historically contributed voting power.
---
