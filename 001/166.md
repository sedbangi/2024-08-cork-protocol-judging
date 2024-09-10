Blurry Blush Mouse

High

# Users redeeming early will withdraw `Ra` without decreasing the amount locked, which will lead to stolen funds when withdrawing after expiry

### Summary

[VaultLib::redeemEarly()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L639) is called when users redeem early via [Vault::redeemEarlyLv()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/Vault.sol#L168), which allows users to redeem `Lv` for `Ra` and pay a fee. 

In the process, the `Vault` burns `Ct` and `Ds` in [VaultLib::_redeemCtDsAndSellExcessCt()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L318) for `Ra`, by calling [PsmLib::PsmLibrary.lvRedeemRaWithCtDs()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L332). However, it never calls [RedemptionAssetManagerLib::decLocked()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/RedemptionAssetManagerLib.sol#L57) to decrease the tracked locked `Ra`, but the `Ra` leaves the `Vault` for the user redeeming.

This means that when a new `Ds` is [issued](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L43) in the `PsmLib` or users call [PsmLib::redeemWithCt()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L448), [PsmLib::_separateLiquidity()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L65) will be called and it will calculated the exchange rate to withdraw `Ra` and `Pa` as if the `Ra` amount withdrawn earlier was still there. When it calls [self.psm.balances.ra.convertAllToFree()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L73), it [converts](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/RedemptionAssetManagerLib.sol#L42) the locked amount to free and assumes these funds are available, when in reality they have been withdrawn earlier. As such, the `Ra` and `Pa` [checkpoint](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L76) will be incorrect and users will redeem more `Ra` than they should, such that the last users will not be able to withdraw and the first ones will profit.

### Root Cause

In `PsmLib.sol:125`, `self.psm.balances.ra.decLocked(amount);` is not called.

### Internal pre-conditions

None.

### External pre-conditions

None.

### Attack Path

1. User calls `Vault::redeemEarlyLv()` or `ModuleCore::issueNewDs()` is called by the admin.

### Impact

Users withdraw more funds then they should via `PsmLib::redeemWithCt()` meaning the last users can not withdraw.

### PoC

`PsmLib::lvRedeemRaWithCtDs()` does not reduce the amount of `Ra` locked.
```solidity
function lvRedeemRaWithCtDs(State storage self, uint256 amount, uint256 dsId) internal {
    DepegSwap storage ds = self.ds[dsId];
    ds.burnBothforSelf(amount);
}
```

### Mitigation

`PsmLib::lvRedeemRaWithCtDs()` must reduce the amount of `Ra` locked.
```solidity
function lvRedeemRaWithCtDs(State storage self, uint256 amount, uint256 dsId) internal {
    self.psm.balances.ra.decLocked(amount);
    DepegSwap storage ds = self.ds[dsId];
    ds.burnBothforSelf(amount);
}
```