Narrow Iron Zebra

High

# Insufficient `Ra/Pa` Balances Cause Reduced Payout In The `redeemWithCt` Function.


## Summary

If the available Pa or Ra balance in the protocol is insufficient, users may redeem their CT tokens but receive fewer Ra and Pa tokens than they are entitled to, resulting in a disproportionate or reduced payout.

## Vulnerability Detail


1. **User Deposits Ra Tokens:**
   - The user deposits `100e18` Ra tokens into the PSM. The contract mints `100e18` CT tokens in return, along with `100e18` Ds tokens, based on a 1:1 exchange rate. The deposit process locks and transfer the Ra tokens from the depositor and issues the corresponding CT:DS tokens.

```solidity
    function deposit(State storage self, address depositor, uint256 amount)
        internal
        returns (uint256 dsId, uint256 received, uint256 _exchangeRate)
    {
        if (amount == 0) {
            revert ICommon.ZeroDeposit();
        }

        dsId = self.globalAssetIdx;

        DepegSwap storage ds = self.ds[dsId];

        Guard.safeBeforeExpired(ds);
        _exchangeRate = ds.exchangeRate();

        received = MathHelper.calculateDepositAmountWithExchangeRate(amount, _exchangeRate);

        self.psm.balances.ra.lockFrom(amount, depositor);

        ds.issue(depositor, received);
    }
```
2. **User Redeems CT Tokens:**
   - The user initiates the redemption process by returning `100e18` CT tokens to the contract and called redeemWithCt function. The protocol then calculates how many Ra and Pa tokens the user should receive in exchange for the redeemed CT tokens.

```solidity
    function redeemWithCt(
        State storage self,
        address owner,
        uint256 amount,
        uint256 dsId,
        bytes memory rawCtPermitSig,
        uint256 deadline
    ) internal returns (uint256 accruedPa, uint256 accruedRa) {
        DepegSwap storage ds = self.ds[dsId];
        Guard.safeAfterExpired(ds);
        if (deadline != 0) {
            DepegSwapLibrary.permit(ds.ct, rawCtPermitSig, owner, address(this), amount, deadline);
        }
        _separateLiquidity(self, dsId);

        uint256 totalCtIssued = self.psm.poolArchive[dsId].ctAttributed;
        PsmPoolArchive storage archive = self.psm.poolArchive[dsId];

        (accruedPa, accruedRa) = _calcRedeemAmount(amount, totalCtIssued, archive.raAccrued, archive.paAccrued);

        _beforeCtRedeem(self, ds, dsId, amount, accruedPa, accruedRa);

        _afterCtRedeem(self, ds, owner, amount, accruedPa, accruedRa);
    }
```

3. **Calling _separateLiquidity to Update Available Balances:**
   - Then `redeemWithCt` function calls the `_separateLiquidity` function in that function all locked Ra and PaBalance tokens are converted into "free" balances, which become available for redemption.

```solidity
    function _separateLiquidity(State storage self, uint256 prevIdx) internal {
        if (self.psm.liquiditySeparated.get(prevIdx)) {
            return;
        }

        DepegSwap storage ds = self.ds[prevIdx];
        Guard.safeAfterExpired(ds);

@>>       uint256 availableRa = self.psm.balances.ra.convertAllToFree();
@>>       uint256 availablePa = self.psm.balances.paBalance;

        self.psm.poolArchive[prevIdx] = PsmPoolArchive(availableRa, availablePa, IERC20(ds.ct).totalSupply());

        // reset current balances
        self.psm.balances.ra.reset();
        self.psm.balances.paBalance = 0;

        self.psm.liquiditySeparated.set(prevIdx);
    }
```
4. **Available Ra and Pa Balances Are Checked:**
   - The `_calcRedeemAmount` function is called to calculate the accrued Ra and Pa tokens based on the available supply of Ra (`availableRa`) and Pa (`availablePa`) in the protocol, along with the total amount of CT tokens issued (`totalCtIssued`).
   - This calculation distributes the available Ra and Pa proportionally based on the amount of CT tokens the user redeems.

```solidity
    function _calcRedeemAmount(uint256 amount, uint256 totalCtIssued, uint256 availableRa, uint256 availablePa)
        internal
        pure
        returns (uint256 accruedPa, uint256 accruedRa)
    {
        accruedPa = MathHelper.calculateAccrued(amount, availablePa, totalCtIssued);

        accruedRa = MathHelper.calculateAccrued(amount, availableRa, totalCtIssued);
    }
```

4. **Insufficient Ra or Pa Tokens:**
   - If the available Ra or Pa tokens in the protocol are insufficient (i.e., there are fewer Ra or Pa tokens than needed to fully satisfy the redemption), the user will still redeem their `100e18` CT tokens but will receive fewer Ra and Pa tokens than expected.
   - This discrepancy occurs because the calculation divides the available Ra and Pa tokens across all issued CT tokens, and if the balances are lower than expected, the user gets less than the full value they anticipated.

```solidity
        _beforeCtRedeem(self, ds, dsId, amount, accruedPa, accruedRa);

        _afterCtRedeem(self, ds, owner, amount, accruedPa, accruedRa);


    function _beforeCtRedeem(
        State storage self,
        DepegSwap storage ds,
        uint256 dsId,
        uint256 amount,
@>        uint256 accruedPa,
@>        uint256 accruedRa
    ) internal {
        ds.ctRedeemed += amount;
        self.psm.poolArchive[dsId].ctAttributed -= amount;
@>        self.psm.poolArchive[dsId].paAccrued -= accruedPa;
@>        self.psm.poolArchive[dsId].raAccrued -= accruedRa;
    }

    function _afterCtRedeem(
        State storage self,
        DepegSwap storage ds,
        address owner,
        uint256 ctRedeemedAmount,
        uint256 accruedPa,
        uint256 accruedRa
    ) internal {
        // user transfer full amount ctRedeemedAmount but possible receive a reduced Ra:PA
        IERC20(ds.ct).transferFrom(owner, address(this), ctRedeemedAmount);
        IERC20(self.info.peggedAsset().asErc20()).safeTransfer(owner, accruedPa);
        IERC20(self.info.redemptionAsset()).safeTransfer(owner, accruedRa);
    }
```

5. **No Mechanism to Prevent Under-Redemption:**
   - The contract does not have a guard or mechanism to stop the transaction or alert the user if the available Ra and Pa tokens are insufficient. As a result, users may unknowingly redeem their CT tokens and receive a reduced payout.


This behavior can be exploited if users are unaware of the current liquidity in the protocol, leading to a potential loss in the amount of Ra and Pa tokens they would otherwise expect to receive.


## Impact

Users may redeem their CT tokens while receiving fewer Ra and Pa tokens than expected due to insufficient balances in the protocol. This could result in an unfair distribution of assets, causing users to unknowingly incur losses. 


## Code Snippet

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L448-L471

## Tool used

Manual Review

## Recommendation

Implement a Check for Sufficient Balances before processing a redemption request, the contract should check if the available Ra and Pa balances are sufficient to cover the full redemption amount. If not, the transaction should be reverted.
