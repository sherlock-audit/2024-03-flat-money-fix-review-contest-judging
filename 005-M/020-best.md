Helpful Linen Wombat

medium

# More points were minted to the long trader side

## Summary

More points were minted to the long trader side than expected, creating unfairness to the LP within the protocol.

## Vulnerability Detail

When opening a leverage position, the amount of points minted to the long traders is based on the position size.

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/LeverageModule.sol#L137

```solidity
File: LeverageModule.sol
080:     function executeOpen(
..SNIP..
135:         // Mint points
136:         IPointsModule pointsModule = IPointsModule(vault.moduleAddress(FlatcoinModuleKeys._POINTS_MODULE_KEY));
137:         pointsModule.mintLeverageOpen(_account, announcedOpen.additionalSize);
```

When depositing assets to the protocol, the amount of points minted to the LP (Short side) is based on the deposited amount.

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/StableModule.sol#L81

```solidity
File: StableModule.sol
58:     function executeDeposit(
..SNIP..
79:         // Mint points
80:         IPointsModule pointsModule = IPointsModule(vault.moduleAddress(FlatcoinModuleKeys._POINTS_MODULE_KEY));
81:         pointsModule.mintDeposit(_account, _announcedDeposit.depositAmount);
```

Assume that the `pointsPerSize` is 10 points per ETH within the point module, and the max leverage is set to 25x within the protocol.

Alice, who is a LP (short-side), deposited 10 ETH and received 100 points. Bob, who is the long trader, deposited a margin of 10 ETH and opened a position of 250 ETH (25x leverage), and he received 2500 points, which is much higher than Alice's.

Both Alice and Bob expose the same amount of ETH in the protocol, and the directional movement of the ETH price will affect both sides (LP and long traders) in a roughly similar manner. The risk exposed by the long trader and LP (short) side is also roughly the same. Otherwise, the economic model of the protocol is broken as it favors one side more than the other, and no one would be incentivized to be on the disadvantaged side.

Assuming a perfectly balanced market with 50% short/LP and 50% long, the gain of long traders will lead to a loss to LP (appreciation of the ETH price that LP is entitled to will be nullified as they need to be pay as profit to the long side). On the other hand, the loss of long traders will lead to a gain to LP (long traders lose their margin to LP).

Thus, the points minted for the long traders should be based on the deposited margin instead of the position size to reflect better the risk taken by both sides.

## Impact

More points were minted to the long trader side than expected, creating unfairness to the LP within the protocol. In the main audit contest, points are deemed items that contain value as issues related to points are classified as Medium.

## Code Snippet

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/LeverageModule.sol#L137

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/StableModule.sol#L81

## Tool used

Manual Review

## Recommendation

It is recommended that points should be minted for the long traders based on the deposited margin instead of the position size to reflect better the risk taken by both sides. 

Alternatively, a better solution would be to determine the number of points to long traders more sophisticatedly by multiplying the deposited margin by a risk factor parameter if the risk incurred by the long traders is not exactly the same as the LPs.
