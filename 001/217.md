Acrobatic Cider Cougar

High

# Incorrect calculations of `ra` and `ct` in the `MathHelper.calculateProvideLiquidityAmountBasedOnCtPrice()` function

## Summary

In the `VaultLib.__provideLiquidityWithRatio()` function, the amounts of `RA` and `CT` to be added to the Uniswap V2 pool are calculated using the `MathHelper.calculateProvideLiquidityAmountBasedOnCtPrice()` function. However, this approach is flawed. The `MathHelper.calculateProvideLiquidityAmountBasedOnCtPrice()` function requires that `ra + ct = amountra` (see line 34), which indicates that ct should represent an amount of RA rather than an amount of CT. This misunderstanding will lead to incorrect behavior in the subsequent action: `__provideLiquidity()`, where `ct` is intended to denote the amount of `CT` to be added to the Uniswap V2 pool.

## Vulnerability Detail

In the `VaultLib.__provideLiquidityWithRatio()` function, `amount` represents the total amount of `RA`, which should be divided into two parts. The first part is the amount of `RA` to be added to the Uniswap V2 pool, while the second part is to be exchanged for `CT`, which will also be added to the Uniswap V2 pool alongside the first part. In this function, `ra` corresponds to the first part, and `ct` represents the amount of `CT` exchanged from the second part. These values are calculated using the `MathHelper.calculateProvideLiquidityAmountBasedOnCtPrice()` function (line 134); however, this is incorrect. The `MathHelper.calculateProvideLiquidityAmountBasedOnCtPrice()` function requires that `ra + ct = amountra` (see line 34), implying that `ct` should represent an amount of `RA` corresponding to the second part, rather than the amount of `CT`. This misunderstanding will lead to incorrect behavior in the subsequent action: `__provideLiquidity()` (line 136), where `ct` is intended to denote the amount of `CT` to be added to the Uniswap V2 pool.

```solidity
VaultLib.sol

    function __provideLiquidityWithRatio(
        State storage self,
        uint256 amount,
        IDsFlashSwapCore flashSwapRouter,
        address ctAddress,
        IUniswapV2Router02 ammRouter
    ) internal returns (uint256 ra, uint256 ct) {
        uint256 dsId = self.globalAssetIdx;

        uint256 ctRatio = __getAmmCtPriceRatio(self, flashSwapRouter, dsId);

134     (ra, ct) = MathHelper.calculateProvideLiquidityAmountBasedOnCtPrice(amount, ctRatio);

136     __provideLiquidity(self, ra, ct, flashSwapRouter, ctAddress, ammRouter, dsId);
    }

------------------------

MathHelper.sol

    function calculateProvideLiquidityAmountBasedOnCtPrice(uint256 amountra, uint256 priceRatio)
        external
        pure
        returns (uint256 ra, uint256 ct)
    {
        ct = (amountra * 1e18) / (priceRatio + 1e18);
        ra = (amountra - ct);

34      assert((ct + ra) == amountra);
    }
```

Let's consider the following scenario:

1. `ctRatio = 1.1e18, amount = 210`.
2. `(ra, ct) = (110, 100)` (from the `MathHelper`).
3. `100 * (CT + DS)` is [minted](https://github.com/sherlock-audit/2024-08-cork-protocol/tree/main/Depeg-swap/contracts/libraries/VaultLib.sol#L167) during the call to the `__provideLiquidity()` function.

Finally, 110 `RA` and 100 `CT` are added to the Uniswap V2 pool. This indicates that 100 `RA` is exchanged for `100 * (CT + DS)` as if the exchange rate as `1:1`.

## Impact

Incorrectly treating the exchange between `RA` and `(CT + DS)` as `1:1`.

## Code Snippet

https://github.com/sherlock-audit/2024-08-cork-protocol/tree/main/Depeg-swap/contracts/libraries/VaultLib.sol#L123-L137

https://github.com/sherlock-audit/2024-08-cork-protocol/tree/main/Depeg-swap/contracts/libraries/MathHelper.sol#L26-L35

## Tool used

Manual Review

## Recommendation

Improve the `MathHelper.calculateProvideLiquidityAmountBasedOnCtPrice()` function.

```diff
    function calculateProvideLiquidityAmountBasedOnCtPrice(uint256 amountra, uint256 priceRatio)
        external
        pure
        returns (uint256 ra, uint256 ct)
    {
-       ct = (amountra * 1e18) / (priceRatio + 1e18);
-       ra = (amountra - ct);

-       assert((ct + ra) == amountra);

+       ct = (amountra / 2) * 1e18 / (priceRatio + 1e18);
+       ra = amountra - (amountra / 2);
    }
```