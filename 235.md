Blurry Blush Mouse

Medium

# Rebasing tokens are not supported contrary to the readme and will lead to loss of funds

### Summary

The readme states that rebasing tokens are [supported](https://github.com/sherlock-audit/2024-08-cork-protocol?tab=readme-ov-file#q-if-you-are-integrating-tokens-are-you-allowing-only-whitelisted-tokens-to-work-with-the-codebase-or-any-complying-with-the-standard-are-they-assumed-to-have-certain-properties-eg-be-non-reentrant-are-there-any-types-of-weird-tokens-you-want-to-integrate)
> Rebasing tokens are supported with exchange rate mechanism

However, only non rebasing tokens such as the wrapped version `wsteth` are supposed. If `stETH` is used, it will accrue value in the `Psm` and `Vault` (technically they are the same contract) which will be left untracked as `Ra` and `Pa` deposits are tracked in state [variables](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/State.sol#L48-L53).

### Root Cause

The code does not handle rebasing tokens even though the readme says it does. The exchange rate mechanism only supports non rebasing tokens such as `wsteth`.

### Internal pre-conditions

None.

### External pre-conditions

None.

### Attack Path

Admin creates `steth` pairs using it as `Ra` or `Pa`, whose value will grow in the protocol but left untracked as the quantites are tracked with state variables.

### Impact

Stuck yield accruel in the Vault/Psm contracts.

### PoC

`State.sol` tracks the balances:
```solidity
struct Balances {
    PsmRedemptionAssetManager ra;
    uint256 dsBalance;
    uint256 ctBalance;
    uint256 paBalance;
}
```

### Mitigation

Don't set rebasing tokens are `Ra` or `Pa` or implement a way to sync the balances.