Helpful Linen Wombat

high

# An accounting error within the protocol could lead to a loss of assets for the affected LPs

## Summary

An accounting error within the protocol could lead to a loss of assets for the affected LPs.

## Vulnerability Detail

Assume the following states:

- Alice (LP) deposits 25 ETH and mints 25 UNIT
- Bob (LP) deposits 25 ETH and mints 25 UNIT
- LP's `stableCollateralTotal` = 25 ETH + 25 ETH = 50 ETH
- Charles (Trader) deposits 10 ETH as margin and opens a long position with the size of 100 ETH (10x leverage)
- `globalPositions.marginDepositedTotal` = 10 ETH (Consists of margin of Charles position)
- Total ETH within the protocol = 60 ETH (25 + 25 + 10)

Assume that the accrued funding fee is 40 ETH, and the long traders must pay the LP (short). Note that these values are intentionally chosen or inflated to demonstrate the accounting/math error in the implementation, and they do not affect the validity of this issue.

The settled margin of Charles's long position is as follows after factoring in the accrued fee. Since Charles's settled margin is below zero, his position can be subjected to liquidation.

```solidity
settled margin = margin deposited + PnL + accrued fee
settled margin = 10 ETH + 0 + (-40 ETH) = -30 ETH
```

Whenever any of the transactions (e.g. announce or execute order, change protocol's parameters) within the protocol is executed, the following `FlatcoinVault.settleFundingFees` function will be executed to settle the global funding fees. In addition, the `FlatcoinVault.settleFundingFees` function is also permissionless and can be executed by anyone at any point in time.

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L224

```solidity
File: FlatcoinVault.sol
224:     function settleFundingFees() public returns (int256 _fundingFees) {
225:         (int256 fundingChangeSinceRecomputed, int256 unrecordedFunding) = _getUnrecordedFunding();
226: 
227:         // Record the funding rate change and update the cumulative funding rate.
228:         cumulativeFundingRate = PerpMath._nextFundingEntry(unrecordedFunding, cumulativeFundingRate);
229: 
230:         // Update the latest funding rate and the latest funding recomputation timestamp.
231:         lastRecomputedFundingRate += fundingChangeSinceRecomputed;
232:         lastRecomputedFundingTimestamp = (block.timestamp).toUint64();
233: 
234:         // Calculate the funding fees accrued to the longs.
235:         // This will be used to adjust the global margin and collateral amounts.
236:         _fundingFees = PerpMath._accruedFundingTotalByLongs(_globalPositions, unrecordedFunding);
237: 
238:         // In the worst case scenario that the last position which remained open is underwater,
239:         // We set the margin deposited total to negative. Once the underwater position is liquidated,
240:         // then the funding fees will be reverted and the total will be positive again.
241:         _globalPositions.marginDepositedTotal = _globalPositions.marginDepositedTotal + _fundingFees;
242: 
243:         _updateStableCollateralTotal(-_fundingFees);
244:     }
```

Assume that the `FlatcoinVault.settleFundingFees` function is executed at this point. Line 241 above will be evaluated as follows. The `globalPositions.marginDepositedTotal` will be updated to -30 ETH.

```solidity
_globalPositions.marginDepositedTotal = _globalPositions.marginDepositedTotal + _fundingFees;
_globalPositions.marginDepositedTotal = 10 ETH + (-40 ETH) = -30 ETH
```

Line 243 above will be evaluated as follows. LP's `stableCollateralTotal` will be updated to 90 ETH. Note that at this point, the LP's `stableCollateralTotal` is actually artificially inflated on paper because out of the 40 ETH credited, only 10 ETH is backed by real ETH assets residing on the contract.

```solidity
_updateStableCollateralTotal(-_fundingFees)
_updateStableCollateralTotal(-(-40))
_updateStableCollateralTotal(+40))

stableCollateralTotal = stableCollateralTotal + _stableCollateralAdjustment
stableCollateralTotal = 50 + 40 = 90 ETH
```

At this point, assume that Alice decided to withdraw all his stake (25 UNIT) from the protocol. Since Alice owned 50% of the UNIT's total supply, she will receive back 50% of LP's `stableCollateralTotal` (90 ETH), which is equal to 45 ETH. After Alice has withdrawn from the protocol, the state is as follows:

- LP's `stableCollateralTotal` = 90 ETH - 45 ETH = 45 ETH
- Total ETH within the protocol = 60 ETH - 45 ETH = 15 ETH

There is an issue here: if Bob attempts to exit now, he will not be able to do so, as the protocol will attempt to transfer 45 ETH to Bob's wallet, but it will revert due to insufficient ETH on the contract. Apart from this, there is a more serious issue, as described below.

Assume a liquidator liquidates Charles's position. Line 146 within the `LiquidationModule.liquidate` function will be executed and evaluated as follows:

```solidity
vault.updateStableCollateralTotal(settledMargin - positionSummary.profitLoss);
vault.updateStableCollateralTotal(-30 ETH - 0);

stableCollateralTotal = stableCollateralTotal + _stableCollateralAdjustment
stableCollateralTotal = 45 ETH + (-30 ETH) = 15 ETH
```

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/LiquidationModule.sol#L146

```solidity
File: LiquidationModule.sol
084:     function liquidate(uint256 tokenId) public nonReentrant whenNotPaused liquidationInvariantChecks(vault, tokenId) {
..SNIP..
110:         // If the settled margin is greater than 0, send a portion (or all) of the margin to the liquidator and LPs.
111:         if (settledMargin > 0) {
..SNIP..
142:         } else {
143:             // If the settled margin is -ve then the LPs have to bear the cost.
144:             // Adjust the stable collateral total to account for user's negative remaining margin (includes PnL).
145:             // Note: The following is similar to giving the margin and the settled funding fees associated with the position to the LPs.
146:             vault.updateStableCollateralTotal(settledMargin - positionSummary.profitLoss);
147:         }
148: 
149:         // Update the global leverage position data.
150:         vault.updateGlobalPositionData({
151:             price: position.averagePrice,
152:             marginDelta: -(int256(position.marginDeposited) + positionSummary.accruedFunding),
153:             additionalSizeDelta: -int256(position.additionalSize) // Since position is being closed, additionalSizeDelta should be negative.
154:         });
```

After the liquidation, the state is as follows:

- LP's `stableCollateralTotal` ended at 15 ETH
- `globalPositions.marginDepositedTotal` = 0 ETH (-30 ETH + 30 ETH during the `vault.updateGlobalPositionData` at Line 150 above) - Correct, as there should be no more margin left after Charles is liquidated.

Bob decided to withdraw all his stakes (25 UNIT) from the protocol. Since he owned all of the UNIT's total supply, he will receive all the remaining LP's `stableCollateralTotal` of 15 ETH.

Notice that Bob invested 25 ETH but only received 15 ETH when the UNIT tokens actually gained or appreciated. If the accounting/math is correct in the first place, Bob should receive back 30 ETH (25 + 5) instead because he is entitled to half of the margin (10 ETH / 2) deposited by Charles that he forfeited. Bob loses 15 ETH in this scenario.

On the other hand, Alice received more assets than expected (15 ETH more).

## Impact

Loss of assets for the LPs as shown in the scenario above.

## Code Snippet

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L224

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/LiquidationModule.sol#L146

## Tool used

Manual Review

## Recommendation

In the above scenario, there is only 10 ETH of deposited margin left in the protocol. Thus, the maximum amount of funding fee that can be credited to the LP would be 10 ETH instead of 30 ETH. Beyond 10 ETH, the amount will not be backed by any ETH asset, leading to the issue mentioned in this report.

The root cause is that within the `settleFundingFees()` function, it proceeds to credit LP's `stableCollateralTotal` with 30 ETH instead of capping it at 10 ETH, leading to the LP's `stableCollateralTotal` being inflated and Alice being able to withdraw more assets than she should.

```solidity
File: FlatcoinVault.sol
224:     function settleFundingFees() public returns (int256 _fundingFees) {
..SNIP..
238:         // In the worst case scenario that the last position which remained open is underwater,
239:         // We set the margin deposited total to negative. Once the underwater position is liquidated,
240:         // then the funding fees will be reverted and the total will be positive again.
241:         _globalPositions.marginDepositedTotal = _globalPositions.marginDepositedTotal + _fundingFees;
242: 
243:         _updateStableCollateralTotal(-_fundingFees);
244:     }
```