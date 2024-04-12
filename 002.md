Wobbly Myrtle Mantaray

high

# The transfer lock for leveraged position orders can still be easily bypassed

## Summary

The transfer lock for leveraged position orders can be bypassed

## Vulnerability Detail

This report is highly inspired by the previously submitted report [here](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues?q=is%3Aissue+is%3Aopen+The+transfer+lock+for+leveraged+position+orders+can+be+bypassed) and it's duplicates, namely:[1](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues?q=is%3Aissue+Race+Condition%3A+Shared+write+access+to+isLocked+allows+NFTs+with+open+announcements+to+become+tradeable.+is%3Aclosed) & [2](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues?q=is%3Aissue+LimitOrder.cancelLimitOrder+can+be+used+to+unlock+position+with+LeverageClose+order+is%3Aclosed), considering this audit is a fix review, all `Will Fix` issue should be fixed, however that's not the case here as the locks can still be bypassed, this is cause going through the currently in-scope [DelayedOrder.announceLeverageClose()](https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/539341fdc92090c48792c454e35e5d739ee62820/flatcoin-v1/src/DelayedOrder.sol#L321-L373) or [LimitOrder.announceLimitOrder()](https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/539341fdc92090c48792c454e35e5d739ee62820/flatcoin-v1/src/LimitOrder.sol#L53-L83) does not prevent announcing order if the leveraged position is already locked.

From here: https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/539341fdc92090c48792c454e35e5d739ee62820/flatcoin-v1/src/DelayedOrder.sol#L627 we can see that the ` leverageModule.executeAdjust()` is still being queried with the `account:account` instead of getting the owner this way `account: leverageModule.ownerOf(leverageClose.tokenId),`, the same case for how the `leverageModule.executeClose()` is being called too while closing, as can be seen https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/539341fdc92090c48792c454e35e5d739ee62820/flatcoin-v1/src/DelayedOrder.sol#L627, this still been the case easily allows the bug report explained in one of the duplicates, i.e [here](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues?q=is%3Aissue+LimitOrder.cancelLimitOrder+can+be+used+to+unlock+position+with+LeverageClose+order+is%3Aclosed) to hold since one can use the `cancelLimitOrder()` to unlock a position with LeverageClose order.

## Impact

Bypass of the transferability of a locked token, this then leads to other quite more impactful cases, like the attacker selling the leveraged position with a close order opened, execute the order afterward, and steal the underlying collateral.

Note that where as there are multiple duplicates to the primary issue from the previous contest, they all explain the bug window via a quite distinct window, showcasing different impacts, and as such kindly refer to [this](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/48#issue-2117166455) and it's duplicates .

## Code Snippet

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/539341fdc92090c48792c454e35e5d739ee62820/flatcoin-v1/src/DelayedOrder.sol#L321-L373

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/539341fdc92090c48792c454e35e5d739ee62820/flatcoin-v1/src/DelayedOrder.sol#L321-L373

https://github.com/sherlock-audit/2024-03-flat-money-fix-review-contest/blob/539341fdc92090c48792c454e35e5d739ee62820/flatcoin-v1/src/LimitOrder.sol#L53-L83

## Tool used

Manual Review

- Previous Audit Reports: [here](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues?q=is%3Aissue+is%3Aopen+The+transfer+lock+for+leveraged+position+orders+can+be+bypassed) and it's duplicates, namely:[1](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues?q=is%3Aissue+Race+Condition%3A+Shared+write+access+to+isLocked+allows+NFTs+with+open+announcements+to+become+tradeable.+is%3Aclosed) & [2](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues?q=is%3Aissue+LimitOrder.cancelLimitOrder+can+be+used+to+unlock+position+with+LeverageClose+order+is%3Aclose).

## Recommendation

Do not announce the order either through DelayedOrder.announceLeverageClose or LimitOrder.announceLimitOrder if the leveraged position is already locked, and also have the real owner passed as the account, i.e queried via `account: leverageModule.ownerOf(leverageClose.tokenId),` when calling executeAdjust/executeClose().
