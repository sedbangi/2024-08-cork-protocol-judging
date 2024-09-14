Blurry Blush Mouse

Medium

# Admin will not be able to upgrade the smart contracts, breaking core functionality and rendering the upgradeable contracts useless

### Summary

The [AssetFactory](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/assets/AssetFactory.sol#L14) and [FlashSwapRouter](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L25) inherit the `UUPSUpgradeable` contract in order to be upgradeable. However, [AssetFactory::initialize()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/assets/AssetFactory.sol#L48), [FlashSwapRouter::initialize()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L32), [AssetFactory::_authorizeUpgrade()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/assets/AssetFactory.sol#L195) and [FlashSwapRouter::_authorizeUpgrade()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L41) have the `notDelegated`, which means they can not be called in the context of a proxy, hence they can not be upgradeable.

This renders the inherited `UUPSUpgradeable` useless and the 2 contracts will not be upgradeable. Additionally, the [AssetFactory](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/scripts/deploy.ts#L65-L66) and [FlashSwapRouter](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/scripts/deploy.ts#L71) contracts are not deployed behind proxies, meaning that this problem would be noticed when trying to upgrade and failing.

### Root Cause

In `AssetFactory.sol:48`, `AssetFactory.sol:195`, `FlashSwapRouter.sol:32` and `FlashSwapRouter.sol:41` the `notDelegated` modifiers are used.

### Internal pre-conditions

None.

### External pre-conditions

None.

### Attack Path

Admin tries to upgrade the `AssetFactory` and `FlashSwapRouter` contracts but fails.

### Impact

The `UUPSUpgradeable` contract is rendered useless, which means the `AssetFactory` and `FlashSwapRouter` contracts can not be upgraded. This leads to breaking major functionality as well as the possibility of stuck/lost funds.

### PoC

`FlashSwapRouter`
```solidity
contract RouterState is IDsFlashSwapUtility, IDsFlashSwapCore, OwnableUpgradeable, UUPSUpgradeable, IUniswapV2Callee {
    ...
    function initialize(address moduleCore, address _univ2Router) external initializer notDelegated {
        __Ownable_init(moduleCore);
        __UUPSUpgradeable_init();

        univ2Router = IUniswapV2Router02(_univ2Router);
    }
    ...
    function _authorizeUpgrade(address newImplementation) internal override onlyOwner notDelegated {}
}
```
`AssetFactory`

```solidity
contract AssetFactory is IAssetFactory, OwnableUpgradeable, UUPSUpgradeable {
    ...
    function initialize(address moduleCore) external initializer notDelegated {
        __Ownable_init(moduleCore);
        __UUPSUpgradeable_init();
    }
    ...
    function _authorizeUpgrade(address newImplementation) internal override onlyOwner notDelegated {}
}

```

### Mitigation

Remove the `notDelegated` modifiers.