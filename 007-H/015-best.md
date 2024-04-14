Helpful Linen Wombat

high

# Price deviation check not performed on critical transactions

## Summary

Price deviation checks are not performed on critical transactions, leading to various negative impacts, including asset loss.

## Vulnerability Detail

The following is the diff of the `getPrice()` function before and after the updates. It shows that the price deviation check has been disabled in the `getPrice()` function.

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/OracleModule.sol#L80

```diff
     function getPrice() public view returns (uint256 price, uint256 timestamp) {
-        (price, timestamp) = _getPrice(type(uint32).max);
+        (price, timestamp) = _getPrice(type(uint32).max, false);
     }
```

The following functions rely on the `getPrice()` function, which does not have the price deviation check enabled.

- LeverageModule.getPositionSummary
- LeverageModule.getMarketSummary
- LimitOrder._closePosition
- LiquidationModule.liquidationPrice
- LiquidationModule.canLiquidate
- LiquidationModule.getLiquidationFee
- LiquidationModule.getLiquidationMargin
- KeeperFee.getKeeperFee

In the previous audit, the `getPrice()` function always performs a price deviation check to ensure that the Pyth price does not deviate from the Chainlink oracle price.

As highlighted during the main audit contest and fix review, the `updatePythPrice` modifier cannot enforce a Pyth price update because malicious users can bypass the update of the rETH price via various methods, such as passing in a `updateData` payload to the `updatePythPrice` modifier to update another asset apart from rETH, resulting in the on-chain Pyth's rETH price to remain outdated or stale during the execution of the functions.

Due to technical limitations, the original issue during the main audit contest cannot be fully remediated. The risk was mitigated by enforcing a price deviation check to reduce the impact. However, with the price deviation check removed now, the risk of [Issue 194](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/194) can be realized again in some functions (e.g., execution of limit order), and the risk is no longer considered fully mitigated. Thus, Issue 194 is considered as not fully remediated.

## Impact

For critical transactions that involve moving assets between accounts, such as executing open, adjust, close positions, and liquidation, a price deviation check must always be enabled to prevent malicious users from using the outdated/stale price to gain an advantage.

Following is a list of non-exhaustive issues/impacts that can arise if this requirement is broken:

1. The threshold check within the `executeLimitOrder` function uses the `getPrice()` function without a price deviation check. Malicious keepers/users could execute a limit order (via `executeLimitOrder`) of a victim with a stale/outdated price, resulting in their position being closed even if the limit order's threshold has not reached the actual market price, leading to a loss for the victim. 

## Code Snippet

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/OracleModule.sol#L80

## Tool used

Manual Review

## Recommendation

Ideally, a check should be performed to check that the `priceId` within the `PriceInfo` inside the price update array matches the price ID of the collateral (rETH), and the callers should pass in the latest/most up-to-date market price at the time of execution to completely remediate Issue 194. However, it was understood from the team that due to technical limitations, it is not possible to enforce this.

Thus, to mitigate this risk, it is recommended that a price deviation check be enforced for critical transactions, as the `updatePythPrice` modifier cannot enforce a Pyth price update and can be bypassed.

In addition to that, further reduce the risk by implementing the following measures:

- Setting the appropriate `maxExectabilityAge` (as small as possible) to minimize the windows of opportunity where the onchain's Pyth price deviates significantly from the actual market price.
- Setting `maxDiffPercent` that is as small as possible but still doesn't negatively impact daily trading operations to ensure that even in a scenario where the keepers intend to favor one side of the parties, the impact is minimal.
- Robust keeper ecosystem so that the order would be executed as soon as it passes the `minExecutabilityAge` to minimize the windows of opportunity where the onchain's Pyth price deviates significantly from the actual market price.