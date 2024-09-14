Trendy Tin Turtle

Medium

# Unaccounted Minting of `ds` Tokens Leads to Locked Funds and Inaccessible User Deposits


## Summary

The vulnerability arises from the improper minting and handling of `ds` tokens during the deposit process in the protocol. The `ds` tokens are minted to the `moduleCore` address without being accounted for in any variable or given to the depositor, as stated in the protocol's documentation. This results in a situation where the `ds` tokens are effectively locked within the protocol, causing potential loss or inaccessibility of user funds.

## Vulnerability Detail

The execution flow that leads to the vulnerability is as follows:
1. **`Vault:depositLv()`** ➔ **`VaultLib:deposit()`** ➔ **`__provideLiquidityWithRatio()`** ➔ **`__provideLiquidity()`**

Within the `VaultLib:__provideLiquidity()` function, a call is made to the `PsmLibrary`'s `unsafeIssueToLv()` function:
```solidity
PsmLibrary.unsafeIssueToLv(self, ctAmount);
```

### `PsmLibrary:unsafeIssueToLv()`

Inside the `unsafeIssueToLv()` function, a call is made to the `DepegSwapLib`'s `issue()` function:
```solidity
ds.issue(address(this), amount);
```

### Core Issue

The `issue()` function mints both `ds` and `ct` tokens to the `moduleCore` address. However, the minting of `ds` tokens is **not accounted for in any variable** within the protocol. According to the protocol's documentation, `ds` tokens are supposed to be provided to the depositor when a user makes a deposit, but:
- **The `ds` tokens are not given to the user**.
- **They are minted to the `moduleCore` address and not properly tracked or accounted for**.

This results in `ds` tokens being locked in the protocol without being accessible or useful to the intended user. The tokens remain in the `moduleCore` address without serving their purpose, effectively leading to their loss or indefinite inaccessibility.

## Impact

- **Loss of User Funds:** Users depositing into the protocol expect to receive `ds` tokens representing their deposit, but due to this flaw, they do not receive these tokens.
- **Locked Tokens:** A significant amount of `ds` tokens are locked within the protocol and cannot be accessed or used, leading to an unintended accumulation of value that is neither useful to the protocol nor its users.
- **Breaks Protocol Functionality:** The intended flow and accounting for deposits are broken, leading to potential inconsistencies, financial loss, or lack of trust in the protocol.

## Code Snippet
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L115-L123

## Tool Used

- **Manual Review**

## Recommendation

1. **Proper Accounting:** Ensure that the `ds` tokens minted during the deposit process are correctly accounted for and transferred to the appropriate recipient, usually the depositor.
