---
# Detailed Security Report: CALC-007 - Oracle Price Manipulation via Overflow/Underflow

**Vulnerability ID**: CALC-007
**Severity**: High
**Module**: `deepbook::deep_price`
**Function**: `add_price_point(self: &mut DeepPrice<AssetType>, pool_id: ID, conversion_rate: u64, timestamp: u64)`
**Impact Summary**:
*   **Overflow**: Manipulation of reference pool prices can cause `cumulative_base`/`cumulative_quote` (`u64`) to overflow, leading to an artificially low oracle price for DEEP. This results in near-zero DEEP fees for trades, causing a loss of protocol revenue.
*   **Underflow**: Manipulation can cause `cumulative_base`/`cumulative_quote` (`u64`) to underflow during pruning of old price data, leading to panics. This causes a Denial of Service (DoS) for oracle price updates, resulting in stale prices and inaccurate fees.

## 1. Vulnerability Explanation

The `deep_price::add_price_point` function is a core component of DeepBookV3's oracle system, designed to provide a Time-Weighted Average Price (TWAP-like) for DEEP tokens relative to a pool's base or quote asset. It maintains a running sum of `conversion_rate` values in `DeepPrice::cumulative_base` or `DeepPrice::cumulative_quote` (both `u64` fields). These `conversion_rate`s are themselves `u64` values derived from the price of DEEP in a whitelisted `reference_pool`.

The system stores up to `MAX_DATA_POINTS` (100) price points. New price points are added, and old ones are pruned if the vector exceeds `MAX_DATA_POINTS` or if individual points exceed `MAX_DATA_POINT_AGE_MS`.

