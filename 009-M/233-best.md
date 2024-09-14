Amateur Jade Trout

Medium

# DOS risk for too strict slippage protection.

## Summary

There is a slippage protection that calculates asset tolerance amounts to prevent the user from getting less tokens than they wanted due to high volatility. However the result of the calculate of these asset tolerance amounts are few below than expected amount, this create a risk of DOS during periods of high volatility.

## Vulnerability Detail

Here is where the calculation of the asset tolerance amounts starts to define the minimum amounts of tokens desired when user deposit/add liquidity:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L64
And here is the formula to get those values:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/MathHelper.sol#L152

Let's make the calculation to get the `raTolerance` value assuming the `ra` is 1e18.

`raTolerance = ra - ((ra * 1e18 * tolerance) / (100 * 1e18) / 1e18);`

Here the tolerance value is hardcoded, been equal to 1e18 as you can see here:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/MathHelper.sol#L16

```js
Step 1: Calculate ra x 1e18 x tolerance
  ra x 1e18 x tolerance = 1e18 x 1e18 x 1e18 = 1e54
Step 2: Divide by 100 x 1e18
  1e54 / (100 x 1e18) = 1e54 / 1e20 = 1e34
Step 3: Divide  by 1e18
  1e34 / 1e18 = 1e16
Step 4: Subtract from ra
  raTolerance = 1e18 - 1e16 = 0.99e18
```
In this scenario 0.99e18 is the 99% of the desired `ra` amount, which is too strict. This strictness could lead to DOS scenarios due to high market volatility.
  
## Impact
User won't be able to deposit/add liquidity on those scenarios of high volatility.

## Tool used

Manual Review

## Recommendation

These calculated tolerance values can be  used a default slippage, but consider users should been able to override it with their own slippage to ensure they can transact even during times of high volatility.