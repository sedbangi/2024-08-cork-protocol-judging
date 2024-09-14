Jumpy Lime Oyster

Medium

# Attacker Can Decide The Initialization Ratio Of The AMM Pair

## Summary

When the RA/CT is deposited in the AMM pool it must follow a pre defined ratio , but the attacker can see the pair has been deployed on uniV2 and make the first deposit (adding liquidity) and distrupt the intended ratio of the CT/RA .

## Vulnerability Detail

1.) When depositing , the vault decides the ratio of the RA/CT to be deposited in the AMM pool ->

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L204


```solidity
 function __provideLiquidityWithRatio(
        State storage self,
        uint256 amount,
        IDsFlashSwapCore flashSwapRouter,
        address ctAddress,
        IUniswapV2Router02 ammRouter
    ) internal returns (uint256 ra, uint256 ct) {
        uint256 dsId = self.globalAssetIdx;

        uint256 ctRatio = __getAmmCtPriceRatio(self, flashSwapRouter, dsId);

        (ra, ct) = MathHelper.calculateProvideLiquidityAmountBasedOnCtPrice(amount, ctRatio);

        __provideLiquidity(self, ra, ct, flashSwapRouter, ctAddress, ammRouter, dsId);
    }
```

```solidity
function __getAmmCtPriceRatio(State storage self, IDsFlashSwapCore flashSwapRouter, uint256 dsId)
        internal
        view
        returns (uint256 ratio)
    {
        // This basically means that if the reserve is empty, then we use the default ratio supplied at deployment
        ratio = self.ds[dsId].exchangeRate() - self.vault.initialDsPrice;

        // will always fail for the first deposit
        try flashSwapRouter.getCurrentPriceRatio(self.info.toId(), dsId) returns (uint256, uint256 _ctRatio) {
            ratio = _ctRatio;
        } catch {}
    }
```

If the pool is empty (it's the first liquidity addition) the ratio should follow the default ratio decided at development. 

2.) But an attacker can frontrun this first deposit in the vault which adds liquidity to the AMM , and manually call _addLiquidity in the UniV2 pool which calls-> 

https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router02.sol#L33

and since the pool is empty the attacker can decide the initial ratio of the pool and distrupt the ratio (can be as extreme as possible) and the amount of subsequent deposits of CT/RA in the AMM would follow this initial ratio. 

## Impact

The attacker can decide the initial ratio of the AMM pool and distrupt subsequent CT/RA deposits in the pool and incorrect DS mints, offload that initialization on the config contract so that only the config contract owner can initialize the vault.

## Code Snippet

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L204

## Tool used

Manual Review

## Recommendation

offload that initialization on the config contract so that only the config contract owner can initialize the vault.