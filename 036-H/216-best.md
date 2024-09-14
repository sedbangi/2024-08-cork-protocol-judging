Acrobatic Cider Cougar

High

# Improper resetting of `reservedDs` in the `VaultLib._redeemCtDsAndSellExcessCt()` function

## Summary

In the `VaultLib._redeemCtDsAndSellExcessCt()` function, the calculation of `ctSellAmount` is incorrect because it uses the reset `reservedDs` instead of the original value.

## Vulnerability Detail

In the `_redeemCtDsAndSellExcessCt()` function, `reservedDs` represents the amount of `DS`, while `ammCtBalance` indicates the amount of `CT`. The process begins by redeeming as much `RA` as possible using the total of `CT + DS`, and any remaining `CT` is subsequently sold. However, the remaining amount of `CT` is miscalculated because the calculation relies on a reset `reservedDs`.

Consider the following scenario:

1. `reservedDs = 100, ammCtBalance = 70`.
2. `redeemAmount = 70` (see line 327).
3. `reservedDs` is reset to 30 at line 329.
4. `ctSellAmount = 70 - 30 = 40` (see line 334).

In this scenario, `ctSellAmount` is incorrectly calculated. At step 2, since `redeemAmount` is 70, it implies that `70 * (CT + DS)` is used to redeem `RA`, leaving the remaining `CT` amount at 0. Therefore, `ctSellAmount` should actually be 0.

The issue arises from step 3, where `reservedDs` is reset at line 329.

Because `ctSellAmount` is calculated to be greater than the actual remaining `CT` amount, the transaction will revert at line 345 due to insufficient `CT` to sell. This will disrupt the core functionality `redeemEarly`, as it [invokes the `_liquidateLpPartial()` function](https://github.com/sherlock-audit/2024-08-cork-protocol/tree/main/Depeg-swap/contracts/libraries/VaultLib.sol#L656), which in turn [calls the `_redeemCtDsAndSellExcessCt()` function](https://github.com/sherlock-audit/2024-08-cork-protocol/tree/main/Depeg-swap/contracts/libraries/VaultLib.sol#L315).

```solidity
    function _redeemCtDsAndSellExcessCt(
        State storage self,
        uint256 dsId,
        IUniswapV2Router02 ammRouter,
        IDsFlashSwapCore flashSwapRouter,
        uint256 ammCtBalance
    ) internal returns (uint256 ra) {
325     uint256 reservedDs = flashSwapRouter.getLvReserve(self.info.toId(), dsId);

327     uint256 redeemAmount = reservedDs >= ammCtBalance ? ammCtBalance : reservedDs;

329     reservedDs = flashSwapRouter.emptyReservePartial(self.info.toId(), dsId, redeemAmount);

        ra += redeemAmount;
        PsmLibrary.lvRedeemRaWithCtDs(self, redeemAmount, dsId);

334     uint256 ctSellAmount = reservedDs >= ammCtBalance ? 0 : ammCtBalance - reservedDs;

        DepegSwap storage ds = self.ds[dsId];
        address[] memory path = new address[](2);
        path[0] = ds.ct;
        path[1] = self.info.pair1;

        ERC20(ds.ct).approve(address(ammRouter), ctSellAmount);

        if (ctSellAmount != 0) {
            // 100% tolerance, to ensure this not fail
345         ra += ammRouter.swapExactTokensForTokens(ctSellAmount, 0, path, address(this), block.timestamp)[1];
        }
    }
```

## Impact

Disruption of `redeemEarly`, which is a core functionality of the protocol.

## Code Snippet

https://github.com/sherlock-audit/2024-08-cork-protocol/tree/main/Depeg-swap/contracts/libraries/VaultLib.sol#L318-L347

## Tool used

Manual Review

## Recommendation

Don't reset the value of `reservedDs`.

```diff
    function _redeemCtDsAndSellExcessCt(
        State storage self,
        uint256 dsId,
        IUniswapV2Router02 ammRouter,
        IDsFlashSwapCore flashSwapRouter,
        uint256 ammCtBalance
    ) internal returns (uint256 ra) {
        uint256 reservedDs = flashSwapRouter.getLvReserve(self.info.toId(), dsId);

        uint256 redeemAmount = reservedDs >= ammCtBalance ? ammCtBalance : reservedDs;

-       reservedDs = flashSwapRouter.emptyReservePartial(self.info.toId(), dsId, redeemAmount);
+       flashSwapRouter.emptyReservePartial(self.info.toId(), dsId, redeemAmount);

        ra += redeemAmount;
        PsmLibrary.lvRedeemRaWithCtDs(self, redeemAmount, dsId);

        uint256 ctSellAmount = reservedDs >= ammCtBalance ? 0 : ammCtBalance - reservedDs;

        DepegSwap storage ds = self.ds[dsId];
        address[] memory path = new address[](2);
        path[0] = ds.ct;
        path[1] = self.info.pair1;

        ERC20(ds.ct).approve(address(ammRouter), ctSellAmount);

        if (ctSellAmount != 0) {
            // 100% tolerance, to ensure this not fail
            ra += ammRouter.swapExactTokensForTokens(ctSellAmount, 0, path, address(this), block.timestamp)[1];
        }
    }
```