Glorious Brick Weasel

High

# Incorrect Calculation in `getAmountOutDs` Function Leads to Errors in DS Token Swaps

## Summary
The `getAmountOutDs` function in the `DsSwapperMathLib.sol` contract, which calculates the amount of DS tokens a user receives (`s`), has a discrepancy between the documented formula and the actual implementation. This error cascades, affecting the `getAmountOutBuyDS` function in the `DsFlashSwap.sol` contract and the `_swapRaforDs` function in the `FlashSwapRouter.sol` contract, resulting in incorrect DS token swaps.

## Vulnerability Detail
According to the contract comments, the formula to calculate the amount of DS tokens received (`s`) should be `S = (E + x - y + q) / 2`. However, the actual implementation uses `S = (E - (x - y) + q) / 2`, based on the equation `r2 = q - r1`.
[2024-08-cork-protocol/Depeg-swap/contracts/libraries/DsSwapperMathLib.sol:getAmountOutDs_L124-L135](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/DsSwapperMathLib.sol#L124C1-L135C32)
```solidity
   function getAmountOutDs(int256 x, int256 y, int256 e) external pure returns (uint256 r, uint256 s) {
        // first we solve the sqrt part of the equation first

        // E^2
        int256 q1 = e ** 2;
        // 2E(x + y)
        int256 q2 = 2 * e * (x + y);
        // (x - y)^2
        int256 q3 = (x - y) ** 2;

        // q = sqrt(E^2 + 2E(x + y) + (x - y)^2)
        uint256 q = SignedMath.abs(q1 + q2 + q3);
        q = Math.sqrt(q);

        // then we substitue back the sqrt part to the main equation
125 @audit=>        // S = (E + x - y + q) / 2

        // r1 = x - y (to absolute, and we reverse the equation)
        uint256 r1 = SignedMath.abs(x - y);
        // r2 = -r1 + q  = q - r1
130 @audit=>        uint256 r2 = q - r1;
        // E + r2
        uint256 r3 = r2 + SignedMath.abs(e);

        // S = r3/2 (we multiply by 1e18 to have 18 decimals precision)
135 @audit=>        s = (r3 * 1e18) / 2e18;

        // R = s - e (should be fine with direct typecasting)
        r = s - SignedMath.abs(e);
    }
}

```

Due to this incorrect calculation, the `getAmountOutBuyDS` function in the `DsFlashSwap.sol` contract cannot correctly determine the amount of RA and CT assets a user needs to borrow and repay when buying a specified amount of DS assets. This, in turn, impacts the `_swapRaforDs` function in the `FlashSwapRouter.sol` contract, which fails to transfer the correct amount of DS tokens to users during swaps based on the RA assets borrowed.
[2024-08-cork-protocol/Depeg-swap/contracts/libraries/DsFlashSwap.sol:getAmountOutBuyDS_L187](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/DsFlashSwap.sol#L187)
```solidity
    function getAmountOutBuyDS(AssetPair storage assetPair, uint256 amount)
        internal
        view
        returns (uint256 amountOut, uint256 borrowedAmount, uint256 repaymentAmount)
    {
        (uint112 raReserve, uint112 ctReserve) = getReservesSorted(assetPair);

        (borrowedAmount, amountOut) =
187 @aucit=>             SwapperMathLibrary.getAmountOutDs(int256(uint256(raReserve)), int256(uint256(ctReserve)), int256(amount));

        repaymentAmount = MinimalUniswapV2Library.getAmountIn(borrowedAmount, ctReserve, raReserve - borrowedAmount);
    }

    function isRAsupportsPermit(address token) internal view returns (bool) {
        return PermitChecker.supportsPermit(token);
    }
}
```

[2024-08-cork-protocol/Depeg-swap/contracts/libraries/
/FlashSwapRouter.sol:_swapRaforDs_L128](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L128C1-L128C80)
```solidity

    function _swapRaforDs(
        ReserveState storage self,
        AssetPair storage assetPair,
        Id reserveId,
        uint256 dsId,
        uint256 amount,
        uint256 amountOutMin
    ) internal returns (uint256 amountOut) {
        uint256 borrowedAmount;

        // calculate the amount of DS tokens attributed
        (amountOut, borrowedAmount,) = assetPair.getAmountOutBuyDS(amount);

        // calculate the amount of DS tokens that will be sold from LV reserve
        uint256 amountSellFromReserve =
            amountOut - MathHelper.calculatePrecentageFee(self.reserveSellPressurePrecentage, amountOut);

        // sell all tokens if the sell amount is higher than the available reserve
        amountSellFromReserve = assetPair.reserve < amountSellFromReserve ? assetPair.reserve : amountSellFromReserve;

        // sell the DS tokens from the reserve if there's any
        if (amountSellFromReserve != 0) {
            // decrement reserve
            assetPair.reserve -= amountSellFromReserve;

            // sell the DS tokens from the reserve and accrue value to LV holders
            uint256 vaultRa = __swapDsforRa(assetPair, reserveId, dsId, amountSellFromReserve, 0);
            IVault(owner()).provideLiquidityWithFlashSwapFee(reserveId, vaultRa);

            // recalculate the amount of DS tokens attributed, since we sold some from the reserve
128 @audit=>            (amountOut, borrowedAmount,) = assetPair.getAmountOutBuyDS(amount);
        }

        if (amountOut < amountOutMin) {
            revert InsufficientOutputAmount();
        }
```



## Impact
This calculation error creates a significant problem in the swap mechanism, leading to incorrect DS token amounts being transferred to users. This could potentially result in users receiving more or fewer DS tokens than expected, undermining trust in the platform and creating financial imbalances.

## Code Snippet
[2024-08-cork-protocol/Depeg-swap/contracts/libraries/DsSwapperMathLib.sol:getAmountOutDs_L124-L135](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/DsSwapperMathLib.sol#L124C1-L135C32)
[2024-08-cork-protocol/Depeg-swap/contracts/libraries/DsFlashSwap.sol:getAmountOutBuyDS_L187](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/DsFlashSwap.sol#L187)
[2024-08-cork-protocol/Depeg-swap/contracts/libraries/
/FlashSwapRouter.sol:_swapRaforDs_L128](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L128C1-L128C80)

## Tool used

Manual Review

## Recommendation
Modify the `getAmountOutDs` function in the `DsSwapperMathLib.sol` contract to align with the documented formula. This correction will ensure that the calculation of the amount of DS tokens received is accurate, fixing the issues in the swap process and preventing further miscalculations in downstream contracts like `DsFlashSwap.sol` and `FlashSwapRouter.sol`.