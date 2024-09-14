Acrobatic Cider Cougar

Medium

# `DoS` to the `FlashSwapRouter.__afterFlashswapBuy()` function

## Summary

In the `FlashSwapRouter.__afterFlashswapBuy()` function, it exchanges `RA` for `CT + DS` and sends `CT` to the Uniswap V2 pool. However, the sent amount might be greater than the generated amount, which could lead to the reversal of the transaction.

## Vulnerability Detail

As you can see in the `__afterFlashswapBuy()` function:

1. It exchanges `RA` for `CT + DS` (line 368).
2. It sends `CT` to the `msg.sender` (line 377).

The amount of `RA` to be exchanged in step 1 is `dsAttributed`, so the amount of generated `CT + DS` is `dsAttributed / exchangeRate`. If the exchange rate is greater than 1, the amount of generated `CT` will be smaller than `dsAttributed`, resulting in the reversal of sending `CT` in step 2 due to insufficient `CT`.

Since the `__afterFlashswapBuy()` function is invoked when buying `DS`, this could lead to a `DoS` when buying `DS`.

```solidity
    function __afterFlashswapBuy(
        ReserveState storage self,
        Id reserveId,
        uint256 dsId,
        address caller,
        uint256 dsAttributed
    ) internal {
        AssetPair storage assetPair = self.ds[dsId];
        assetPair.ra.approve(owner(), dsAttributed);

        IPSMcore psm = IPSMcore(owner());
368     psm.depositPsm(reserveId, dsAttributed);

        // should be the same, we don't compare with the RA amount since we maybe dealing
        // with a non-rebasing token, in which case the amount deposited and the amount received will always be different
        // so we simply enforce that the amount received is equal to the amount attributed to the user

        // send caller their DS
        assetPair.ds.transfer(caller, dsAttributed);
        // repay flash loan
377     assetPair.ct.transfer(msg.sender, dsAttributed);
    }
```

## Impact

Buying `DS` could be `DoS`ed.

## Code Snippet

https://github.com/sherlock-audit/2024-08-cork-protocol/tree/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L357-L378

## Tool used

Manual Review

## Recommendation

Sending amount should be fixed as follows.

```diff
    function __afterFlashswapBuy(
        ReserveState storage self,
        Id reserveId,
        uint256 dsId,
        address caller,
        uint256 dsAttributed
    ) internal {
        AssetPair storage assetPair = self.ds[dsId];
        assetPair.ra.approve(owner(), dsAttributed);

        IPSMcore psm = IPSMcore(owner());
        psm.depositPsm(reserveId, dsAttributed);

        // should be the same, we don't compare with the RA amount since we maybe dealing
        // with a non-rebasing token, in which case the amount deposited and the amount received will always be different
        // so we simply enforce that the amount received is equal to the amount attributed to the user

        // send caller their DS
        assetPair.ds.transfer(caller, dsAttributed);
        // repay flash loan
-       assetPair.ct.transfer(msg.sender, dsAttributed);
+       assetPair.ct.transfer(msg.sender, dsAttributed * 1e18 / assetPair.ds.exchangeRate());
    }
```