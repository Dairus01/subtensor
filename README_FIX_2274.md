# Fix for SubnetAlphaOut Accounting Discrepancies (Issue #2274)

## Problem Description

The issue identified a growing discrepancy between `SubnetAlphaOut` (the accounting variable tracking total active alpha on a subnet) and `TotalHotkeyAlpha` (the sum of actual stake held by all hotkeys on that subnet).

This discrepancy manifested in two ways:

1.  **Positive Burn (Drift)**: `SubnetAlphaOut` becomes greater than `TotalHotkeyAlpha`. This means the system thinks there is more alpha in circulation than actually exists in accounts. This effectively "burns" alpha from the perspective of accessibility, but the accounting doesn't reflect it properly as burned (removed from supply).
2.  **Negative Burn (Inflation)**: `SubnetAlphaOut` becomes less than `TotalHotkeyAlpha`. This means there is more alpha in accounts than the system accounting believes exists. This is a critical accounting error resembling phantom inflation or "negative burn".

The evidence provided showed that `SubnetAlphaOut` was significantly lower than `TotalHotkeyAlpha` on several subnets, indicating a strong prevalence of the "Negative Burn" scenario, likely masking any "Positive Burn" issues.

## Investigation & Thought Process

### 1. Analyzing Positive Burn (The Burn Extrinsic)
The issue description pointed out that the `burn` extrinsic reduced a coldkey's stake without reducing `SubnetAlphaOut`.

*   **Investigation**: I examined `pallets/subtensor/src/staking/helpers.rs` and found the `burn_subnet_alpha` function.
    ```rust
    pub fn burn_subnet_alpha(_netuid: NetUid, _amount: AlphaCurrency) {
        // Do nothing; TODO: record burned alpha in a tracker
    }
    ```
*   **Finding**: The function was indeed empty. When a user burned alpha, their personal stake (`TotalHotkeyAlpha`) decreased, but the global subnet counter (`SubnetAlphaOut`) remained unchanged.
*   **Result**: `SubnetAlphaOut > TotalHotkeyAlpha` (Positive Drift).

### 2. Analyzing Negative Burn (Incentive/Stake Mismatch)
The evidence showed `SubnetAlphaOut < TotalHotkeyAlpha` (Negative Drift). This implies `SubnetAlphaOut` was being reduced *more* than it should be, or `TotalHotkeyAlpha` was increasing without `SubnetAlphaOut` increasing.

*   **Investigation**: I traced the `BalanceOps::decrease_stake` implementation in `pallets/subtensor/src/lib.rs`.
    ```rust
    fn decrease_stake(..., amount: AlphaCurrency) -> ... {
        // ... checks ...

        // 1. BLINDLY decrement SubnetAlphaOut
        SubnetAlphaOut::<T>::mutate(netuid, |total| {
            *total = total.saturating_sub(amount);
        });

        // 2. Attempt to decrement user stake
        Ok(Self::decrease_stake_for_hotkey_and_coldkey_on_subnet(..., amount))
    }
    ```
*   **Finding**: The code decremented `SubnetAlphaOut` by the *requested* `amount` immediately. Then it called `decrease_stake_for_hotkey_and_coldkey_on_subnet`.
    Inside `decrease_stake_for_hotkey_and_coldkey_on_subnet` (in `stake_utils.rs`), if the user had insufficient stake (or precision issues occurred), it might return `0` or a partial amount, *without* reducing the user's stake fully.
*   **Scenario**: A user requests to unstake 100 Alpha. They only have 10 Alpha.
    1. `SubnetAlphaOut` reduces by 100.
    2. `decrease_stake_for_hotkey...` sees insufficient funds, returns `0` (or fails silently in terms of state change depending on exact logic path, but often returns actual decreased amount).
    3. Result: `SubnetAlphaOut` -100. `TotalHotkeyAlpha` -0.
*   **Result**: `SubnetAlphaOut` drops significantly below `TotalHotkeyAlpha`. This matches the "Negative Burn" evidence.

## Solution

### 1. Fix Positive Burn
I updated `burn_subnet_alpha` in `pallets/subtensor/src/staking/helpers.rs` to correctly decrement the `SubnetAlphaOut` storage value.

```rust
pub fn burn_subnet_alpha(netuid: NetUid, amount: AlphaCurrency) {
    SubnetAlphaOut::<T>::mutate(netuid, |total| {
        *total = total.saturating_sub(amount);
    });
}
```

### 2. Fix Negative Burn
I refactored `decrease_stake` in `pallets/subtensor/src/lib.rs` to prioritize the actual stake removal logic and only update `SubnetAlphaOut` based on the *actual* amount removed.

```rust
fn decrease_stake(..., amount: AlphaCurrency) -> ... {
    // ... checks ...

    // 1. Perform stake decrease first, capturing the ACTUAL amount removed
    let actual_alpha = Self::decrease_stake_for_hotkey_and_coldkey_on_subnet(
        hotkey, coldkey, netuid, amount,
    );

    // 2. Decrement SubnetAlphaOut by the ACTUAL amount
    SubnetAlphaOut::<T>::mutate(netuid, |total| {
        *total = total.saturating_sub(actual_alpha);
    });

    Ok(actual_alpha)
}
```

## Verification
Reproduction tests were created to confirm:
1.  **Positive Burn Fix**: Burning alpha now reduces `SubnetAlphaOut` equally.
2.  **Negative Burn Fix**: Attempting to decrease more stake than available (or 0 stake) no longer reduces `SubnetAlphaOut` incorrectly. `SubnetAlphaOut` stays in sync with the actual stake change.
