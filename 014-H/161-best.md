Blurry Blush Mouse

High

# Attackers will steal the reserve from the `Vault` by receiving `ra` in `FlashSwapRouter::__swapDsforRa()`

### Summary

[FlashSwapRouter::__swapDsforRa()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L288) is called as part of [FlashSwapRouter::_swapRaforDs()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L98) [whenever](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L124) the reserve is sold and the resulting `Ra` is used to provide liquidity to the `Vault` by calling [Vault::provideLiquidityWithFlashSwapFee()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L125). 

However, if we follow the code, `FlashSwapRouter::__swapDsforRa()` calls [FlashSwapRouter::__flashSwap()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L301), which calls [UniswapV2Pair::swap()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L335). Then, the Uniswap pair calls [uniswapV2Call()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L338), which calls [FlashSwapRouter::__afterFlashswapSell()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L353) and sends the `Ra` to the [caller](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L404).

Thus, the vault is providing a fee, but does not use the acquired funds to fund it, but the already existing `Ra` in the contract. The `Ra` meant for the liquidity fee is sent to the user calling `FlashSwapRouter::_swapRaforDs()`.

### Root Cause

In `FlashSwapRouter.sol:124`, `FlashSwapRouter::__swapDsforRa()` is called, which ultimately sends the `Ra` to the `msg.sender`.

### Internal pre-conditions

None.

### External pre-conditions

None.

### Attack Path

1. User calls `swapRaforDs()` and ends up receiving `Ra` which was intended for the `Vault`.

### Impact

Users can steal all reserve from the `Vault`.

### PoC

The following code snippets show how the `Ra` ends up in the `caller`, that is, the user that calls `swapRaforDs()`.

```solidity
function __flashSwap(...) internal {
    ...
    bytes memory data = abi.encode(reserveId, dsId, buyDs, msg.sender, extraData);

    univ2Pair.swap(amount0out, amount1out, address(this), data);
}

function uniswapV2Call(address sender, uint256 amount0, uint256 amount1, bytes calldata data) external {
    (Id reserveId, uint256 dsId, bool buyDs, address caller, uint256 extraData) =
        abi.decode(data, (Id, uint256, bool, address, uint256));
    ...
    if (buyDs) {
        ...
    } else {
        uint256 amount = amount0 == 0 ? amount1 : amount0;

        __afterFlashswapSell(self, amount, reserveId, dsId, caller, extraData);
    }
}

function __afterFlashswapSell(...) internal {
    ...
    IERC20(ra).safeTransfer(caller, raAttributed);
    ...
}
```

### Mitigation

The `FlashSwapRouter::__flashSwap()` has to be modified to accept a caller argument, where it accepts either the `msg.sender` or `owner()`. In the flow of selling the reserve, it should send the `Ra` to the `owner`, which is the `Vault`.