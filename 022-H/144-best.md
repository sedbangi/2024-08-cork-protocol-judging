Blurry Blush Mouse

High

# Users will steal excess funds from the Vault due to `VaultPoolLib::redeem()` not always decreasing `self.withdrawalPool.raBalance` and `self.withdrawalPool.paBalance`

### Summary

[Vault::redeemExpiredLv()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/Vault.sol#L128) calls [VaultLib::redeemExpired()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L514), which allows users to withdraw funds after expiry, even if they have not requested a redemption. This redemption happens in [VaultPoolLib::redeem()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L549), when [userEligible < amount](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L83), it calls internally [__redeemExcessFromAmmPool()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L89), where only [self.ammLiquidityPool.balance](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L167) is reduced, but not `self.withdrawalPool.raBalance` and `self.withdrawalPool.paBalance`. As such, when calculing the withdrawal pool balance in the next issuance on [VaultPoolLibrary::reserve()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L12), it will double count all the already [withdraw self.withdrawalPool.raBalance](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L17) and [self.withdrawalPool.paBalance](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L26), allowing users to withdraw the same funds twice.

### Root Cause

In `VaultPoolLib::__redeemExcessFromAmmPool()`, `self.withdrawalPool.raBalance` and `self.withdrawalPool.paBalance` are not decreased, but `ra` and `pa` are also withdrawn from the withdrawal pool when the user has partially requested redemption.

### Internal pre-conditions

1. User requests redemption of an amount smaller than the total withdrawn in `VaultLib::redeemExpired()`, that is, `userEligible < amount`.

### External pre-conditions

None.

### Attack Path

1. User calls `Vault::redeemExpiredLv()`, withdrawing from the withdrawal pool, but `self.withdrawalPool.raBalance` and `self.withdrawalPool.paBalance` are not decreased.
2. A new issuance starts, and in `VaultPoolLib::reserve()`, the funds are double counted as not all withdrawals were reduced.
3. As such, `self.withdrawalPool.raExchangeRate` and `self.withdrawalPool.paExchangeRate` will be inflated by double the funds and users will redeem more funds than they should, leading to the insolvency of the Vault.

### Impact

Users steal funds while unaware users will not be able to withdraw.

### PoC

`__tryRedeemExcessFromAmmPool()` does not decrease the withdrawn `self.withdrawalPool.raBalance` and `self.withdrawalPool.paBalance`.
```solidity
function __tryRedeemExcessFromAmmPool(VaultPool storage self, uint256 amountAttributed, uint256 excessAmount)
    internal
    view
    returns (uint256 ra, uint256 pa, uint256 withdrawnFromAmm)
{
    (ra, pa) = __tryRedeemfromWithdrawalPool(self, amountAttributed);

    withdrawnFromAmm =
        MathHelper.calculateRedeemAmountWithExchangeRate(excessAmount, self.withdrawalPool.raExchangeRate); //@audit PA is ignored here

    ra += withdrawnFromAmm;
}
```

### Mitigation

Replace `__tryRedeemfromWithdrawalPool()` with `__redeemfromWithdrawalPool()`.
```solidity
function __tryRedeemExcessFromAmmPool(VaultPool storage self, uint256 amountAttributed, uint256 excessAmount)
    internal
    view
    returns (uint256 ra, uint256 pa, uint256 withdrawnFromAmm)
{
    (ra, pa) = __redeemfromWithdrawalPool(self, amountAttributed);

    withdrawnFromAmm =
        MathHelper.calculateRedeemAmountWithExchangeRate(excessAmount, self.withdrawalPool.raExchangeRate); //@audit PA is ignored here

    ra += withdrawnFromAmm;
}
```