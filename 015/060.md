Funny Fossilized Shetland

High

# Incorrect calculation of amounts used in swaps in MinimalUniswapV2Library

## Summary
MinimalUniswapV2Library - `getAmountOut` and `getAmountIn` return wrong amounts due to flaw in calculation which leads to wrong amount used in swap.
## Vulnerability Detail
Here is the correct AMM related maths, (assuming no fees) that UniswapV2Library uses and that `MinimalUniswapV2Library` failed to mimic.

let $ReserveIn = Ri, ReserveOut = Ro, amountIn = ai, AmountOut = ao$

the basic AMM eqn,

$$Ri \times Ro = K$$

now when swapping, the `ReserveIn` will increase by `amountIn` and the `ReserveOut` will decrease by `amountOut`, K remains constant.

$$(RI + aI)(Ro - ao) = K$$

$$(Ri + ai)(Ro - ao) = Ri \times Ro$$

Rearranging that eqn to get `ao` and `ai` 

$$ao = \frac{ai \times Ro}{Ri + ai}$$

multiplying by 1000 to get the actual eqn to be used in the code,

$$ao = \frac{ai \times 1000 \times Ro}{1000Ri + 1000ai}$$

and for ai,

$$ai = \frac{Ri \times ao \times 1000}{(Ro - ao)\\times 1000}$$

Looking at the functions, the denominators are miscalculated as seen below,
```solidity
function getAmountOut(uint256 amountIn, uint256 reserveIn, uint256 reserveOut)
 // ...
        uint256 amountInWithFee = amountIn * NO_FEE;
        uint256 numerator = amountInWithFee * reserveOut;
        uint256 denominator = reserveIn * 1000; // @audit WRONG
        amountOut = numerator / denominator;
 ```
```solidity
function getAmountIn(uint256 amountOut, uint256 reserveIn, uint256 reserveOut)
// ...
        uint256 numerator = reserveIn * amountOut * 1000;
        uint256 denominator = reserveOut * NO_FEE; // @audit WRONG
        amountIn = (numerator / denominator);
```
## Impact
The user always gets incorrect amount, for instance in `VaultLib::previewRedeemEarly` the `received` is miscalculated due to this.
## Code Snippet
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/uni-v2/UniswapV2Library.sol#L62

Here is the UniswapV2 library original implementation (the correct one)
https://github.com/Uniswap/v2-periphery/blob/0335e8f7e1bd1e8d8329fd300aea2ef2f36dd19f/contracts/libraries/UniswapV2Library.sol#L43
## Tool used
Manual Review

## Recommendation
```diff
function getAmountOut(uint256 amountIn, uint256 reserveIn, uint256 reserveOut)
 // ...
-        uint256 denominator = reserveIn * 1000;
+       uint256 denominator = reserveIn * 1000 + amountInWithFee;
 ```
```diff
function getAmountIn(uint256 amountOut, uint256 reserveIn, uint256 reserveOut)
// ...
-        uint256 denominator = reserveOut * NO_FEE; 
+       uint256 denominator = (reserveOut - amountOut) * NO_FEE;
```