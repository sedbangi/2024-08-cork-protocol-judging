Narrow Iron Zebra

High

# Incorrect Token Amount Passed to `unsafeIssueToLv` in `VaultLib::__provideLiquidity` Function


### Summary

The `__provideLiquidity` function in the VaultLibrary incorrectly handles the token amounts when locking and minting tokens. Specifically, the function passes the `ctAmount` (Cover Token amount) to `PsmLibrary.unsafeIssueToLv()` instead of `raAmount` (Redemption Asset amount). This can lead to incorrect RA token locking and CT:DS minting.


### Vulnerability Detail

The `__provideLiquidityWithRatio` function calculates `raAmount` and `ctAmount` based on the `ctRatio`, which may not always be a 1:1 ratio. This means that `raAmount` and `ctAmount` could differ.So in the `__provideLiquidity` function, passing `ctAmount` to the `PsmLibrary.unsafeIssueToLv()` function results in an incorrect amount of RA being locked and CT:DS minted, leading to discrepancies in the liquidity vault's token accounting and PSM.


```solidity
    function __provideLiquidityWithRatio(
        State storage self,
        uint256 amount,
        IDsFlashSwapCore flashSwapRouter,
        address ctAddress,
        IUniswapV2Router02 ammRouter
    ) internal returns (uint256 ra, uint256 ct) {
        uint256 dsId = self.globalAssetIdx;

        uint256 ctRatio = __getAmmCtPriceRatio(self, flashSwapRouter, dsId);

        (ra, ct) = MathHelper.calculateProvideLiquidityAmountBasedOnCtPrice(amount, ctRatio);

 @>>       __provideLiquidity(self, ra, ct, flashSwapRouter, ctAddress, ammRouter, dsId);
    }


    function __provideLiquidity(
        State storage self,
        uint256 raAmount,
        uint256 ctAmount,
        IDsFlashSwapCore flashSwapRouter,
        address ctAddress,
        IUniswapV2Router02 ammRouter,
        uint256 dsId
    ) internal {
        // no need to provide liquidity if the amount is 0
        if (raAmount == 0 && ctAmount == 0) {
            return;
        }

@>>        PsmLibrary.unsafeIssueToLv(self, ctAmount);

        __addLiquidityToAmmUnchecked(self, raAmount, ctAmount, self.info.redemptionAsset(), ctAddress, ammRouter);

        _addFlashSwapReserve(self, flashSwapRouter, self.ds[dsId], ctAmount);
    }
```

### Impact

The PSM internal accounting might be inaccurate, as it doesn't reflect the full amount of RA that has been deposited.


### Code Snippet

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L167

### Tool Used

Manual Review

### Recommendation

Replace `ctAmount` with `raAmount` in the call to `PsmLibrary.unsafeIssueToLv()` to ensure the correct handling of the Redemption Asset.

```diff

    function __provideLiquidity(
        State storage self,
        uint256 raAmount,
        uint256 ctAmount,
        IDsFlashSwapCore flashSwapRouter,
        address ctAddress,
        IUniswapV2Router02 ammRouter,
        uint256 dsId
    ) internal {
        // no need to provide liquidity if the amount is 0
        if (raAmount == 0 && ctAmount == 0) {
            return;
        }

-        PsmLibrary.unsafeIssueToLv(self, ctAmount);
+        PsmLibrary.unsafeIssueToLv(self, raAmount);

        __addLiquidityToAmmUnchecked(self, raAmount, ctAmount, self.info.redemptionAsset(), ctAddress, ammRouter);

        _addFlashSwapReserve(self, flashSwapRouter, self.ds[dsId], ctAmount);
    }
```

