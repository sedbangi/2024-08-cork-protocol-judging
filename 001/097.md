Proper Turquoise Goldfish

High

# Lack of Slippage Protection for Reserve during swaps

### Summary

The `swapRaforDs()` function allows users to swap Redemption Asset (RA) for Depeg Swap (DS) tokens:

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L98-L138
`swapRaforDs()` -> `_swapRaforDs()`:

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
            uint256 vaultRa = __swapDsforRa(assetPair, reserveId, dsId, amountSellFromReserve, 0); //@audit - no slippage
            IVault(owner()).provideLiquidityWithFlashSwapFee(reserveId, vaultRa);

  
            // recalculate the amount of DS tokens attributed, since we sold some from the reserve
            (amountOut, borrowedAmount,) = assetPair.getAmountOutBuyDS(amount);
        }

        if (amountOut < amountOutMin) {
            revert InsufficientOutputAmount();
        }

        // trigger flash swaps and send the attributed DS tokens to the user

        __flashSwap(assetPair, assetPair.pair, borrowedAmount, 0, dsId, reserveId, true, amountOut);
    }
```

If a user wants to have slippage protection, he can set it as a parameter, and at the end of the function, we have a check.

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L124

```solidity
        if (amountOut < amountOutMin) {
            revert InsufficientOutputAmount();
        }
```

However, when the protocol executes the swap for its **reserve**, there is no slippage protection in place, exposing the protocol to unfavorable market conditions. Specifically, the reserve sells **DS** tokens without ensuring a minimum acceptable amount of **RA** in return, potentially resulting in significant financial losses for the protocol.

```solidity
uint256 vaultRa = __swapDsforRa(assetPair, reserveId, dsId, amountSellFromReserve, 0);
```

In this line, the reserve executes the swap without any slippage protection (the slippage is set to `0`), meaning the reserve will accept any amount of **RA** in return for the **DS** tokens it sells. This makes the protocol vulnerable to market volatility or manipulation.

There are three variants here:

1. A regular user sets a slippage for him, but in the swap protocol loses tokens because there is no slippage protection for the reserve. 
2. A regular user decides he doesn't need slippage protection and leaves it at 0. The protocol takes a huge loss because of this user decision.
3. Self-sandwich Attack. I will explain in more detail in the Attack Path. 

### Root Cause

The lack of slippage protection when the reserve sells its **DS** tokens in the `_swapRaforDs()` function:

```solidity
uint256 vaultRa = __swapDsforRa(assetPair, reserveId, dsId, amountSellFromReserve, 0);
```



### Internal pre-conditions

Мalicious or normal users need to set `amountOutMin` to 0. In the Attack Path section, I have explained both variants. 
- If a normal user calls `swapRaforDs()` with slippage protection, the loss to the protocol is minimal - Low Severity.
- If a normal user calls `swapRaforDs()` without slippage protection, the loss to the protocol is significant - High Severity.
- Self-sandwich Attack - High Severity

### External pre-conditions

No external pre-conditions

### Attack Path

#### Normal Attack Path
- - A user calls the `swapRaforDs()` function to swap RA for DS tokens and does not want slippage protection for **himself**.
- The function internally calls the `_swapRaforDs()` function.
- Inside `_swapRaforDs()`, the protocol attempts to sell DS tokens from the reserve using the following code:
```solidity
uint256 vaultRa = __swapDsforRa(assetPair, reserveId, dsId, amountSellFromReserve, 0);
```
- The **reserve** is selling DS tokens with **0 slippage protection**.
- The **reserve** could suffer significant losses
***
#### Self-sandwich Attack Path

In this scenario, the attacker uses two accounts (let's call them **Account A** and **Account B**) to manipulate the protocol’s lack of slippage protection for personal profit.

- The attacker controls two accounts: **Account A** and **Account B**.
- **Account A** will make a trade that shifts the market price. 
- **Account B** will execute the swap using `swapRaforDs()` with **zero slippage** protection to exploit the price difference caused by **Account A**.
- **Account A** places a transaction that manipulates the price of **DS** tokens, for example, by buying a large amount of **DS** or swapping another token to increase **DS** price or decrease liquidity. This transaction front-runs **Account B’s** swap, ensuring that when **Account B**'s transaction executes, it does so at a manipulated and unfavorable price.
- Since there is no slippage protection, the protocol will complete the trade at a much worse price than it should have, causing the reserve to lose value.
- After **Account B** has executed its swap and the protocol has lost value, **Account A** back-runs the transaction by reversing the initial trade or taking advantage of the new price levels.
- **Account A** sells **DS** tokens at the inflated price, securing a profit.

### Impact

The **protocol** suffers losses when the **reserve** sells **DS** tokens without any slippage protection. In case of market volatility or manipulation, the protocol will receive less **RA** in return than the actual value of the **DS** tokens being sold.

### PoC

_No response_

### Mitigation

Introduce Slippage Protection for the Reserve. If a user decides they don't need slippage protection, then you can set a standard base percentage to protect the protocol.