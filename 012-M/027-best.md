High Flint Llama

Medium

# Premature LV Redemption Leading to Asset Loss For user

### Summary

 According to the Cork protocol's litepaper, Liquidity Vault tokenholders can submit a withdrawal request prior to expiry, which will be processed at expiry and can be claimed by burning the Liquidity Vault Token. However, the current implementation allows users to call `VaultCore::redeemExpiredLv` before expiry, resulting in the complete loss of user funds.

### Root Cause

In `VaultLibrary::redeemExpired`, if a user calls the function before the expiry and their userEligible balance is greater than or equal to the requested amount, the transaction proceeds without invoking `_liquidatedLp(self, dsId, ammRouter, flashSwapRouter)`. 

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L514

```solidity
 {
            uint256 dsId = self.globalAssetIdx;
            DepegSwap storage ds = self.ds[dsId];

            
            uint256 userEligible = self.vault.pool.withdrawEligible[owner];

            if (userEligible == 0 && !ds.isExpired()) {
                revert Unauthorized(owner);
            }

            // user can only redeem up to the amount they requested, when there's a DS active
            // if there's no DS active, then there's no cap on the amount of LV that can be redeemed
            if (!ds.isExpired() && userEligible < amount) {
                revert InsufficientBalance(owner, amount, userEligible);
            }

            if (ds.isExpired() && !self.vault.lpLiquidated.get(dsId)) {
                _liquidatedLp(self, dsId, ammRouter, flashSwapRouter);
                assert(self.vault.balances.ra.locked == 0);
            }
        }
```

Since this function (`_liquidatedLp`) is responsible for setting the exchange rates (`withdrawalPool.raExchangeRate` and `withdrawalPool.paExchangeRate`), these rates remain uninitialized (set to 0).

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L392


As a result, when calculating the user's allocated Redemption Asset (RA) and Pegged Asset (PA), the allocation is zero. 

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L137

The function burns Liquidity Vault (LV) tokens, decreases userEligible, but sends 0 RA and PA to the user, leading to a complete loss of funds.


### Impact

Users who attempt to call `redeemExpiredLv` before the Depeg Swap (DS) expiry risk losing all their funds.

### PoC

Run the poc in test/contracts/LvCore.ts

```ts
 describe("LossFund", function () {
    it("call redeemExpiredLv before expiry ", async function () {

      const { Id, dsId } = await issueNewSwapAssets(expiry);

      let paBalance = await fixture.pa.read.balanceOf(
        [secondSigner.account.address],
      );
      let rabalance = await fixture.ra.read.balanceOf([
        secondSigner.account.address,
      ]);

      console.log("Ra balance of user before deposit ",rabalance);


      await moduleCore.write.depositLv([Id, depositAmount]);
      await moduleCore.write.depositLv([Id, depositAmount], {
        account: secondSigner.account,
      });

      const msgPermit = await helper.permit({
        amount: depositAmount,
        deadline,
        erc20contractAddress: fixture.lv.address!,
        psmAddress: moduleCore.address,
        signer: secondSigner,
      });

      await moduleCore.write.requestRedemption(
        [Id, depositAmount, msgPermit, deadline],
        {
          account: secondSigner.account,
        }
      );

      let lvBalance = await moduleCore.read.lockedLvfor(
        [Id, secondSigner.account.address],
        {
          account: secondSigner.account,
        }
      );

      console.log("LV locked for user before request redemption ",lvBalance);
    
      await moduleCore.write.redeemExpiredLv(
        [Id, secondSigner.account.address, depositAmount],
        {
          account: secondSigner.account,
        }
      );

      paBalance = await fixture.pa.read.balanceOf(
        [secondSigner.account.address],
      );
      rabalance = await fixture.ra.read.balanceOf([
        secondSigner.account.address,
      ]);

      lvBalance = await moduleCore.read.lockedLvfor(
        [Id, secondSigner.account.address],
        {
          account: secondSigner.account,
        }
      );

      console.log("LV locked for user after redemption ", lvBalance);
      
      console.log("PA balance of user after redemption ", paBalance);
      console.log("RA balance of user after redemption ", rabalance);
    });
  });
```

### Output

![Screenshot from 2024-09-04 18-47-09](https://github.com/user-attachments/assets/f150cb67-7a2b-4cfa-9adf-6c72cfbd2063)


### Mitigation

The `redeemExpired` function is intended to be called at DS expiry. To prevent misuse, a simple fix would be to add a check to ensure it reverts when called before expiry. The `redeemEarlyLv` function already handles LV redemption before expiry. However, if the current behavior is intended, the internal function logic should be refactored to properly calculate and set the exchange rates even before expiry to avoid zero allocations and loss of user funds. 