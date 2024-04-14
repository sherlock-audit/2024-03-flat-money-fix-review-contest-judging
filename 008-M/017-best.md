Helpful Linen Wombat

high

# Long traders unable to withdraw their assets

## Summary

Long traders are unable to withdraw their assets when their settled margin is more than the ETH balance on the contract.

## Vulnerability Detail

While reviewing the fix for [Issue 196](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/196), a new edge case was discovered that could prevent the long traders from withdrawing their assets under the same condition.

Using back the same scenario in the original report:

Assume that:

- Bob's long position: Margin = 50 ETH
- Alice's LP: Deposited = 50 ETH
- Collateral Balance = 100 ETH (Actual balance of ETH on the contract)
- Tracked Balance = 100 ETH (Stable Collateral Total = 50 ETH, Margin Deposited Total = 50 ETH)

Assume that Bob's long position gains a profit of 51 ETH, and the total fee (trade + keeper fee) is zero to keep the math simple.

Bob wants to close his position, so the [`LeverageModule.executeClose`](https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/LeverageModule.sol#L301) function will be executed. The settled margin of Bob's long position will be as follows:

```solidity
settled margin = margin + PnL + accrued funding fee
settled margin = 50 ETH + 51 ETH + 0
settled margin = 101 ETH
```

At the end of the function, the following code will attempt to send 101 ETH (Bob's settled margin) to Bob's address. However, the problem is that the vault only has a total balance of 100 ETH. Due to insufficient ETH, the transfer will fail and revert. As a result, Bob is unable to close his position, and his assets are stuck in the contract.

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/LeverageModule.sol#L357

```solidity
vault.sendCollateral({to: _account, amount: uint256(settledMargin) - totalFee}); // transfer remaining amount to the trader
vault.sendCollateral({to: _account, amount: 101 ETH - 0});
vault.sendCollateral({to: _account, amount: 101 ETH);
```

## Impact

The users will lose assets as they cannot close their positions and withdraw their assets.

## Code Snippet

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/main/flatcoin-v1/src/LeverageModule.sol#L357

## Tool used

Manual Review

## Recommendation

Consider checking if there are sufficient assets in the vault before transferring them. In the above example, the code attempts to transfer 101 ETH to Bob while there are only 100 ETH in the vault, causing the transaction to fail and revert.

To remediate the issue, the maximum amount of assets that could be transferred to the users should capped at the total balance of the vault. In this case, only 100 ETH should be transferred to Bob when he closed his position. This will ensure that Bob's position will close successfully, and Bob will receive his profit without any revert.