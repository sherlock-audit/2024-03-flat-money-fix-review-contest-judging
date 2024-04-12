Bent Menthol Turtle

medium

# Collateral cap is not correctly checked due to unrealized PnL

## Summary

Before LPs deposit funds into the protocol, there is a check to ensure that the total collateral doesn't surpass the cap set by the admins. However, this check doesn't account for unrealized PnL of open leverage positions and that will lead to two different issues. 

## Vulnerability Detail

When LPs deposit funds into the protocol, they call `announceStableDeposit` function at `DelayedOrder` contract. In that function, there is a check that ensures that the total collateral deposited by LPs doesn't surpass the cap set by the admins:

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/DelayedOrder.sol#L75
```solidity
vault.checkCollateralCap(depositAmount);
```

The function `checkCollateralCap` is the following:

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L324-L329
```solidity
    function checkCollateralCap(uint256 _depositAmount) public view {
        uint256 collateralCap = stableCollateralCap;

        if (stableCollateralTotal + _depositAmount > collateralCap)
            revert FlatcoinErrors.DepositCapReached(collateralCap);
    }
```

However, the variable `stableCollateralTotal` doesn't include the global profits and losses of the opened positions on the market. Therefore, the value of `stableCollateralTotal` is not accurate because it doesn't reflect the actual liquidity LPs can access. 

This will cause two problematic scenarios:

**1. Price of ETH goes up**: The profits made by the leverage positions (losses by LPs) won't be accounted into `stableCollateralTotal`, so when LPs want to deposit more liquidity, the check will fail because `stableCollateralTotal` will be a higher value than it should. 

**1. Price of ETH goes down**: The losses made by the leverage positions (profits by LPs) won't be accounted into `stableCollateralTotal`, so LPs will be able to deposit too much liquidity to the market, thereby making the cap meaningless. 

## Impact

When the price of ETH goes up and some leverage positions are open, LPs won't be able to deposit more liquidity. 
When the price of ETH goes down and some leverage positions are open, LPs will be able to deposit too much liquidity, making the cap meaningless. 

## Code Snippet

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L324-L329

## Tool used

Manual Review

## Recommendation

Include the global profits and losses into the `checkCollateralCap()` function.
