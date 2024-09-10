Proper Turquoise Goldfish

Medium

# Missing Fees in `redeemRaWithCtDs()`

### Summary

In the `PSM.sol` contract, the `redeemRaWithDs()` function collects fees when redeeming the Redemption Asset (RA) using Depeg Swaps (DS).

```solidity
    function redeemRaWithDs(Id id, uint256 dsId, uint256 amount)
        external
        override
        nonReentrant
        onlyInitialized(id)
        PSMWithdrawalNotPaused(id)
    {
        State storage state = states[id];
        // gas savings
        uint256 feePrecentage = psmBaseRedemptionFeePrecentage;

        (uint256 received, uint256 _exchangeRate, uint256 fee) =
            state.redeemWithDs(_msgSender(), amount, dsId, bytes(""), 0, feePrecentage); 

        VaultLibrary.provideLiquidityWithFee(state, fee, getRouterCore(), getAmmRouter());

        emit DsRedeemed(id, dsId, _msgSender(), amount, received, _exchangeRate, feePrecentage, fee);
    }
```

However, in the `redeemRaWithCtDs()` function, which redeems RA using both DS and Cover Tokens (CT), **no fees are collected**.

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/Psm.sol#L242-L255

```solidity
    function redeemRaWithCtDs( 
        Id id,
        uint256 amount,
        bytes memory rawDsPermitSig,
        uint256 dsDeadline,
        bytes memory rawCtPermitSig,
        uint256 ctDeadline
    ) external override nonReentrant PSMWithdrawalNotPaused(id) {
        State storage state = states[id];
        (uint256 ra, uint256 dsId, uint256 rates) =
            state.redeemRaWithCtDs(_msgSender(), amount, rawDsPermitSig, dsDeadline, rawCtPermitSig, ctDeadline); 

        emit Cancelled(id, dsId, _msgSender(), ra, amount, rates);
    }
```

This discrepancy creates an **unfair advantage for users** redeeming RA through `redeemRaWithCtDs()` since they bypass the fee collection mechanism that is intended to apply to all redemptions within the Peg Stability Module (PSM).

From the protocol Litepaper we can see: 

> Revenue sources & fee mechanisms
> - Redemption Fees: A fee in the Peg Stability Module to execute **Redemptions**.

https://corkfi.notion.site/Cork-Protocol-Litepaper-f21a57d5c19d48209dfa0f0c2ab776c4

This missing fee can result in **revenue loss for the protocol** and encourages strategic redemption behavior where users deliberately choose the method with no fees, undermining the protocol’s intended revenue-generation mechanisms. 

Additionally, the lack of fees in `redeemRaWithCtDs()` could disrupt the balance between different redemption methods, as users may consistently prefer this route, resulting in liquidity imbalances or mispricing within the PSM.

Note: Probably fees should also be taken from `redeemWithCT()`.

### Root Cause

**Lack of fee collection logic** in the `redeemRaWithCtDs()` function when performing redemptions using Cover Tokens (CT) and Depeg Swaps (DS), which contradicts the protocol's documented fee structure for redemptions within the Peg Stability Module (PSM).

### Internal pre-conditions

No

### External pre-conditions

No

### Attack Path

- A user with both DS and CT tokens decides to redeem RA via the `redeemRaWithCtDs()` function.
- The user bypasses the fee mechanism that would have been applied in other redemption methods (such as `redeemRaWithDs()`).
- This leads to an unfair fee avoidance, allowing the user to retain more RA than they would have if a fee were applied.

### Impact

The function does not collect fees.

### PoC

_No response_

### Mitigation

Introduce fee logic in `redeemRaWithCtDs()` that is similar to the logic in `redeemRaWithDs()`.