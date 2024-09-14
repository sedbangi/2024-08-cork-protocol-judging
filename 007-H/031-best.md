Proper Turquoise Goldfish

High

# Incorrect argument passing in the fee calculation will cause huge fee

### Summary

When a user wants to withdraw LV tokens before expiration, he uses the `redeemEarlyLv`() function.
```solidity
    function redeemEarlyLv(Id id, address receiver, uint256 amount)
        external
        override
        nonReentrant
        LVWithdrawalNotPaused(id)
    {
        State storage state = states[id];
        (uint256 received, uint256 fee, uint256 feePrecentage) =
            state.redeemEarly(_msgSender(), receiver, amount, getRouterCore(), getAmmRouter(), bytes(""), 0);

        emit LvRedeemEarly(id, _msgSender(), receiver, received, fee, feePrecentage);
    }
```

The main logic is in the internal function `redeemEarly()`:

```solidity
    function redeemEarly(
        State storage self,
        address owner,
        address receiver,
        uint256 amount,
        IDsFlashSwapCore flashSwapRouter,
        IUniswapV2Router02 ammRouter,
        bytes memory rawLvPermitSig,
        uint256 deadline
    ) external returns (uint256 received, uint256 fee, uint256 feePrecentage) {
        safeBeforeExpired(self);
        if (deadline != 0) {
            DepegSwapLibrary.permit(self.vault.lv._address, rawLvPermitSig, owner, address(this), amount, deadline);
        }

        feePrecentage = self.vault.config.fee;

        received = _liquidateLpPartial(self, self.globalAssetIdx, flashSwapRouter, ammRouter, amount);

        fee = MathHelper.calculatePrecentageFee(received, feePrecentage); //@audit - incorect agruments


        provideLiquidityWithFee(self, fee, flashSwapRouter, ammRouter);
        received = received - fee;

        ERC20Burnable(self.vault.lv._address).burnFrom(owner, amount);
        self.vault.balances.ra.unlockToUnchecked(received, receiver);
    }
```

What we need to pay attention to is how the fees are calculated. 
```solidity
fee = MathHelper.calculatePrecentageFee(received, feePrecentage);
```
Note how the function arguments are passed. 

This is a `calculatePrecentageFee()` function:
```solidity
    /**
     * @dev calculate the fee in respect to the amount given
     * @param fee1e18 the fee in 1e18
     * @param amount the amount of lv user want to withdraw
     */
    function calculatePrecentageFee(uint256 fee1e18, uint256 amount) external pure returns (uint256 precentage) {
        precentage = (((amount * 1e18) * fee1e18) / (100 * 1e18)) / 1e18;
    }
```

The function misplaces the arguments when calling `MathHelper.calculatePrecentageFee()`, passing the **withdrawal amount** (`received`) as the fee percentage, and the **fee percentage** (`feePrecentage`) as the withdrawal amount. 

This mistake causes the contract to calculate an unreasonably high fee, leading to a significant loss of funds.

***

**Note**: The exact same bug occurs in `previewRepurchase()` and `_afterRedeemWithDs()` in `PsmLib.sol` contract. The impact is also High when a user redeems an RA with DS or when the `repurchase()` function is used by the user. 

For better structuring of the issue, I have given an example with `redeemEarlyLv()`, but the same applies to the mentioned functions.
***

### Root Cause

The root cause of this vulnerability is the incorrect order of arguments passed to the `MathHelper.calculatePrecentageFee` function. Instead of passing the **fee percentage** as the first argument and the **received amount** as the second, the arguments are swapped, leading to an incorrect fee calculation.

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L658
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L678
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L276
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L341

### Internal pre-conditions

No Internal pre-conditions. Always happening

### External pre-conditions

No External pre-conditions. Always happening

### Attack Path

- The user initiates the early redemption process by calling the `redeemEarlyLv()` function with a valid amount.
-  The function call `calculatePrecentageFee` with wrong arguments.
- The function treats the **received amount** as the fee percentage and the **fee percentage** as the amount, leading to a drastically inflated fee.

### Impact

Liquidity Vault (LV) token holders suffer a huge loss. The user does not gain any additional funds but rather faces significant financial loss, as the system overcharges them on fees during early withdrawal.

### PoC

_No response_

### Mitigation

The correct function call should pass the **fee percentage** as the first argument and the **received amount** as the second, like this:

```solidity
fee = MathHelper.calculatePrecentageFee(feePrecentage, received);
```

Fix the bug also in `previewRepurchase()` and `_afterRedeemWithDs()`.