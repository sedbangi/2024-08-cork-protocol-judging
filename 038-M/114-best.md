Colossal Magenta Elk

Medium

# If all LV tokens are requested for redeem, issuing new CT and DS tokens will revert

### Summary

The Corc protocol creates a pair for RA:PA tokens, and then can issue CT and DS tokens for that pair, which can be used for different purposes such as hedging against PA tokens deepening from the RA token. The CT and DS tokens have an expiration. Once they have expired the protocol admins can issue new CT and DS tokens via the [ModuleCore::issueNewDs()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/ModuleCore.sol#L57-L86) function. However in some cases that won't be possible. The [ModuleCore::issueNewDs()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/ModuleCore.sol#L57-L86) function calls a lot of functions that have to do with deploying asset contracts and setting up parameters, but the problem arises in the [VaultLib::onNewIssuance()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L92-L108) function, which calls the [VaultLib::__provideAmmLiquidityFromPool()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L174-L189) function:
```solidity
    function __provideAmmLiquidityFromPool(
        State storage self,
        IDsFlashSwapCore flashSwapRouter,
        address ctAddress,
        IUniswapV2Router02 ammRouter
    ) internal {
        uint256 dsId = self.globalAssetIdx;

        uint256 ctRatio = __getAmmCtPriceRatio(self, flashSwapRouter, dsId);

        (uint256 ra, uint256 ct) = self.vault.pool.rationedToAmm(ctRatio);

        __provideLiquidity(self, ra, ct, flashSwapRouter, ctAddress, ammRouter, dsId);

        self.vault.pool.resetAmmPool();
    }
```
There are several overengineered calls that first separate the liquidity for the previously expired CT and DS tokens, based on how much LV tokens users minted, and how much of those LV tokens were requested for withdrawal. If all of the users that minted LV tokens request to withdraw them, the internal call to [VaultPoolLib::rationedToAmm()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L170-L174) function will return 0. And when the contract tries to deposit 0 RA and 0 CT tokens to a newly created UniV2 Pair, the contract will revert due to underflow. 
```solidity
    function mint(address to) external lock returns (uint liquidity) {
        ...
        uint _totalSupply = totalSupply; // gas savings, must be defined here since totalSupply can update in _mintFee
        if (_totalSupply == 0) {
            liquidity = Math.sqrt(amount0.mul(amount1)).sub(MINIMUM_LIQUIDITY);
            _mint(address(0), MINIMUM_LIQUIDITY); // permanently lock the first MINIMUM_LIQUIDITY tokens
        } else {
            liquidity = Math.min(amount0.mul(_totalSupply) / _reserve0, amount1.mul(_totalSupply) / _reserve1);
        }
        ...
    }
```
All users requesting to redeem their LV in the same time frame is fairly possible scenario. Keep in mind that there is one LV token for a RA:PA pair, and several CT and DS tokens can be issued for a pair of RA:PA tokens. So it just may happens that at the 3rd issuing of CT and DS tokens, all users have requested to redeem their LV tokens. 

### Root Cause

In the [VaultLib::__provideAmmLiquidityFromPool()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L174-L189) function there is no  check whether there is liquidity that can be provided to the newly created UniV2 pair.

### Internal pre-conditions
1. All the LV tokens that were minted by the system are requested for redemption.

### External pre-conditions

_No response_

### Attack Path

_No response_

### Impact

The [ModuleCore::issueNewDs()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/ModuleCore.sol#L57-L86) function is most critical function for the protocol, without it new DS and CT tokens can't be issued and the protocol becomes obsolete, simply redeploying the protocol won't fix this issue, as there can be other pairs for RA:PA tokens, and there can be tokens that users are still to reclaim, keep in mind that once CT and DS tokens have expired RA tokens that were deposited by users can still be withdrawn from users by calling the [Psm::redeemWithCT()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/Psm.sol#L198-L210) function and depositing CT tokens, that have already expired. Thus the medium severity.

### PoC

[Gist](https://gist.github.com/AtanasDimulski/3f9bfc84c63e1c977b877613b644c0e2)

After following the steps in the above mentioned [gist](https://gist.github.com/AtanasDimulski/3f9bfc84c63e1c977b877613b644c0e2) add the following test to the ``AuditorTests.t.sol`` file:

```solidity
    function test_IssuingNewDsWillRevertWhenNoLiquidityCanBeProvided() public {
        vm.startPrank(alice);
        WETH.mint(alice, 10e18);
        WETH.approve(address(moduleCore), type(uint256).max);
        moduleCore.depositLv(id, 10e18);
        Asset(lvAddress).approve(address(moduleCore), type(uint256).max);
        moduleCore.requestRedemption(id, 10e18);
        vm.stopPrank();

        vm.startPrank(bob);
        WETH.mint(bob, 10e18);
        WETH.approve(address(moduleCore), type(uint256).max);
        moduleCore.depositPsm(id, 10e18);
        vm.stopPrank();

        vm.startPrank(tom);
        WETH.mint(tom, 10e18);
        WETH.approve(address(moduleCore), type(uint256).max);
        moduleCore.depositLv(id, 10e18);
        Asset(lvAddress).approve(address(moduleCore), type(uint256).max);
        moduleCore.requestRedemption(id, 10e18);
        vm.stopPrank();

        /// @notice skip 1100 seconds so that the CT and DS tokens are expired
        skip(1100);

        vm.startPrank(owner);
        vm.expectRevert(bytes("ds-math-sub-underflow"));
        corkConfig.issueNewDs(id, block.timestamp + expiry, 1e18, 5e18);
        vm.stopPrank();
    }
```

To run the test use: ``forge test -vvv --mt test_IssuingNewDsWillRevertWhenNoLiquidityCanBeProvided``

### Mitigation

In the [VaultLib::__provideAmmLiquidityFromPool()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L174-L189) function first check whether there is liquidity that can be provided.