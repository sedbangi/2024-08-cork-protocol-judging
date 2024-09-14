Jumpy Lime Oyster

High

# Using getReserves() Instead Of getReservesSorted() Will Give Incorrect RA Back To User

## Summary

While redeeming early the LP is liquidated partially and then depending on the reserves in the AMM the RA amount received is calculated which is paid out to the user for his LV tokens. But when fetching the reserves from the AMM the reserves are not sorted (as they are everywhere else in the code) which leads to user receiving incorrect (more or less depending on reserve ratio) amount of RA tokens.


## Vulnerability Detail

1.) User calls for a early redeem ->

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L639

2.) This function calls `_liquidateLpPartial()` which calls -> `__calculateCtBalanceWithRate()` and this function is used to fetch the `raPerLv, ctPerLv, raPerLp, ctPerLp`  ->

```solidity
function __calculateCtBalanceWithRate(State storage self, IDsFlashSwapCore flashSwapRouter, uint256 dsId)
        internal
        view
        returns (uint256 raPerLv, uint256 ctPerLv, uint256 raPerLp, uint256 ctPerLp)
    {
        (uint256 raReserve, uint256 ctReserve,) = flashSwapRouter.getUniV2pair(self.info.toId(), dsId).getReserves();

        (,, raPerLv, ctPerLv, raPerLp, ctPerLp) = __calculateTotalRaAndCtBalanceWithReserve(
            self, raReserve, ctReserve, flashSwapRouter.getLvReserve(self.info.toId(), dsId)
        );
    }
```

But we can see to fetch the reserves from the AMM `getReserves()` has been used instead of `getReservesSorted()` therefore it is possible that `raReserve` is actually the ctReserve and vice versa. Meaning , incorrect amount of reserves would be passed in the `__calculateTotalRaAndCtBalanceWithReserve` function to fetch `(,, raPerLv, ctPerLv, raPerLp, ctPerLp)`

3.) Therefore , the `(,, raPerLv, ctPerLv, raPerLp, ctPerLp) ` would be calculated incorrectly , and the LP would also be calculated incorrectly here 

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L312

4.) This means the RA amount received by the user by the partial liquidation would be incorrect and incorrect amount of liquidity would be removed from the AMM pool  if the tokens are not in the intended order(dependent on addresses).

## Impact

This means the RA amount received by the user by the partial liquidation would be incorrect and incorrect amount of liquidity would be removed from the AMM pool.

## Code Snippet

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L443

## Tool used

Manual Review

## Recommendation

Use getReservesSorted() instead