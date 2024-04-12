Wobbly Myrtle Mantaray

high

# Functions using the invariance modifiers could still lead to DOS


## Summary

There is an edge case where the stable collateral total is not adjusted to it's real values to avoid a negative value. This adjustments would easily cause an imbalance in the collateral net balance that will revert the transaction when any function called uses say the `orderInvariantChecks()` or `liquidationInvariantChecks()` modifiers.

## Vulnerability Detail

First, from this section of the [readMe](https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest?tab=readme-ov-file#q-should-potential-issues-like-broken-assumptions-about-function-behavior-be-reported-if-they-could-pose-risks-in-future-integrations-even-if-they-might-not-be-an-issue-in-the-context-of-the-scope-if-yes-can-you-elaborate-on-propertiesinvariants-that-should-hold) we can see that protocol has clearly requested we submit bug ideas that break the invariant checks, to quote them:

> - The protocol has hardcoded invariant checks in `InvariantChecks.sol`. If these can be made to revert, then we should know about it.

Now, take a look at https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/539341fdc92090c48792c454e35e5d739ee62820/flatcoin-v1/src/FlatcoinVault.sol#L442-L449

```solidity
    function _updateStableCollateralTotal(int256 _stableCollateralAdjustment) private {
        int256 newStableCollateralTotal = int256(stableCollateralTotal) + _stableCollateralAdjustment;

        // The stable collateral shouldn't be negative as the other calculations which depend on this
        // @audit
        // will behave in unexpected manners.
        stableCollateralTotal = (newStableCollateralTotal > 0) ? uint256(newStableCollateralTotal) : 0;
    }

```

We can see that this function adjusts the stable collateral total by adding/subtracting to it `_stableCollateralAdjustment`, now when this value is negative and greater than the stable collateral total in absolute value, stableCollateralTotal is set to 0.

Since this edge case does not adjust the stable collateral total to its real value, this leads to an imbalance in the accounting of the calculation of the collateral net balance seen here https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/539341fdc92090c48792c454e35e5d739ee62820/flatcoin-v1/src/misc/InvariantChecks.sol#L94-L103

```solidity
    function _getCollateralNet(IFlatcoinVault vault) private view returns (int256 netCollateral) {
        int256 collateralBalance = int256(vault.collateral().balanceOf(address(vault)));
        int256 trackedCollateral = int256(vault.stableCollateralTotal()) +
            vault.getGlobalPositions().marginDepositedTotal;

        if (collateralBalance < trackedCollateral) revert FlatcoinErrors.InvariantViolation("collateralNet1");

        return collateralBalance - trackedCollateral;
    }

```

Keep in mind that the `_getCollateralNet()` is always called in the invariant checks modifiers, i.e the `orderInvariantChecks` & `liquidationInvariantChecks` modifiers as can be seen [here](https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/539341fdc92090c48792c454e35e5d739ee62820/flatcoin-v1/src/misc/InvariantChecks.sol#L60-L74) and [here](https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/539341fdc92090c48792c454e35e5d739ee62820/flatcoin-v1/src/misc/InvariantChecks.sol#L33-L45), so having the implementation of `_getCollateralNet()` not using the imbalanced value for the stable collateral, this leads to [this check](https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/539341fdc92090c48792c454e35e5d739ee62820/flatcoin-v1/src/misc/InvariantChecks.sol#L98-L100) to revert and essentially a reversion to all functions that integrate with these modifiers.

Keep in mind that from this `will fix` tagged issue: [here](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues?q=is%3Aissue+is%3Aopen+Long+trader%27s+deposited+margin+can+be+wiped+out) and it's duplicates, namely: [1](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues?q=is%3Aissue+DoS+for+functions+with+invariant+modifiers+is%3Aclosed) we can see that protocol intended to fix this and infact applied a fix, i.e to the `settleFundingFees()` instance where [it no longer sets the margin deposit total value to 0 ](https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/539341fdc92090c48792c454e35e5d739ee62820/flatcoin-v1/src/FlatcoinVault.sol#L238-L242) compared to the previous codebase, where [the margin deposit total value is set to 0 when the funding fees are greater than the margin deposited total](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L230-L234), however a similar fix is missing in the `_updateStableCollateralTotal()` execution which is what this report is about.

## Impact

Break in protocol's core functionality cause now functions LiquidationModule.liquidate(), LimitOrder.executeLimitOrder() & DelayedOrder.executeOrder() will not be executable when the stable collateral total is not adjusted to it's real value, due to the `orderInvariantChecks` and `liquidationInvariantChecks` modifiers reverting the transactions.

Putting the unavailability of `LimitOrder.executeLimitOrder() & DelayedOrder.executeOrder() ` aside, keep in mind that the unavailability to `LiquidationModule.liquidate()` alone means protocol would now be engaging with bad debt essentially loss of funds, since liquidations can't go through, which makes the severity for this report High.

## Code Snippet

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/539341fdc92090c48792c454e35e5d739ee62820/flatcoin-v1/src/misc/InvariantChecks.sol#L94-L103

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/539341fdc92090c48792c454e35e5d739ee62820/flatcoin-v1/src/misc/InvariantChecks.sol#L33-L45

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/539341fdc92090c48792c454e35e5d739ee62820/flatcoin-v1/src/misc/InvariantChecks.sol#L60-L74

## Tool used

- Manual Review
- Will fix tagged issues from previous contest, [1](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues?q=is%3Aissue+is%3Aopen+Long+trader%27s+deposited+margin+can+be+wiped+out), [2](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues?q=is%3Aissue+DoS+for+functions+with+invariant+modifiers+is%3Aclosed) & [3](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues?q=is%3Aissue+User+can+abuse+of+settleFundingFees+method+to+prevent+being+liquidated+is%3Aclosed).

## Recommendation

Consider reimplementing the way these modifiers are used in very trivial functions that must be executed like `liquidate()` and the rest or adapting to account for this edge case.