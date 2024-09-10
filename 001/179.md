Blurry Blush Mouse

High

# Users redeeming with `Ds` in the `Psm` will not decrease the correct amount of locked `Ra`, leading to stolen funds on withdrawals

### Summary

[Psm::redeemRaWithDs()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/Psm.sol#L142) calls [PsmLib::redeemWithDs()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L364), which [calls](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L379) [PsmLib::_afterRedeemWithDs()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L329) to send the unlocked `Ra` to the owner and [decrease](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/RedemptionAssetManagerLib.sol#L70) `self.psm.balances.ra.locked` by calling [self.psm.balances.ra.unlockTo(owner, received);](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L346), that is, the amount of `Ra` locked in the contract. 

However, in the process, the `Ra` amount decreased from `self.psm.balances.ra.locked` is just the `received`, which has been [subtracted](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L342) the fee. But, this fee is used to provide liquidity, which means it is no longer locked and available, leading to the `Psm` having a bigger `self.psm.balances.ra.locked` than it should.

As such, on [PsmLib::_separateLiquidity()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L73), more `Ra` than what really exists is assumed to be [available](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/RedemptionAssetManagerLib.sol#L42) for withdrawal, which will make some users steal `Ra` when withdrawing and the last users not being able to withdraw.

### Root Cause

In `PsmLib:346`, the `Ra` locked is [decreased](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/RedemptionAssetManagerLib.sol#L70) by the received amount [minus](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L342) the fee.

### Internal pre-conditions

None.

### External pre-conditions

None.

### Attack Path

1. User calls `Psm::redeemWithDs()`.

### Impact

Users withdrawing first steal funds and the last users will not be able to withdraw their funds at all.

### PoC

In `PsmLib::_afterRedeemWithDs()`, `self.psm.balances.ra.unlockTo(owner, received);` is called with the received amount minus the fee:
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

In `RedemptionAssetManagerLib::unlockTo()` it decreases the locked `Ra` by the amount passed:
```solidity
function unlockTo(PsmRedemptionAssetManager storage self, address to, uint256 amount) internal {
    decLocked(self, amount);
    unlockToUnchecked(self, amount, to);
}
```

### Mitigation

In `PsmLib::_afterRedeemWithDs()`, the user should receive the amount minus the fee but the locked `Ra` should decrease by the full amount:
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

    self.psm.balances.ra.decLocked(received); //@audit full amount is decreased
    received -= fee;
    self.psm.balances.ra.unlockToUnchecked(received, owner); //@audit owner receives amount - fee

    IERC20(self.info.peggedAsset().asErc20()).safeTransferFrom(owner, address(this), amount);
}
```