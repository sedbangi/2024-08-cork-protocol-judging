Glorious Brick Weasel

High

# Incorrect Price Ratio Calculation in DsSwapperMathLib.sol Leads to Mispricing in DS and RA Asset Swaps

## Summary

`getPriceRatioUniv2` function calculates the price ratio using the `encode` and `uqdiv` functions from the `UQ112x112` contract, i.e., ct / ra for raPriceRatio and ra / ct for ctPriceRatio. However, this causes the `calculateDsPrice` function to return an incorrect DS price, leading to errors in the `getAmountIn` and `getAmountOut` functions, which fail to properly compute the RA required to buy DS or the RA obtained from selling DS. However, the `getPriceRatio` and `tryGetPriceRatioAfterSellDs` functions in `DsFlashSwap.sol` calculate RA and CT price ratios using `getPriceRatioUniv2` function, affecting the `getCurrentPriceRatio` and `previewSwapRaforDs` functions in the `FlashSwapRouter.sol` contract, preventing accurate swap outcomes for RA and DS assets,  if `getPriceRatioUniv2` function is revised for the `calculateDsPrice` function.

## Vulnerability Detail

The calculateDsPrice function of the DsSwapperMathLib.sol contract calculates ds price = (ds + ct) / ra - ct/ ra = ds / ra. And the `calculateDsPrice` function will call the `getPriceRatioUniv2` function of this contract to calculate ctPriceRatio.
[Depeg-swap/contracts/libraries/DsSwapperMathLib.sol:calculateDsPrice_L56](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/DsSwapperMathLib.sol#L56)

```solidity
                function calculateDsPrice(uint112 raReserve, uint112 ctReserve, uint256 dsExchangeRate)
                    public
                    pure
                    returns (uint256 price)
                {
56 @audit=>        (, uint256 ctPriceRatio) = getPriceRatioUniv2(raReserve, ctReserve);

                    price = dsExchangeRate - ctPriceRatio;
                }

```

The problem is that when calculating `ctPriceRatio`, the `getPriceRatioUniv2` function will call the encode and uqdiv functions of the UQ112x112 contract, that is, `ctPriceRatioUQ = encodedRaReserve.uqdiv(ctReserve)`. Therefore the calculated `ctPriceRatioUQ` can be regarded as equal to ra / ct, not ct / ra and the ds price is incorrectly calculated to (ds + ct) / ra - ra/ ct, instead of (ds + ct) / ra - ct/ ra.
[Depeg-swap/contracts/libraries/DsSwapperMathLib.sol:getPriceRatioUniv2_L43](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/DsSwapperMathLib.sol#L43)

```solidity
                // Calculate price ratio of two tokens in a uniswap v2 pair, will return ratio on 18 decimals precision
                function getPriceRatioUniv2(uint112 raReserve, uint112 ctReserve)
                    public
                    pure
                    returns (uint256 raPriceRatio, uint256 ctPriceRatio)
                {
                    if (raReserve <= 0 || ctReserve <= 0) {
                        revert ZeroReserve();
                    }

                    // Encode uni v2 reserves to UQ112x112 format
                    uint224 encodedRaReserve = UQ112x112.encode(raReserve);
                    uint224 encodedCtReserve = UQ112x112.encode(ctReserve);

                    // Calculate price ratios using uqdiv
                    uint224 raPriceRatioUQ = encodedCtReserve.uqdiv(raReserve);
43 @audit=>         uint224 ctPriceRatioUQ = encodedRaReserve.uqdiv(ctReserve);

                    // Convert UQ112x112 to regular uint (divide by 2**112)
                    // we time by 18 to have 18 decimals precision
                    raPriceRatio = (uint256(raPriceRatioUQ) * 1e18) / UQ112x112.Q112;
                    ctPriceRatio = (uint256(ctPriceRatioUQ) * 1e18) / UQ112x112.Q112;
                }
```

This will cause the `calculateDsPrice` function to fail to calculate the correct ds price, which in turn causes the `getAmountIn` function and `getAmountOut` function of this contract to fail to correctly calculate the number of RA assets required to buy a specified number of DS assets, and fail to correctly calculate the number of RA assets that can be obtained by selling a specified number of DS assets.
[Depeg-swap/contracts/libraries/DsSwapperMathLib.sol:getAmountIn_L75](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/DsSwapperMathLib.sol#L75)

```solidity
                function getAmountIn(
                    uint256 amountOut, // Amount of DS tokens to buy
                    uint112 raReserve, // Reserve of the input token
                    uint112 ctReserve, // Reserve of the other token (needed for price ratio calculation)
                    uint256 dsExchangeRate // DS exchange rate
                ) external pure returns (uint256 amountIn) {
                    if (amountOut == 0) {
                        revert InsufficientOutputAmount();
                    }

                    if (raReserve == 0 || ctReserve == 0) {
                        revert InsufficientLiquidity();
                    }

75 @audit=>         uint256 dsPrice = calculateDsPrice(raReserve, ctReserve, dsExchangeRate);

                    amountIn = (amountOut * dsPrice) / 1e18;
                }
```

[Depeg-swap/contracts/libraries/DsSwapperMathLib.sol:getAmountOut_L94](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/DsSwapperMathLib.sol#L94)

```solidity
                function getAmountOut(
                    uint256 amountIn, // Amount of input tokens
                    uint112 reserveIn, // Reserve of the input token
                    uint112 reserveOut, // Reserve of the other token (needed for price ratio calculation)
                    uint256 dsExchangeRate // DS exchange rate
                ) external pure returns (uint256 amountOut) {
                    if (amountIn == 0) {
                        revert InsufficientInputAmount();
                    }

                    if (reserveIn == 0 || reserveOut == 0) {
                        revert InsufficientLiquidity();
                    }

94 @audit=>         uint256 dsPrice = calculateDsPrice(reserveIn, reserveOut, dsExchangeRate);

                    amountOut = amountIn / dsPrice;
                }

```

Additionally, the `getPriceRatio` function and `tryGetPriceRatioAfterSellDs` function of the DsFlashSwap.sol contract will call the `getPriceRatioUniv2` function to calculate the price ratio of RA and CT, as well as the price ratio of RA and CT after selling DS assets.
[Depeg-swap/contracts/libraries/DsFlashSwap.sol:getPriceRatio_L96](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/DsFlashSwap.sol#L96)

```solidity
                function getPriceRatio(ReserveState storage self, uint256 dsId)
                    internal
                    view
                    returns (uint256 raPriceRatio, uint256 ctPriceRatio)
                {
                    AssetPair storage asset = self.ds[dsId];

                    address token0 = asset.pair.token0();
                    address token1 = asset.pair.token1();

                    (uint112 token0Reserve, uint112 token1Reserve,) = self.ds[dsId].pair.getReserves();

                    (uint112 raReserve, uint112 ctReserve) = MinimalUniswapV2Library.reverseSortWithAmount112(
                        token0, token1, address(asset.ra), address(asset.ct), token0Reserve, token1Reserve
                    );

96 @audit=>         (raPriceRatio, ctPriceRatio) = SwapperMathLibrary.getPriceRatioUniv2(raReserve, ctReserve);
                }
```

[Depeg-swap/contracts/libraries/DsFlashSwap.sol:tryGetPriceRatioAfterSellDs_L119](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/DsFlashSwap.sol#L119)

```solidity
                function tryGetPriceRatioAfterSellDs(
                    ReserveState storage self,
                    uint256 dsId,
                    uint256 ctSubstracted,
                    uint256 raAdded
                ) internal view returns (uint256 raPriceRatio, uint256 ctPriceRatio) {
                    AssetPair storage asset = self.ds[dsId];

                    address token0 = asset.pair.token0();
                    address token1 = asset.pair.token1();

                    (uint112 token0Reserve, uint112 token1Reserve,) = self.ds[dsId].pair.getReserves();

                    (uint112 raReserve, uint112 ctReserve) = MinimalUniswapV2Library.reverseSortWithAmount112(
                        token0, token1, address(asset.ra), address(asset.ct), token0Reserve, token1Reserve
                    );

119 @audit=>        raReserve += uint112(raAdded);
                    ctReserve -= uint112(ctSubstracted);

                    (raPriceRatio, ctPriceRatio) = SwapperMathLibrary.getPriceRatioUniv2(raReserve, ctReserve);
                }
```

This will cause the `getCurrentPriceRatio` function of the FlashSwapRouter.sol contract to obtain the desired price ratio of RA and CT when calling the `getPriceRatio` function of the DsFlashSwap.sol contract. And the `previewSwapRaforDs` function of the FlashSwapRouter.sol contract calculate the correct amount of DS that will be received from swapping RA after calling the `tryGetPriceRatioAfterSellDs` function of the DsFlashSwap.sol contract, together with the `ct = (amountra * 1e18) / (priceRatio + 1e18)` from the `calculateProvideLiquidityAmountBasedOnCtPrice` in the MathHelper.sol contract.
[Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol:getCurrentPriceRatio_L90](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L90)

```solidity
                function getCurrentPriceRatio(Id id, uint256 dsId)
                    external
                    view
                    override
                    returns (uint256 raPriceRatio, uint256 ctPriceRatio)
                {
90 @audit=>         (raPriceRatio, ctPriceRatio) = reserves[id].getPriceRatio(dsId);
                }

```

[Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol:previewSwapRaforDs_L225](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L225)

```solidity
                function previewSwapRaforDs(Id reserveId, uint256 dsId, uint256 amount) external view returns (uint256 amountOut) {
                    ReserveState storage self = reserves[reserveId];
                    AssetPair storage assetPair = self.ds[dsId];

                    uint256 borrowedAmount;
                    (amountOut, borrowedAmount,) = assetPair.getAmountOutBuyDS(amount);

                    // calculate the amount of DS tokens that will be sold from LV reserve
                    uint256 amountSellFromReserve =
                        amountOut - MathHelper.calculatePrecentageFee(self.reserveSellPressurePrecentage, amountOut);

                    // sell all tokens if the sell amount is higher than the available reserve
                    amountSellFromReserve = assetPair.reserve < amountSellFromReserve ? assetPair.reserve : amountSellFromReserve;

                    // sell the DS tokens from the reserve if there's any
                    if (amountSellFromReserve != 0) {
                        (uint112 raReserve, uint112 ctReserve) = assetPair.getReservesSorted();

                        // we borrow the same amount of CT tokens from the reserve
                        ctReserve -= uint112(amountSellFromReserve);

                        (uint256 vaultRa, uint256 raAdded) = assetPair.getAmountOutSellDS(amountSellFromReserve);
                        raReserve += uint112(raAdded);

                        // emulate Vault way of adding liquidity using RA from selling DS reserve
225 @audit=>            (, uint256 ratio) = self.tryGetPriceRatioAfterSellDs(dsId, amountSellFromReserve, raAdded);
                        uint256 ctAdded;
                        (raAdded, ctAdded) = MathHelper.calculateProvideLiquidityAmountBasedOnCtPrice(vaultRa, ratio);

                        raReserve += uint112(raAdded);
                        ctReserve += uint112(ctAdded);

                        // update amountOut since we sold some from the reserve
                        (, amountOut) = SwapperMathLibrary.getAmountOutDs(
                            int256(uint256(raReserve)), int256(uint256(ctReserve)), int256(amount)
                        );
                    }
                }

```

[Depeg-swap/contracts/libraries/MathHelper.sol:calculateProvideLiquidityAmountBasedOnCtPrice_L31](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/MathHelper.sol#L31)

```solidity

                function calculateProvideLiquidityAmountBasedOnCtPrice(uint256 amountra, uint256 priceRatio)
                    external
                    pure
                    returns (uint256 ra, uint256 ct)
                {
31 @audit=>         ct = (amountra * 1e18) / (priceRatio + 1e18);
                    ra = (amountra - ct);

                    assert((ct + ra) == amountra);
                }

```

Therefore, simply revising the ratio calculation in the `getPriceRatioUniv2` function will not fully resolve the issue. The `getCurrentPriceRatio` and `previewSwapRaforDs` functions in the `FlashSwapRouter.sol` contract would receive incorrect RA/CT price ratios, leading to further inaccuracies in their calculations, if `getPriceRatioUniv2` function is revised for the `calculateDsPrice` function.

## Impact

This miscalculation has a cascading effect:

- The `calculateDsPrice` function fails to correctly calculate the DS price.
- As a result, the `getAmountIn` and `getAmountOut` functions in the contract cannot accurately compute the amount of RA assets required to buy a specified amount of DS, nor the amount of RA assets obtained from selling DS.
- The `getPriceRatio` and `tryGetPriceRatioAfterSellDs` functions in the `DsFlashSwap.sol` contract also rely on `getPriceRatioUniv2`. Therefore, the `FlashSwapRouter.sol` contract's `getCurrentPriceRatio` and `previewSwapRaforDs` functions will not receive the correct RA/CT price ratio, leading to inaccurate estimations in swaps between RA and DS, if the `getPriceRatioUniv2` function is revised for the `calculateDsPrice` function.

## Code Snippet

[Depeg-swap/contracts/libraries/DsSwapperMathLib.sol:calculateDsPrice_L56](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/DsSwapperMathLib.sol#L56)
[Depeg-swap/contracts/libraries/DsSwapperMathLib.sol:getPriceRatioUniv2_L43](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/DsSwapperMathLib.sol#L43)
[Depeg-swap/contracts/libraries/DsSwapperMathLib.sol:getAmountIn_L75](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/DsSwapperMathLib.sol#L75)
[Depeg-swap/contracts/libraries/DsSwapperMathLib.sol:getAmountOut_L94](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/DsSwapperMathLib.sol#L94)
[Depeg-swap/contracts/libraries/DsFlashSwap.sol:getPriceRatio_L96](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/DsFlashSwap.sol#L96)
[Depeg-swap/contracts/libraries/DsFlashSwap.sol:tryGetPriceRatioAfterSellDs_L119](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/DsFlashSwap.sol#L119)
[Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol:getCurrentPriceRatio_L90](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L90)
[Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol:previewSwapRaforDs_L225](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L225)

## Tool used

Manual Review

## Recommendation

It is recommended to modify the calculation of the price ratio in the `calculateDsPrice` function so that ds price = (ds + ct) / ra - ct/ ra = ds / ra.

This will ensure that the `calculateDsPrice` function, and subsequently the `getAmountIn`, `getAmountOut`, `getCurrentPriceRatio`, and `previewSwapRaforDs` functions, can correctly compute the DS price and ensure proper RA/DS swaps.