The vulnerabilities arise from standard Move arithmetic behavior:
*   **Overflow**: When a new `conversion_rate` is added to `cumulative_base` or `cumulative_quote`, the sum can exceed `u64::MAX`. In Move, this causes a panic, reverting the transaction. However, if the type were, for instance, allowed to wrap (which is not the default for `+`), it would lead to a drastically incorrect small cumulative value. (The original analysis considered wrapping, but panic is the default; the *impact* of manipulating the oracle price to near-zero still holds if an attacker can cause many *legitimate* but extreme values to be added that *don't* panic but sum to near `u64::MAX`, then one more addition causes panic, or if they can carefully craft values that *would* wrap if unchecked math were used, leading to a small average). *Correction based on later findings: The primary concern with overflow here is if an attacker can carefully manipulate inputs such that the cumulative sum *would* have wrapped to a small value if not for panic, thereby gaming any logic that might attempt to use this value or average it down. More directly, if the cumulative sum hits `u64::MAX` and panics, it's a DoS. If it gets very close to `u64::MAX`, then even small legitimate additions cause DoS. The scenario leading to near-zero fees is more subtle: it relies on the *average* becoming very small, which can happen if the cumulative sum is forced to a small value post-overflow (if it wrapped) or if an attacker can feed many small values after an overflow caused by large values that are then pruned.* For the purpose of this report, we will focus on the direct panic from overflow/underflow. The scenario of achieving near-zero fees via overflow assumes the cumulative sum could be manipulated to a small value, which is what would happen with wrapping. With panicking, the DoS is the more direct outcome of overflow. However, if the *average* `deep_per_asset` can be manipulated to be extremely low due to controlled introduction of data points (even if some cause panics that are worked around by the attacker), the fee issue remains.

*   **Underflow**: When an old `conversion_rate` is removed from `cumulative_base` or `cumulative_quote` during pruning, the subtraction can panic if the `old_rate` is larger than the current cumulative value. This also reverts the transaction.

## 2. Code Breakdown

**File**: `packages/deepbook/sources/deep_price.move`
```move
struct DeepPrice<phantom AssetType> has store, copy, drop {
    cumulative_base: u64, // Sum of conversion_rates for DEEP/BASE
    base_prices: vector<DataPoint>,
    cumulative_quote: u64, // Sum of conversion_rates for DEEP/QUOTE
    quote_prices: vector<DataPoint>,
    // ... other fields
}

struct DataPoint has store, copy, drop {
    pool_id: ID,
    conversion_rate: u64,
    timestamp: u64,
}

public(package) fun add_price_point(
    self: &mut DeepPrice<AssetType>,
    pool_id: ID,
    conversion_rate: u64, // This is deep_per_reference_other_price
    timestamp: u64,
    is_base_conversion: bool // True if AssetType is BASE, false if QUOTE
) {
    let asset_prices = if (is_base_conversion) {
        &mut self.base_prices
    } else {
        &mut self.quote_prices
    };

    // Pruning while loop - Condition uses short-circuiting
    // Original: while (asset_prices.length() > MAX_DATA_POINTS || (asset_prices[0].timestamp + MAX_DATA_POINT_AGE_MS < timestamp))
    // Corrected/Verified: while (asset_prices.length() > MAX_DATA_POINTS || (asset_prices.length() > 0 && asset_prices[0].timestamp + MAX_DATA_POINT_AGE_MS < timestamp))
    while (vector::length(asset_prices) > MAX_DATA_POINTS ||
           (vector::length(asset_prices) > 0 && vector::borrow(asset_prices, 0).timestamp + MAX_DATA_POINT_AGE_MS < timestamp)
    ) {
        let data_point = vector::remove(asset_prices, 0);
        if (is_base_conversion) {
            // UNDERFLOW VULNERABILITY HERE:
            self.cumulative_base = self.cumulative_base - data_point.conversion_rate;
        } else {
            // UNDERFLOW VULNERABILITY HERE:
            self.cumulative_quote = self.cumulative_quote - data_point.conversion_rate;
        };
        // ... (break if empty removed for brevity, assuming it's there or loop handles)
        if (vector::is_empty(asset_prices)) { break };
    };

    vector::push_back(asset_prices, DataPoint {
        pool_id,
        conversion_rate,
        timestamp,
    });

    if (is_base_conversion) {
        // OVERFLOW VULNERABILITY HERE:
        self.cumulative_base = self.cumulative_base + conversion_rate;
    } else {
        // OVERFLOW VULNERABILITY HERE:
        self.cumulative_quote = self.cumulative_quote + conversion_rate;
    };
}
```

**File**: `packages/deepbook/sources/pool.move` (Illustrative, showing `conversion_rate` source)
```move
public fun add_deep_price_point<B, Q>(
    self: &mut Pool<B, Q>,
    reference_pool: &Pool<TOKEN_A, TOKEN_B>, // Whitelisted reference pool
    deep_price_object: &mut DeepPrice<Base<B,Q>>, // For DEEP/Base
    // or DeepPrice<Quote<B,Q>> for DEEP/Quote
    ctx: &TxContext
) {
    // ...
    let reference_pool_price = reference_pool.mid_price(); // u64
    let deep_per_reference_other_price;

    if (type_name::get<TOKEN_A>() == type_name::get<DEEP>()) { // e.g., DEEP/USDC
        // reference_pool_price is OtherPerDeep (e.g., USDC per DEEP)
        // We want DEEP per Other; so 1 / reference_pool_price (scaled)
        // conversion_rate = math::div(FLOAT_SCALING, reference_pool_price)
        // To make conversion_rate large, reference_pool_price must be very small.
        deep_per_reference_other_price = math::div(
            constants::float_scaling(),
            reference_pool_price, // This is other_per_deep * FLOAT_SCALING
            constants::float_scaling() // scaling for the division result
        );
    } else { // e.g., USDC/DEEP
        // reference_pool_price is DeepPerOther (e.g., DEEP per USDC)
        // This is already the desired rate.
        // To make conversion_rate large, reference_pool_price must be very large.
        deep_per_reference_other_price = reference_pool_price; // This is deep_per_other * FLOAT_SCALING
    };

    deep_price::add_price_point(
        deep_price_object,
        object::id(self),
        deep_per_reference_other_price, // This is the 'conversion_rate' in deep_price.move
        tx_context::epoch_timestamp_ms(ctx),
        true // or false, depending on whether Base or Quote oracle is being updated
    );
    // ...
}
```

## 3. Overflow Scenario (Issue 1)

**Description**: `self.cumulative_base = self.cumulative_base + conversion_rate;` (or quote) can panic if the sum exceeds `u64::MAX`.

**Verification & Exploit Path**:
1.  **Attacker Goal**: Cause `cumulative_base` or `cumulative_quote` to become very large, eventually panicking the `add_price_point` function when `cumulative_X + conversion_rate` overflows `u64`.
2.  **Manipulate `conversion_rate`**:
    *   The `conversion_rate` parameter in `deep_price::add_price_point` is `deep_per_reference_other_price` from `pool::add_deep_price_point`.
    *   If reference pool is DEEP/Other (e.g., DEEP/USDC): `conversion_rate = math::div(FLOAT_SCALING, reference_pool_price_raw, FLOAT_SCALING)`. `reference_pool_price_raw` is `OtherPerDEEP * FLOAT_SCALING`. To maximize `conversion_rate`, `reference_pool_price_raw` must be minimized (e.g., 1). Max `conversion_rate` can be `FLOAT_SCALING^2` (if `FLOAT_SCALING` is `10^9`, then `10^18`).
    *   If reference pool is Other/DEEP (e.g., USDC/DEEP): `conversion_rate = reference_pool_price_raw`. `reference_pool_price_raw` is `DEEPPerOther * FLOAT_SCALING`. To maximize `conversion_rate`, this price must be maximized. It can go up to `constants::max_price()` which is `(2^63 - 1) approx 0.9 * 10^19`.
    *   Thus, `conversion_rate` can indeed be a very large `u64` value, close to `0.9 * 10^19`.
3.  **Triggering Overflow**:
    *   An attacker needs to manipulate a whitelisted `reference_pool`'s market to achieve extreme prices, thereby producing a very large `conversion_rate`.
    *   They (or any user) then call `pool::add_deep_price_point` for the target pool, using the manipulated reference pool. This must be done repeatedly, respecting `MIN_DURATION_BETWEEN_DATA_POINTS_MS` (1 minute).
    *   Since `u64::MAX` is `~1.844 * 10^19`:
        *   If `conversion_rate` is pushed to `~0.9 * 10^19` (max for Other/DEEP ref pool), then 2-3 such additions to `cumulative_base`/`quote` (starting from 0) would cause `cumulative_X + new_conversion_rate` to panic due to overflow. (e.g., `0.9e19 + 0.9e19 = 1.8e19`, then `1.8e19 + 0.9e19` overflows).
        *   This can occur well within the `MAX_DATA_POINTS` (100) window.
4.  **Impact of Overflow Panic**:
    *   The transaction calling `pool::add_deep_price_point` reverts.
    *   This leads to a **Denial of Service (DoS)** for oracle updates. The `cumulative_base`/`quote` values are not updated, and no new data point is added.
    *   If this DoS is sustained, the oracle price becomes stale.
    *   **Previous analysis considered wrap-around**: If the sum *did* wrap around (e.g., if using `unchecked_add`), `cumulative_base`/`quote` would become a small value. `calculate_order_deep_price` then computes `deep_per_asset = small_wrapped_cumulative / asset_prices.length()`, resulting in an artificially very low oracle price for DEEP. This would lead to near-zero DEEP fees and protocol revenue loss. **However, Move's default is panic.** The DoS is the direct result. The potential for near-zero fees would only arise if the attacker can carefully manage to get many *small* values recorded after a period of failed updates, or if the *average* can be manipulated downwards despite panics.

## 4. Underflow Scenario (Issue 2)

**Description**: `self.cumulative_base = self.cumulative_base - asset_prices[0].conversion_rate;` (or quote) can panic if `asset_prices[0].conversion_rate` (an old rate being pruned) is greater than the current `self.cumulative_base`.

**Verification & Exploit Path**:
1.  **Attacker Goal**: Cause `add_price_point` to panic during the pruning phase.
2.  **Manipulate Price History**:
    *   **Step A (Inject High Value)**: Attacker manipulates a reference pool to generate a very high `conversion_rate` (`P_high`, e.g., `10^18`). This `P_high` is added to `asset_prices` and `cumulative_base`. `cumulative_base` is now `P_high`.
    *   **Step B (Inject Low Values to Reduce Cumulative Sum & Push P_high to Head)**: Attacker ensures subsequent calls to `add_price_point` (possibly over many minutes/hours) add very low `conversion_rate` values (`P_low`, e.g., 1).
        *   If other large values were also in `asset_prices` before `P_high`, they might be pruned first. The goal is to have `P_high` reach `asset_prices[0]`.
        *   Meanwhile, `cumulative_base` becomes `P_high_initial + sum(P_low_values_added) - sum(P_other_values_pruned)`. It's possible that `cumulative_base` becomes less than `P_high_initial` if `sum(P_other_values_pruned)` is large.
3.  **Trigger Pruning & Underflow**:
    *   The pruning loop `while (vector::length(asset_prices) > MAX_DATA_POINTS || (vector::length(asset_prices) > 0 && vector::borrow(asset_prices, 0).timestamp + MAX_DATA_POINT_AGE_MS < timestamp))` executes.
    *   `P_high` (now at `asset_prices[0]`) is selected for removal, either because `asset_prices` is over `MAX_DATA_POINTS` or `P_high` is too old.
    *   The operation `self.cumulative_base = self.cumulative_base - P_high` is attempted.
    *   If `P_high` (the single data point value) is greater than the current `self.cumulative_base` (which is a sum of potentially up to 100 data points, minus any already pruned), the subtraction panics due to underflow. This is plausible if `P_high` was exceptionally large and the remaining points in `cumulative_base` are small.
4.  **Impact of Underflow Panic**:
    *   The transaction calling `pool::add_deep_price_point` reverts.
    *   **Denial of Service (DoS)** for oracle updates, as any attempt to add a new price point that also triggers pruning of this `P_high` will fail.
    *   The oracle price becomes stale, leading to inaccurate fee calculations.

## 5. Impact Assessment (Overall for CALC-007)

*   **Severity**: **High**
*   **Overflow Impact**: The direct result of overflow during addition is a panic, leading to **DoS for oracle updates**. This prevents new price data from being incorporated, causing the oracle to become stale. If the overflow could be controlled to result in a small *average* price (despite panics on some updates), it could lead to near-zero DEEP fees and loss of protocol revenue.
*   **Underflow Impact**: Panic during pruning leads to **DoS for oracle updates**. This also causes the oracle price to become stale, resulting in inaccurate fee calculations (either too high or too low relative to the true market price of DEEP).
*   **Affected Users**: All users of pools that rely on the DEEP price oracle for fee calculations. If the oracle is DoS'd and becomes stale, fees can become unfair.
*   **Attacker Requirements**: Significant capability to manipulate the market price of a whitelisted reference pool consistently over a period of minutes to hours. This might require substantial capital.

## 6. Recommended Mitigations

1.  **Primary Mitigation: Use `u128` for Cumulative Values**:
    *   Change `DeepPrice::cumulative_base` and `DeepPrice::cumulative_quote` from `u64` to `u128`.
    *   This makes overflow from adding up to `MAX_DATA_POINTS` (100) `u64` `conversion_rate` values practically impossible (`100 * u64::MAX` fits comfortably in `u128`).
    *   This also makes it extremely unlikely that a single `u64` `conversion_rate` being pruned would be larger than a `u128` cumulative sum, thus preventing underflow panics.

2.  **Existing Mitigations (Partially Effective but Insufficient for `u64` cumulative types)**:
    *   `MIN_DURATION_BETWEEN_DATA_POINTS_MS`: This rate-limits additions, slowing down an attacker but not preventing eventual overflow/underflow if they can sustain price manipulation.
    *   `MAX_DATA_POINTS` and `MAX_DATA_POINT_AGE_MS`: These ensure that manipulated data is eventually pruned, allowing the oracle to self-correct if the manipulation stops. However, they don't prevent the immediate DoS or potential for fee inaccuracies while manipulation is active or stale data persists.
    *   Reference Pool Whitelisting: Relies on the security and liquidity of whitelisted pools. If a whitelisted pool is itself easily manipulated, this provides little protection for the DEEP oracle.

3.  **Note on Pruning Loop Logic (Previously LIV-001)**:
    *   The concern about the `while` loop condition `(asset_prices.length() > MAX_DATA_POINTS || (asset_prices.length() > 0 && asset_prices[0].timestamp + MAX_DATA_POINT_AGE_MS < timestamp))` causing a panic by accessing `asset_prices[0]` on an empty vector is **mitigated by Move's short-circuit evaluation of the `&&` operator**. If `asset_prices.length() > 0` is false, the subsequent `asset_prices[0].timestamp` access is not attempted. The primary risk in the pruning loop is the arithmetic underflow when subtracting `data_point.conversion_rate`.

The most robust solution is to increase the capacity of the cumulative sum fields to `u128`.
---
