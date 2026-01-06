# Fix for SubnetAlphaOut Accounting Discrepancies (Issue #2274)

## Problem Description

The issue identified discrepancies between `SubnetAlphaOut` (accounting variable for total active alpha) and `TotalHotkeyAlpha` (sum of actual stake).

The primary evidence showed **Negative Burn** (Phantom Inflation): `SubnetAlphaOut` became significantly lower than `TotalHotkeyAlpha`. This means the system accounted for *less* issued alpha than what users actually held.

## Investigation

The investigation revealed that `BalanceOps::decrease_stake` in `pallets/subtensor/src/lib.rs` was incorrectly updating `SubnetAlphaOut`.

The faulty implementation was:
1.  Decrement `SubnetAlphaOut` by the *requested* amount.
2.  Call `decrease_stake_for_hotkey_and_coldkey_on_subnet` to remove the stake from the user.

If the user had insufficient stake (e.g. requested removal of 100 but only had 10), the inner function would return `0` (or partial amount) and not remove the full requested stake. However, `SubnetAlphaOut` was already decremented by the full requested amount.

This led to `SubnetAlphaOut` decreasing while `TotalHotkeyAlpha` remained unchanged, causing `SubnetAlphaOut < TotalHotkeyAlpha` (Negative Burn).

## Solution

The fix involves refactoring `decrease_stake` in `pallets/subtensor/src/lib.rs` to:
1.  Perform the stake removal first, capturing the **actual** amount removed.
2.  Decrement `SubnetAlphaOut` by the **actual** amount removed.

This ensures `SubnetAlphaOut` stays perfectly in sync with `TotalHotkeyAlpha` during stake decreases.

## Note on "Positive Burn"

The original issue also mentioned "The burn extrinsic: reducing a coldkey's stake without reducing subnet_alpha_out" as a problem. However, upon review, this behavior is intended for the `burn` extrinsic. Burning creates a discrepancy between Issued (`SubnetAlphaOut`) and Owned (`TotalHotkeyAlpha`), effectively removing tokens from circulation (making them inaccessible) while maintaining the issuance record. Therefore, `SubnetAlphaOut` should **not** be decreased when explicitly burning alpha.

## Verification

Regression tests were added in `pallets/subtensor/src/tests/regression_issue_2274.rs` to verify:
1.  **Negative Burn Fix**: Attempting to decrease more stake than available does *not* reduce `SubnetAlphaOut` incorrectly.
2.  **Burn Behavior**: Explicitly burning alpha reduces user stake but *preserves* `SubnetAlphaOut`, correctly reflecting burned tokens.
