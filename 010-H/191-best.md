Blurry Blush Mouse

High

# `VaultPoolLib::reserve()` will store the `Pa` not attributed to user withdrawals incorrectly and leave in untracked once it expires again

### Summary

[VaultPoolLib::reserve()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L12) stores the `Pa` attributed to withdrawals in [self.withdrawalPool.stagnatedPaBalance](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L31) instead of storing the amount `attributedToAmm`. Additionally, this amount of `Pa`, the one attributed to the `Amm` is never dealt with and leads to stuck `PA`.

The comment in the code mentions 
```solidity
    // FIXME : this is only temporary, for now
    // we trate PA the same as RA, thus we also separate PA
    // the difference is the PA here isn't being used as anything
    // and for now will just sit there until rationed again at next expiry.
```
But it is incorrect as it is never rationed again, just forgotten. The [VaultPoolLib::rationedToAmm()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L170) function only uses the `Ra` [balance](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L171), not the `Pa`, which is effectively left untracked.

### Root Cause

In `VaultPoolLib:170`, the leftover non attributed `Pa` is not dealt with.

### Internal pre-conditions

None.

### External pre-conditions

None.

### Attack Path

1. `VaultPoolLib::reserve()` is called when liquidating the lp position of the `Vault` via `VaultLib::_liquidatedLP()`, triggered by users when redeeming expired liquidity vault shares or on the admin trigerring a new issuance.

### Impact

The `Pa` in the `Vault` is stuck.

### PoC

`VaultPoolLib::rationedToAmm()` does not deal with the `Pa`.
```solidity
function rationedToAmm(VaultPool storage self, uint256 ratio) internal view returns (uint256 ra, uint256 ct) {
    uint256 amount = self.ammLiquidityPool.balance;

    (ra, ct) = MathHelper.calculateProvideLiquidityAmountBasedOnCtPrice(amount, ratio);
}
```

### Mitigation

Distributed the `Pa` to users based on their `LV` shares or redeem the `Pa` for `Ra` and add liquidity to the new issued `Ds` or similar.