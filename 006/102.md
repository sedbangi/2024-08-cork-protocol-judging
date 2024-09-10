Raspy Silver Finch

High

# VaultLib.calculateTotalRaAndCtBalanceWithReserve() is used with wrong parameter

## Summary

`__calculateTotalRaAndCtBalanceWithReserve()` expects to get the total RA and CT reserves in a pair as well as the total LP supply to calculate the `raPerLv`, `ctPerLv`, `raPerLp`, `ctPerLp`, `totalRa`, `ammCtBalance`. In the both places where this function is used, the provided `lpSupply` value does not represent the total LP supply.

## Vulnerability Detail

```solidity
function __calculateTotalRaAndCtBalanceWithReserve(
        State storage self,
        uint256 raReserve,
        uint256 ctReserve,
        uint256 lpSupply
    )
        internal
        view
        returns (
            uint256 totalRa,
            uint256 ammCtBalance,
            uint256 raPerLv,
            uint256 ctPerLv,
            uint256 raPerLp,
            uint256 ctPerLp
        )
    {
        (raPerLv, ctPerLv, raPerLp, ctPerLp, totalRa, ammCtBalance) = MathHelper.calculateLvValueFromUniV2Lp(
            lpSupply, self.vault.config.lpBalance, raReserve, ctReserve, Asset(self.vault.lv._address).totalSupply()
        );
    }
```

In `VaultLib.__calculateCtBalanceWithRate()` it used as
```solidity
(,, raPerLv, ctPerLv, raPerLp, ctPerLp) = __calculateTotalRaAndCtBalanceWithReserve(
            self, raReserve, ctReserve, flashSwapRouter.getLvReserve(self.info.toId(), dsId)
        );
```

In `VaultLib.__calculateTotalRaAndCtBalance()` it is used as
```solidity
(,,,, totalRa, ammCtBalance) = __calculateTotalRaAndCtBalanceWithReserve(
            self, raReserve, ctReserve, flashSwapRouter.getLvReserve(self.info.toId(), dsId)
        );
```

`flashSwapRouter.getLvReserve(self.info.toId(), dsId)` gives us the the amount of DS tokens that are reserved in the Liquidity Vault and has nothing to do with the total LP supply.

This logic is part of `redeemEarly` and `previewRedeemExpired` functionalities.
## Impact

This issue lead to unexpected result when users redeem their shares in the LV which can result in loss of funds or complete DoS of the protocol.

## Code Snippet

`__calculateTotalRaAndCtBalance()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L425
`__calculateCtBalanceWithRate()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L438
`__calculateTotalRaAndCtBalanceWithReserve()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L450
## Tool used

Manual Review

## Recommendation

`pair.totalSupply()` can be directly used to account for the LP tokens. Internal accounting is not an option here as LP can be minted by directly providing liquidity to the pair without use of Cork Protocol.