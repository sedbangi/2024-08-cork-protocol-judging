Acrobatic Cider Cougar

High

# The `VaultLib.__liquidateUnchecked()` function unnecessarily reorders the already correctly ordered values of `raReceived` and `ctReceived` from `UniswapV2Router`

## Summary

The `__liquidateUnchecked()` function removes liquidity and receives the corresponding `RA` and `CT` from UniswapV2, returning the amounts of `RA` and `CT` received. When removing liquidity, the `removeLiquidity()` function of the `UniswapV2Router` returns `raReceived` and `ctReceived`, which are the exact amounts of `RA` and `CT` corresponding to the removed liquidity, regardless of the order of tokens `RA` and `CT` in the Uniswap pool. However, in the `__liquidateUnchecked()` function, these values are reordered as if they represent the received amounts of `token0` and `token1`, assuming that `token0 = CT` and `token1 = RA`.

## Vulnerability Detail

As noted in line 282 of the `__liquidateUnchecked()` function, the `ammRouter.removeLiquidity()` function returns two values: `raReceived` and `ctReceived`. These values represent the exact amounts of `RA` and `CT` received from the Uniswap pool, even in the case where `token0 = CT` and `token1 = RA` (see `UniswapV2Router02.sol`). Therefore, there is no need to reorder these values. However, at line 284, the function reorders them, assuming `token0 = CT` and `token1 = RA`. As a result, when `token0 = CT` and `token1 = RA`, the values of `raReceived` and `ctReceived` will be interchanged. This leads to incorrect behavior in the [_liquidatedLp()](https://github.com/sherlock-audit/2024-08-cork-protocol/tree/main/Depeg-swap/contracts/libraries/VaultLib.sol#L370) function, which processes expired states when [issuing new DS](https://github.com/sherlock-audit/2024-08-cork-protocol/tree/main/Depeg-swap/contracts/libraries/VaultLib.sol#L104). Consequently, an incorrect `RA` amount is reserved in the [reserve pool](https://github.com/sherlock-audit/2024-08-cork-protocol/tree/main/Depeg-swap/contracts/libraries/VaultLib.sol#L392), potentially leading to a loss of `RA` for users.

```solidity
VaultLib.sol

    function __liquidateUnchecked(
        State storage self,
        address raAddress,
        address ctAddress,
        IUniswapV2Router02 ammRouter,
        IUniswapV2Pair ammPair,
        uint256 lp
    ) internal returns (uint256 raReceived, uint256 ctReceived) {
        ammPair.approve(address(ammRouter), lp);

        // amountAMin & amountBMin = 0 for 100% tolerence
        (raReceived, ctReceived) =
282         ammRouter.removeLiquidity(raAddress, ctAddress, lp, 0, 0, address(this), block.timestamp);

284     (raReceived, ctReceived) = MinimalUniswapV2Library.reverseSortWithAmount224(
            ammPair.token0(), ammPair.token1(), raAddress, ctAddress, raReceived, ctReceived
        );

        self.vault.config.lpBalance -= lp;
    }

----------------------------

UniswapV2Router02.sol

    function removeLiquidity(
        address tokenA,
        address tokenB,
        uint liquidity,
        uint amountAMin,
        uint amountBMin,
        address to,
        uint deadline
    ) public virtual override ensure(deadline) returns (uint amountA, uint amountB) {
        address pair = UniswapV2Library.pairFor(factory, tokenA, tokenB);
        IUniswapV2Pair(pair).transferFrom(msg.sender, pair, liquidity); // send liquidity to pair
        (uint amount0, uint amount1) = IUniswapV2Pair(pair).burn(to);
        (address token0,) = UniswapV2Library.sortTokens(tokenA, tokenB);
@>      (amountA, amountB) = tokenA == token0 ? (amount0, amount1) : (amount1, amount0);
        require(amountA >= amountAMin, 'UniswapV2Router: INSUFFICIENT_A_AMOUNT');
        require(amountB >= amountBMin, 'UniswapV2Router: INSUFFICIENT_B_AMOUNT');
    }
```

## Impact

Loss of `RA` for users.

## Code Snippet

https://github.com/sherlock-audit/2024-08-cork-protocol/tree/main/Depeg-swap/contracts/libraries/VaultLib.sol#L270-L289

https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router02.sol#L116

## Tool used

Manual Review

## Recommendation

Eliminate the reordering.

```diff
    function __liquidateUnchecked(
        State storage self,
        address raAddress,
        address ctAddress,
        IUniswapV2Router02 ammRouter,
        IUniswapV2Pair ammPair,
        uint256 lp
    ) internal returns (uint256 raReceived, uint256 ctReceived) {
        ammPair.approve(address(ammRouter), lp);

        // amountAMin & amountBMin = 0 for 100% tolerence
        (raReceived, ctReceived) =
            ammRouter.removeLiquidity(raAddress, ctAddress, lp, 0, 0, address(this), block.timestamp);

-       (raReceived, ctReceived) = MinimalUniswapV2Library.reverseSortWithAmount224(
-           ammPair.token0(), ammPair.token1(), raAddress, ctAddress, raReceived, ctReceived
-       );

        self.vault.config.lpBalance -= lp;
    }
```