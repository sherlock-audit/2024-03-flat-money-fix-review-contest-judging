Bent Menthol Turtle

high

# Funding fee will be inaccurate when leveraged positions are not closed

## Summary

With the new changes in the codebase, the global PnL isn't settled until the individual positions are closed or liquidated. This change will cause an inaccurate funding fee when there are some open positions with pending PnL.  

## Vulnerability Detail

The function that calculates the current funding fee is the following:

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L451-L474
```solidity
   function _getUnrecordedFunding()
        private
        view
        returns (int256 fundingChangeSinceRecomputed, int256 unrecordedFunding)
    {
        int256 proportionalSkew = PerpMath._proportionalSkew({
>>        skew: int256(_globalPositions.sizeOpenedTotal) - int256(stableCollateralTotal),
>>        stableCollateralTotal: stableCollateralTotal
        });

        fundingChangeSinceRecomputed = PerpMath._fundingChangeSinceRecomputed({
            proportionalSkew: proportionalSkew,
            prevFundingModTimestamp: lastRecomputedFundingTimestamp,
            maxFundingVelocity: maxFundingVelocity,
            maxVelocitySkew: maxVelocitySkew
        });

        unrecordedFunding = PerpMath._unrecordedFunding({
            currentFundingRate: fundingChangeSinceRecomputed + lastRecomputedFundingRate,
            prevFundingRate: lastRecomputedFundingRate,
            prevFundingModTimestamp: lastRecomputedFundingTimestamp
        });
    }
}
```

The first thing the function does is to calculate the proportional skew, and it uses the total size of the traders (`sizeOpenedTotal`) and the total stable collateral deposited (`stableCollateralTotal`). The issue is that with the new changes, the global PnL isn't settled when settling funding fees, but it's done when every individual position on the market is closed. 

This means that when there are some open positions on the market with pending profits or losses, the value of `stableCollateralTotal` won't be accurate because it won't take into account the unrealized PnL. This will cause the resulting funding fee will be wrong, and that will imbalance the market, causing a huge mess on the FlatMoney protocol.

The docs define the funding rate as the following:

> Borrow Rates (a.k.a., funding rates): To ensure that the price of a perpetual contract stays close to the underlying asset's spot price, funding rates are used. In the Flat Money documentation, we refer to this as the borrow rate. Traders who hold positions that go against the current discrepancy will receive payments, while those on the other side pay. This mechanism encourages traders to close or open positions, which in turn helps align the perpetual contract's price with the spot price.

When the funding is wrongly calculated, the market won't be in equilibrium, and that will undermine the whole "delta-neutral" narrative of the protocol. When UNIT holders (LPs) see that their positions are not delta-neutral as the protocol claims, they'll withdraw their liquidity, causing a loss for the overall protocol.  

## Impact

When there are open positions with pending PnL, the funding fee will be inaccurate. Given that the funding rate is the core mechanism that the protocol has to balance the LPs and the traders, an inaccurate funding fee will cause an imbalance in the protocol, breaking a core invariant and causing a greatly skewed market. 

## Code Snippet

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L456-L459

## Tool used

Manual Review

## Recommendation

It's recommended to take into account the global PnL of the leveraged positions when calculating for the funding rate. 
