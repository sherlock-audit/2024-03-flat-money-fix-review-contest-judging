Bent Menthol Turtle

medium

# Attacker can steal LPs funds by using different oracle prices in the same transaction

## Summary

In the previous audit, there was an [issue](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/216) that pointed out that the PYTH oracle can return different prices within the same transaction. The fix implemented was to prevent the attack path described there, but a malicious LP can steal funds using a similar attack path. 

## Vulnerability Detail

The attack path is as follows:
1. The attacker has some liquidity deposited as an LP with address A.
2. The attacker announces the withdrawal of all liquidity with address A.
3. In the same txn, the attacker announces the deposit of liquidity (same amount as the withdrawal) with address B.
4. After the minimum execution time has elapsed, retrieve two prices from the Pyth oracle where the second price is lower than the first one.
5. Execute the withdrawal of address A sending the higher price.
6. Execute the deposit of address B sending the lower price. 

As a result, the attacker will have the same deposited liquidity in the protocol, plus a profit from the price deviation. 

Even though the withdrawal may reduce the profits of this attack, it doesn't completely prevent it. Moreover, if the attacker is the only LP on the market, there's not going to be a withdrawal fee, increasing the profits of such an attack at the expense of the traders. 

The requirements of this attack are the same as the ones described in the [original issue](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/216), hence the medium severity. 

**Sidenote:**
Now that I'm checking better the fixes on the previous audit, I think that the second attack path described by the [original issue](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/216) is still not fixed:

> Another possible strategy would pass through the following steps:
>- Create a leverage position.
>- Announce another leverage position with the same size.
>- In the same block, announce a limit close order.
>- After the minimum execution time has elapsed, retrieve two prices from the Pyth oracle where the second price is lower than the first one.
>- Execute the limit close order sending the first price.
>- Execute the open order sending the second price.
>
>The result in this case is having a position with the same size as the original one, but having either lowered the position.lastPrice or getting a profit from the original position, depending on how the price has moved since the original position was opened.

I don't have access to the [exact pull request of the fix](https://github.com/dhedge/flatcoin-v1/pull/276) but I'm pretty sure the attack path described above is still feasible. 

## Impact

Different oracle prices can be fetched in the same transaction, which can be used to create arbitrage opportunities that can make a profit with no risk at the expense of users on the other side of the trade.

## Code Snippet

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/OracleModule.sol#L63

## Tool used

Manual Review

## Recommendation

Given the time constraints of this contest, I haven't been able to think about a quick fix for this issue. Even if the protocol team doesn't allow to update the price feeds within the same block, an attacker can still update them by directly calling the PYTH contract. 
