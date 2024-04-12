Bent Menthol Turtle

medium

# Large amount of points can STILL be minted without any cost

## Summary

An issue was raised in the last FlatMoney audit ([here](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/187)) where the watsons pointed out that the points could be minted without any cost, this issue still remains, and now the attacker can prevent other users from earning points. 

## Vulnerability Detail

The issue raised in the last audit pointed out that an attacker could perform the following attack to mint a large number of points:
1. Deposit liquidity
2. After a short amount of time, withdraw it
3. Repeat the attack, minting a huge amount of points

This attack can still be executed to mint a large number of points at almost no cost. The mitigation that was implemented (if I'm not mistaken), consisted of a rate limit on the minted points. Even though this mitigation would reduce the profit of an attacker, it won't prevent the attacker from minting the maximum amount of points until the rate limit is reached. 

Moreover, an attacker could use the rate limit to prevent other users from minting points. By constantly depositing and withdrawing liquidity, it would mint all points for himself until the rate limit is reached. Then, when other innocent users deposit liquidity or make trades, they won't receive any points.

## Impact

The attacker can still mint a large amount of points, and prevent other users from receiving them.

## Code Snippet

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/StableModule.sol#L93

## Tool used

Manual Review

## Recommendation

Reduce the amount of points earned when users withdraw liquidity or implement a withdrawal fee even if the LP is the last in the market. 


