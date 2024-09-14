Colossal Magenta Elk

High

# RA amount is not updated properly in Vault::redeemEarlyLv()

### Summary

The purpose of the [Vault::redeemEarlyLv()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/Vault.sol#L187-L198) function is to redeem LV tokens for RA tokens. However in a series of overengineered internal calls it doesn't update the accounting of the RA asset within the protocol. Which leads to a mismatch between the accounting balance and the actual token balance. This will be problematic when users try to withdraw after  the DC and CT tokens have expired, or new CT and DS tokens are issued by the protocol for the existing pair of RA and PA. The [Vault::redeemEarlyLv()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/Vault.sol#L187-L198) function calls the [VaultLib::redeemEarly()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L639-L665) function which calls other internal functions but at the end the [VaultLib::_redeemCtDsAndSellExcessCt()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L318-L347) function is called:
```solidity
   function _redeemCtDsAndSellExcessCt(
        State storage self,
        uint256 dsId,
        IUniswapV2Router02 ammRouter,
        IDsFlashSwapCore flashSwapRouter,
        uint256 ammCtBalance
    ) internal returns (uint256 ra) {
        uint256 reservedDs = flashSwapRouter.getLvReserve(self.info.toId(), dsId);

        uint256 redeemAmount = reservedDs >= ammCtBalance ? ammCtBalance : reservedDs;

        reservedDs = flashSwapRouter.emptyReservePartial(self.info.toId(), dsId, redeemAmount);

        ra += redeemAmount;
        PsmLibrary.lvRedeemRaWithCtDs(self, redeemAmount, dsId);

        uint256 ctSellAmount = reservedDs >= ammCtBalance ? 0 : ammCtBalance - reservedDs;

        DepegSwap storage ds = self.ds[dsId];
        address[] memory path = new address[](2);
        path[0] = ds.ct;
        path[1] = self.info.pair1;

        ERC20(ds.ct).approve(address(ammRouter), ctSellAmount);

        if (ctSellAmount != 0) {
            // 100% tolerance, to ensure this not fail
            ra += ammRouter.swapExactTokensForTokens(ctSellAmount, 0, path, address(this), block.timestamp)[1];
        }
    }
```
Nowhere in the functions that were called before the [VaultLib::_redeemCtDsAndSellExcessCt()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L318-L347) function is called, is the amount of RA tokens decreased, and as be seen only the amounts of CT and DS tokens are decreased. There is another problem in this function but this is a separate issue. 
### Root Cause

In the [VaultLib::_redeemCtDsAndSellExcessCt()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L318-L347) function the following line:
```solidity
PsmLibrary.lvRedeemRaWithCtDs(self, redeemAmount, dsId);
```
Only burns CT and DS tokens, but it doesn't decrease the RA locked supply in the protocol

### Internal pre-conditions

1. Users have deposited RA tokens in the protocol to mint LV tokens
2. Users decide to redeem their LV tokens early by calling the [Vault::redeemEarlyLv()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/Vault.sol#L187-L198) function

### External pre-conditions

_No response_

### Attack Path

_No response_

### Impact

The RA accounting is not properly updated. Which leads to a mismatch between the accounting balance and the actual token balance. This will be problematic when users try to withdraw after  the DC and CT tokens have expired. The first users that withdraw via the [Psm::redeemWithCT()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/Psm.sol#L198-L210) function, will receive more tokens than they should, and the last users to withdraw will receive much less tokens that they should, if any. This results in the first users that withdraw effectively stealing funds from the last users that withdraw. There will also be problems when it comes to calculating the rewards for LV holders, and how much assets should be deposited into a newly created UniV2 pair, for DS and CT tokens that were just issued (the UniV2 pair consists of RA:CT tokens).

### PoC

[Gist](https://gist.github.com/AtanasDimulski/3f9bfc84c63e1c977b877613b644c0e2)
After following the steps in the above mentioned [gist](https://gist.github.com/AtanasDimulski/3f9bfc84c63e1c977b877613b644c0e2) add the following test to the ``AuditorTests.t.sol`` contract:
<details>
  <summary>POC 1</summary>

```solidity
    function test_RedeemEarlyLvIncorectRaUpdate() public {
        vm.startPrank(owner);
        /// @notice Set the redemption fee to 0 for easier calculations
        corkConfig.updateEarlyRedemptionFeeRate(id, 0);
        vm.stopPrank();

        vm.startPrank(alice);
        WETH.mint(alice, 10e18);
        WETH.approve(address(moduleCore), type(uint256).max);
        moduleCore.depositLv(id, 10e18);
        console2.log("Total RA locked(WETH) before redeem: ", moduleCore.valueLocked(id));
        console2.log("Actual balance of WETH in the contract: ", WETH.balanceOf(address(moduleCore)));
        Asset(lvAddress).approve(address(moduleCore), type(uint256).max);
        moduleCore.redeemEarlyLv(id, alice, 5e18);
        
        /// @notice alice will receive back 5e18 RA tokens(WETH)
        console2.log("Alice's RA(WETH) balance after redeem: ", WETH.balanceOf(alice));
        uint256 raAccountingAfterReddem = moduleCore.valueLocked(id);
        console2.log("Accounting RA locked(WETH) after redeem: ", raAccountingAfterReddem);

        /// @notice we get the RA reserve from the amm pair
        (uint112 raReserve, uint112 ctReserve) = flashSwapRouter.getAmmReserve(id, 1);
        console2.log("RAReserve in the amm pair contract: ", raReserve);
        uint256 actualWETHBalance = WETH.balanceOf(address(moduleCore));
        console2.log("Actual balance of WETH in the contract: ", actualWETHBalance);

        /// @notice calculate the the real amount of WETH in the contract plus the WETH deposited in the AMM
        uint256 actualWETHInTheSystem = uint256(raReserve) + actualWETHBalance;
        console2.log("Total RA(WETH) in the stystem: ", actualWETHInTheSystem);
        console2.log("Ra Accounting + RA in the AMM paid: ", raReserve + moduleCore.valueLocked(id));

        /// @notice alice started with 10e18 ETH, there was no  WETH locked before,
        /// but when the accouting of WETH and the current balacne of alice are added the result is bigger than 10e18,
        /// this is without the RA tokens in the AMM pair
        console2.log("Locked WETH + balance of alice: ", moduleCore.valueLocked(id) + WETH.balanceOf(alice));
        assert(raAccountingAfterReddem > actualWETHInTheSystem);
        vm.stopPrank();
    }
```

```solidity
Logs:
  Total RA locked(WETH) before redeem:  5263157894736842105
  Actual balance of WETH in the contract:  5263157894736842105
  Alice's RA(WETH) balance after redeem:  4999999999999998994
  Accounting RA locked(WETH) after redeem:  5263157894736842105
  RAReserve in the amm pair contract:  2368421052631579424
  Actual balance of WETH in the contract:  2631578947368421582
  Total RA(WETH) in the stystem:  5000000000000001006
  Ra Accounting + RA in the AMM paid:  7631578947368421529
  Locked WETH + balance of alice:  10263157894736841099
```
To run the test use: ``forge test -vvv --mt test_RedeemEarlyLvIncorectRaUpdate``
</details>

<details>
  <summary>POC 2</summary>

There is a second POC to better illustrate the severity of the problem, however there is a separate issue with one of the functions that have to be fixed first, since I am no sure what the correct fix is, that covers all cases I have fixed it in the following way. Note that the fix is not that important for this issue, it is used to better illustrate the severity of the issue.
Change the [DsFlashSwap::emptyReservePartial()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/DsFlashSwap.sol#L70-L78) function to:

```solidity
     function emptyReservePartial(ReserveState storage self, uint256 dsId, uint256 amount, address to)
        internal
        returns (uint256 reserve)
    {
        self.ds[dsId].ds.transfer(to, amount);

        self.ds[dsId].reserve -= amount;

        /// INFO: fix for reserve returning 0 when it is depleted
        reserve = amount;

        // reserve = self.ds[dsId].reserve;
    }
``` 

Then in the ``AuditorTests.t.sol`` add the following test:
```solidity
    function test_RedeemEarlyLvIncorectRaUpdateWithAFixForEmtpyReserve() public {
        vm.startPrank(owner);
        /// @notice Set the redemption fee to 0 for easier calculations
        corkConfig.updateEarlyRedemptionFeeRate(id, 0);
        vm.stopPrank();

        vm.startPrank(alice);
        WETH.mint(alice, 10e18);
        WETH.approve(address(moduleCore), type(uint256).max);
        moduleCore.depositLv(id, 10e18);
        console2.log("Total RA locked(WETH) before redeem: ", moduleCore.valueLocked(id));
        Asset(lvAddress).approve(address(moduleCore), type(uint256).max);
        moduleCore.redeemEarlyLv(id, alice, 10e18);

        console2.log("Actual WETH in the system: ", WETH.balanceOf(address(moduleCore)));
        console2.log("Total RA locked(WETH) after redeem: ", moduleCore.valueLocked(id));
        console2.log("Alice balance of LV: ", Asset(lvAddress).balanceOf(alice));

        (address ct, address ds) = moduleCore.swapAsset(id, 1);
        console2.log("Total ct issued: ", Asset(ct).totalSupply());
        vm.stopPrank();

        vm.startPrank(bob);
        WETH.mint(bob, 1e18);
        WETH.approve(address(moduleCore), type(uint256).max);
        moduleCore.depositPsm(id, 1e18);
        /// @notice we skip 1100 seconds so the ds and ct tokens expire
        skip(1100);
        Asset(ct).approve(address(moduleCore), type(uint256).max);
        console2.log("Total RA locked(WETH) after depositPSM: ", moduleCore.valueLocked(id));
        vm.expectRevert();
        moduleCore.redeemWithCT(id, 1, 0.5e18);
        vm.stopPrank();
    }
```

```solidity
Logs:
  Total RA locked(WETH) before redeem:  5263157894736842105
  Actual WETH in the system:  1057
  Total RA locked(WETH) after redeem:  5263157894736842105
  Alice balance of LV:  0
  Total ct issued:  1057
  Total RA locked(WETH) after depositPSM:  6263157894736842105
```

To run the test use: ``forge test -vvv --mt test_RedeemEarlyLvIncorectRaUpdateWithAFixForEmtpyReserve``
</details>

### Mitigation

In the [VaultLib::_redeemCtDsAndSellExcessCt()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L318-L347) function decrease the RA locked amount in the protocol. 