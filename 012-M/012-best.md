Bent Menthol Turtle

medium

# Function `checkSkewMax` doesn't take into account the unrealized PnL

## Summary

When checking for the maximum skew allowed in the market, the function doesn't take into account the unrealized PnL. This will cause users may create a greater skew than allowed, or users may not be able to withdraw liquidity when it should be possible. 

## Vulnerability Detail

The maximum skew on the market is checked when an LP wants to withdraw liquidity or a trader wants to reduce/close a position. The function that does that is the following:

```solidity
    function checkSkewMax(uint256 _sizeChange, int256 _stableCollateralChange) public view {
        // check that skew is not essentially disabled
        if (skewFractionMax < type(uint256).max) {
            uint256 sizeOpenedTotal = _globalPositions.sizeOpenedTotal;

            if (stableCollateralTotal == 0) revert FlatcoinErrors.ZeroValue("stableCollateralTotal");
            assert(int256(stableCollateralTotal) + _stableCollateralChange >= 0);

            // if the longs are closed completely then there is no reason to check if long skew has reached max
            if (sizeOpenedTotal + _sizeChange == 0) return;

>>        uint256 longSkewFraction = (int256((sizeOpenedTotal + _sizeChange) * 1e18) /
                (int256(stableCollateralTotal) + _stableCollateralChange)).toUint256();

            if (longSkewFraction > skewFractionMax) revert FlatcoinErrors.MaxSkewReached(longSkewFraction);
        }
    }
```

This function divides the total size opened by the positions on the market (`sizeOpenedTotal`) and divides it by the total collateral owned by the LPs (`stableCollateralTotal`). The issue is that, since the last changes on the codebase, now the global PnL is only settled when individual positions are settled. This will cause the variable `stableCollateralTotal` to not represent the total liquidity when the market has positions with unrealized PnL. 

## Impact

When positions have unrealized profits, meaning the LPs have unrealized losses, the variable `stableCollateralTotal` will be higher than it should be, causing the skew to be wrongly higher, preventing LPs from withdrawing liquidity when they should be allowed to. Also, when this scenario is given, traders won't be able to open new positions on the market when they should be able to. 

When positions have unrealized losses, meaning LPs have unrealized profits, the variable `stableCollateralTotal` will be lower than it should be, causing the skew to be wrongly lower, allowing LPs to withdraw too much liquidity when they should not be allowed to. Also, when this scenario is given, traders will be able to open new positions on the market when they shouldn't be able to, causing a more skewed market. 

*_When I talk about unrealized PnL of the LPs, I'm referring to PnL in terms of rETH hold, not in dollar terms._

## Code Snippet

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L304-L320

## Tool used

Manual Review

## Recommendation

When calculating the maximum skew allowed, we should take into account the unrealized PnL on the market. 
