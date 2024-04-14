Helpful Linen Wombat

high

# Code asymmetry of `globalPositions.marginDepositedTotal`

## Summary

Code asymmetry of `globalPositions.marginDepositedTotal` state variable could lead to an accounting error within the protocol, leading to the protocol being broken. It will also lead to a loss of assets for the LP.

## Vulnerability Detail

There is a code asymmetry issue in how the `globalPositions.marginDepositedTotal` is calculated within the `FlatcoinVault.settleFundingFees` and `FlatcoinVault.updateGlobalPositionData` functions.

Within the ``FlatcoinVault.settleFundingFees`` function, the `globalPositions.marginDepositedTotal` is allowed to go to negative. The reason for allowing this is mentioned in the code's comment `Once the underwater position is liquidated, then the funding fees will be reverted and the total will be positive again.` in Line 238 of the ``FlatcoinVault.settleFundingFees`` function. During the liquidation of the underwater position, the liquidation logic will re-adjust the negative `globalPositions.marginDepositedTotal` to be positive again.

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L224

```solidity
File: FlatcoinVault.sol
224:     function settleFundingFees() public returns (int256 _fundingFees) {
..SNIP..
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

However, within the `FlatcoinVault.updateGlobalPositionData` functions, it operates differently where the `globalPositions.marginDepositedTotal` cannot become negative. If the value is negative, it will be set to zero, as seen in Line 191 of the `FlatcoinVault.updateGlobalPositionData` functions above.

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L175

```solidity
File: FlatcoinVault.sol
175:     function updateGlobalPositionData(
176:         uint256 _price,
177:         int256 _marginDelta,
178:         int256 _additionalSizeDelta
179:     ) external onlyAuthorizedModule {
180:         // Note that technically, even the funding fees should be accounted for when computing the margin deposited total.
181:         // However, since the funding fees are settled at the same time as the global position data is updated,
182:         // we can ignore the funding fees here.
183:         int256 newMarginDepositedTotal = _globalPositions.marginDepositedTotal + _marginDelta;
184: 
185:         // Check that the sum of margin of all the leverage traders is not negative.
186:         // Rounding errors shouldn't result in a negative margin deposited total given that
187:         // we are rounding down the profit loss of the position.
188:         // The margin may be negative if liquidations are not happening in a timely manner.
189:         // In such a case, the system should continue to function as normal.
190:         if (newMarginDepositedTotal < 0) {
191:             newMarginDepositedTotal = 0;
192:         }
```

The misalignment of how the `globalPositions.marginDepositedTotal` should behave within different functions causes various issues within the protocol.

Consider the following scenario to demonstrate one of the negative impacts.

- Alice (LP) deposits 25 ETH and mints 25 UNIT
- LP's `stableCollateralTotal` = 25 ETH
- Bob (Trader) deposits 10 ETH as a margin and opens a long position with the size of 100 ETH (10x leverage)
- `globalPositions.marginDepositedTotal` = 10 ETH (Consists of margin of Bob position)
- Total ETH within the protocol = 35 ETH (25 + 10)

Assume the accrued funding fee is 20 ETH, and the long traders must pay the LP (short). Note that these values are intentionally chosen or inflated to demonstrate the accounting/math error in the implementation, and they do not affect the validity of this issue.

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

Assume that the `FlatcoinVault.settleFundingFees` function is executed at this point. Line 241 above will be evaluated as follows. The `globalPositions.marginDepositedTotal` will be updated to -10 ETH.

```solidity
_globalPositions.marginDepositedTotal = _globalPositions.marginDepositedTotal + _fundingFees;
_globalPositions.marginDepositedTotal = 10 ETH + (-20 ETH) = -10 ETH
```

In the comments in Lines 238-240 above within the `FlatcoinVault.settleFundingFees` function, it expects the liquidation process to re-adjust back the `globalPositions.marginDepositedTotal` to positive or the correct value later.

Assume that a liquidation is executed. The `vault.updateGlobalPositionData` function at Line 150 within the  `LiquidationModule.liquidate` function below will be executed, and the value of `marginDelta` parameter will be as follows:

```solidity
marginDelta = -(int256(position.marginDeposited) + positionSummary.accruedFunding)
marginDelta = -(10 ETH + (-20 ETH))
marginDelta = -(-10 ETH)
marginDelta = +10 ETH
```
https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/LiquidationModule.sol#L150
```solidity
File: LiquidationModule.sol
074:     function liquidate(
..SNIP..
149:         // Update the global leverage position data.
150:         vault.updateGlobalPositionData({
151:             price: position.averagePrice,
152:             marginDelta: -(int256(position.marginDeposited) + positionSummary.accruedFunding),
153:             additionalSizeDelta: -int256(position.additionalSize) // Since position is being closed, additionalSizeDelta should be negative.
154:         });
```

Within the `updateGlobalPositionData` function below, Line 183 will evaluate as follows:

```solidity
newMarginDepositedTotal = _globalPositions.marginDepositedTotal + _marginDelta;
newMarginDepositedTotal = -10 + (+10) = 0
```

This is aligned with the earlier code's comments (`Once the underwater position is liquidated, then the funding fees will be reverted and the total will be positive again.`) as the `globalPositions.marginDepositedTotal` indeed is restored back to the positive and correct value after the re-adjustment.

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L175

```solidity
File: FlatcoinVault.sol
175:     function updateGlobalPositionData(
176:         uint256 _price,
177:         int256 _marginDelta,
178:         int256 _additionalSizeDelta
179:     ) external onlyAuthorizedModule {
180:         // Note that technically, even the funding fees should be accounted for when computing the margin deposited total.
181:         // However, since the funding fees are settled at the same time as the global position data is updated,
182:         // we can ignore the funding fees here.
183:         int256 newMarginDepositedTotal = _globalPositions.marginDepositedTotal + _marginDelta;
```

Unfortunately, in reality, the execution of the `settleFundingFees` function is not always followed by the execution of the liquidation logic. Thus, the logic code expect the liquidation process executed later to always re-adjust the `globalPositions.marginDepositedTotal` value back to the intended value.

Let's assume another scenario. Assume that after the `FlatcoinVault.settleFundingFees` function is executed, someone triggers an action (e.g., open, adjust, close order) that executes the `updateGlobalPositionData` function instead of the liquidation. Assume that the `_marginDelta` is insignificant in the following:

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L175

```solidity
File: FlatcoinVault.sol
175:     function updateGlobalPositionData(
176:         uint256 _price,
177:         int256 _marginDelta,
178:         int256 _additionalSizeDelta
179:     ) external onlyAuthorizedModule {
180:         // Note that technically, even the funding fees should be accounted for when computing the margin deposited total.
181:         // However, since the funding fees are settled at the same time as the global position data is updated,
182:         // we can ignore the funding fees here.
183:         int256 newMarginDepositedTotal = _globalPositions.marginDepositedTotal + _marginDelta;
..SNIP..
190:         if (newMarginDepositedTotal < 0) {
191:             newMarginDepositedTotal = 0;
192:         }
```

When Line 190 above is executed, since ``globalPositions.marginDepositedTotal`` is negative (-10 ETH), it will be reset to zero. This is an issue because the code earlier assumes that the liquidation process will help to re-adjust the negative value. However, over here, the state has been wiped out.

When the liquidation is executed subsequently, the `marginDelta` will be `+10 ETH`, similar to the earlier scenario.

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/LiquidationModule.sol#L150

```solidity
File: LiquidationModule.sol
074:     function liquidate(
..SNIP..
149:         // Update the global leverage position data.
150:         vault.updateGlobalPositionData({
151:             price: position.averagePrice,
152:             marginDelta: -(int256(position.marginDeposited) + positionSummary.accruedFunding),
153:             additionalSizeDelta: -int256(position.additionalSize) // Since position is being closed, additionalSizeDelta should be negative.
154:         });
```

Within the `updateGlobalPositionData` function, the `globalPositions.marginDepositedTotal` will be set to as follows:

```solidity
newMarginDepositedTotal = _globalPositions.marginDepositedTotal + _marginDelta
newMarginDepositedTotal = 0 + (+10 ETH)
newMarginDepositedTotal = 10 ETH
```
https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L175
```solidity
File: FlatcoinVault.sol
175:     function updateGlobalPositionData(
176:         uint256 _price,
177:         int256 _marginDelta,
178:         int256 _additionalSizeDelta
179:     ) external onlyAuthorizedModule {
180:         // Note that technically, even the funding fees should be accounted for when computing the margin deposited total.
181:         // However, since the funding fees are settled at the same time as the global position data is updated,
182:         // we can ignore the funding fees here.
183:         int256 newMarginDepositedTotal = _globalPositions.marginDepositedTotal + _marginDelta;
```

At the end of the liquidation, the long trader side, which had lost all its margin earlier, regained 10 ETH, which is incorrect and shows an error in the protocol's accounting. 

The `globalPositions.marginDepositedTotal` is also inflated and not backed by any assets. The system will wrongly assume that there are `globalPositions.marginDepositedTotal`, while in reality, there is none. This results in the system continuing to transfer collateral assets to the LP when settling funding fees when there are none. The LP will receive collateral credit backed by no assets, leading to a loss of assets for them.

## Impact

The `marginDepositedTotal` state is the key variable in the protocol's account system, and its value will be incorrect, leading to the protocol being broken. It will also lead to a loss of assets for the LP.

## Code Snippet

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L224

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L175

## Tool used

Manual Review

## Recommendation

Ensure that the behavior of `globalPositions.marginDepositedTotal` (whether it can go negative) is consistent throughout the codebase to prevent any error from occurring.