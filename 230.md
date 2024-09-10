Acrobatic Cider Cougar

High

# When the price of `PA` declines, users can profit by executing `depositPsm + redeemRaWithDs`

## Summary

When the price of `PA` drops significantly, users can profit by following these steps:

1. Exchange `RA` for `CT + DS`.
2. Exchange `DS + PA` for `RA` (using the generated `DS` from step 1).
3. Swap `CT` (generated in step 1) for `RA` in the Uniswap V2 pool.

## Vulnerability Detail

Let's consider the following scenario:

- `ds.exchangeRate() = 1.17`, so `1 CT + 1 DS = 1.17 RA`.
- `psmBaseRedemptionFeePercentage = 3%` (is used to deduct fees when exchanging `DS + PA` for `RA`).
- `1 CT = 1.07 RA` in the Uniswap V2 pool.
- The price of `PA` has significantly dropped, so `1 PA = 1 RA`.

Under these assumptions:

1. Alice exchanges `117 RA` for `100 * (CT + DS)` by calling the `Psm.depositPsm()` function.
2. Alice exchanges `100 * (DS + PA)` for `117 RA` by calling the `Psm.redeemRaWithDs()` function. However, due to the fee mechanism, Alice receives `(117 * 0.97 = 113.49) RA`.
3. Alice exchanges `100 CT` for `107 RA` in the Uniswap V2 pool.

As a result, Alice makes a profit of `-117 RA - 100 PA + 113.49 RA + 107 RA = 3.49 RA`.

This issue arises because users can immediately use the generated `DS` from step 1 to exchange `DS + PA` for `RA`. It would be more effective to implement a cooldown period from the generation timestamp before allowing the use of `DS`.

## Impact

Users can gain unfair profits.

## Code Snippet

https://github.com/sherlock-audit/2024-08-cork-protocol/tree/main/Depeg-swap/contracts/core/Psm.sol#L90-L101

https://github.com/sherlock-audit/2024-08-cork-protocol/tree/main/Depeg-swap/contracts/core/Psm.sol#L123-L140

## Tool used

Manual Review

## Recommendation

It would be more effective to implement a cooldown period from the generation timestamp before allowing the use of `DS`.