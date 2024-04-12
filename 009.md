Bent Menthol Turtle

high

# When the ETH price goes down, LPs won't be able to withdraw liquidity

## Summary

When the ETH price goes down and there are still some leverage positions open that have not realized their losses, LPs won't be able to withdraw liquidity.

## Vulnerability Detail

Consider the following scenario, we're going to ignore the funding fee for the sake of simplicity:

1. The market has 10 rETH of total stable collateral and `stableCollateralPerShare` is `1e18` (total shares is `10e18`). 
2. After some time, the price of ETH goes down, causing the following:
    a. Total stable collateral after settlement increases to 20 rETH.
    b. `stableCollateralPerShare` increases to `2e18`
3. LP tries to withdraw 11 rETH (`5.5e18` shares) --> transaction will fail with an error `PriceImpactDuringWithdraw`.

This has happened because no leverage positions have been closed, so their loss hasn't been realized yet. This will cause the variable `stableCollateralTotal` to be `10e18` (for the initial liquidity of 10 rETH) instead of `20e18`. So, when the LP tries to withdraw, `stableCollateralTotal` will be capped at zero, and that will increase the value of `stableCollateralPerShare` too much, reverting the whole transaction.

Here is the function to withdraw liquidity:

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/StableModule.sol#L93-L138
```solidity
    function executeWithdraw(
        address _account,
        uint64 _executableAtTime,
        FlatcoinStructs.AnnouncedStableWithdraw calldata _announcedWithdraw
    ) external onlyAuthorizedModule returns (uint256 _amountOut, uint256 _withdrawFee) {
        uint256 withdrawAmount = _announcedWithdraw.withdrawAmount;

        uint32 maxAge = _getMaxAge(_executableAtTime);

        uint256 stableCollateralPerShareBefore = stableCollateralPerShare(maxAge);
        _amountOut = (withdrawAmount * stableCollateralPerShareBefore) / (10 ** decimals());

        // Unlock the locked LP tokens before burning.
        // This is because if the amount to be burned is locked, the burn will fail due to `_beforeTokenTransfer`.
        _unlock(_account, withdrawAmount);

        _burn(_account, withdrawAmount);

>>      vault.updateStableCollateralTotal(-int256(_amountOut));

        uint256 stableCollateralPerShareAfter = stableCollateralPerShare(maxAge);

        // Check that there is no significant impact on stable token price.
        // This should never happen and means that too much value or not enough value was withdrawn.
        if (totalSupply() > 0) {
            if (
                stableCollateralPerShareAfter < stableCollateralPerShareBefore - 1e6 ||
                stableCollateralPerShareAfter > stableCollateralPerShareBefore + 1e6
>>          ) revert FlatcoinErrors.PriceImpactDuringWithdraw();

            // ...
    }
```

When the function `updateStableCollateralTotal` is called, `stableCollateralTotal` will be capped at zero:

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L442-L448
```solidity
    function _updateStableCollateralTotal(int256 _stableCollateralAdjustment) private {
        int256 newStableCollateralTotal = int256(stableCollateralTotal) + _stableCollateralAdjustment;

        // The stable collateral shouldn't be negative as the other calculations which depend on this
        // will behave in unexpected manners.
        stableCollateralTotal = (newStableCollateralTotal > 0) ? uint256(newStableCollateralTotal) : 0;
    }
```

The value of `stableCollateralTotal` will be capped at zero because `_stableCollateralAdjustment` is `-11e18` and `stableCollateralTotal` is `10e18`. After this, when `stableCollateralPerShareAfter` is calculated, the result will be significantly higher than `stableCollateralPerShareBefore` because of this behavior.

```solidity
BEFORE:
stableCollateralTotal = 10e18
stableCollateralTotalAfterSettlement = 20e18
totalSupply = 10e18
stableCollateralPerShareBefore = 2e18 (20e18 * 1e18 / 10e18)

AFTER: 
stableCollateralTotal = 0
stableCollateralTotalAfterSettlement = 10e18 (should be `9e18`)
totalSupply = 4.5e18
stableCollateralPerShareAfter = 2.2e18 (10e18 * 1e18 / 4.5e18)
```

Because the variables `stableCollateralPerShareBefore` and `stableCollateralPerShareAfter` have such a difference, the whole transaction will revert with the error `PriceImpactDuringWithdraw`. 

## Impact

When the price of ETH goes down, LPs won't be able to withdraw liquidity as they should

## Code Snippet

See above

## Tool used

Manual Review

## Recommendation

Allow the value of `stableCollateralTotal` to go below zero to deal with these edge cases. 
