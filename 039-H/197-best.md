Tame Lemonade Squid

High

# `_afterRedeemWithDs` function have Incorrect PA Transfer that Blocks DS Redemptions

### Summary

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L329

in the  function  `_afterRedeemWithDs`


```solidity
function _afterRedeemWithDs(
    State storage self,
    DepegSwap storage ds,
    address owner,
    uint256 amount,
    uint256 feePrecentage
) internal returns (uint256 received, uint256 _exchangeRate, uint256 fee) {
    IERC20(ds._address).transferFrom(owner, address(this), amount);

    _exchangeRate = ds.exchangeRate();
    received = MathHelper.calculateRedeemAmountWithExchangeRate(amount, _exchangeRate);

    fee = MathHelper.calculatePrecentageFee(received, feePrecentage);
    received -= fee;

    IERC20(self.info.peggedAsset().asErc20()).safeTransferFrom(owner, address(this), amount);

    self.psm.balances.ra.unlockTo(owner, received);
}
```

The logic bug in this function is :

The function is transferring both the Depeg Swap (DS) tokens and the Pegged Asset (PA) tokens from the user, but it's only giving back the Redemption Asset (RA) in return. This is inconsistent with the intended behavior of the protocol as described in the documentation.

According to the litepaper, when redeeming with DS:

> 1. The user should be able to exchange their Pegged Asset + Depeg Swap for the Redemption Asset.
> 2. The exchange should be 1:1 (minus fees).

However, in this function:

1. It takes `amount` of DS tokens from the user.
2. It also takes `amount` of PA tokens from the user.
3. It calculates the received amount based only on the DS amount and exchange rate.
4. It sends back only the RA to the user.

Context from the codebase:
   - The `_afterRedeemWithDs` function is called by `redeemWithDs` in the `PsmLibrary`.
   - `redeemWithDs` is then called by `redeemRaWithDs` in the `PsmCore` contract.

2. Intended flow:
   - Users should be able to redeem their DS (Depeg Swap) tokens for RA (Redemption Asset) tokens.
   - The PA (Pegged Asset) is not involved in this specific redemption process.

3. Actual implementation:
   - The `redeemRaWithDs` function in `PsmCore` only takes DS tokens from the user.
   - It doesn't request PA tokens from the user for this operation.

4. The confusing part in `_afterRedeemWithDs`:
   - There is indeed a line that transfers PA tokens:
     ```solidity
     IERC20(self.info.peggedAsset().asErc20()).safeTransferFrom(owner, address(this), amount);
     ```
   -  this transfer will fail because the PSM contract doesn't have approval to transfer PA tokens from the user in this context.


The presence of the PA transfer in `_afterRedeemWithDs` is unnecessary and causes all DS redemptions to fail. This is a significant bug

Impact:

1. Users cannot redeem their DS tokens for RA tokens.
2. This breaks a core functionality of the protocol, preventing users from exiting their positions.


Proof of Concept:

1. User acquires DS tokens.
2. User attempts to redeem DS tokens for RA using `redeemRaWithDs`.
3. The transaction always reverts due to the unauthorized PA transfer attempt.
4. User is unable to complete the redemption, locking their assets in the protocol.

To fix this, the PA transfer line should be removed from the `_afterRedeemWithDs` function. The correct implementation should only involve transferring DS tokens from the user and RA tokens to the user.















### Root Cause

_No response_

### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

_No response_

### Impact

_No response_

### PoC

_No response_

### Mitigation

_No response_