Trendy Tin Turtle

Medium

# Permit Will not work for DAI

## Summary
Interaction of the protocol with the DAI `permit` function will fail.

## Vulnerability Detail
The protocol intends to interact with the DAI token, which is used as a redemption asset, as mentioned in their [documentation](https://corkfi.notion.site/Cork-Protocol-Litepaper-f21a57d5c19d48209dfa0f0c2ab776c4).

In the `FlashSwapRouter:swapRaforDs` function, the protocol attempts to use the `permit` functionality to approve token transfers. Although the function checks whether the token supports `permit` or not, and DAI does support `permit`, the function signature for DAI's `permit` is different from the standard `ERC20Permit` function signature. This mismatch will lead to transaction failures.

The `ERC20Permit` function signature is:
```solidity
function permit(
    address owner,
    address spender,
    uint256 value,
    uint256 deadline,
    uint8 v,
    bytes32 r,
    bytes32 s
) public virtual
```

However, the `permit` function for DAI, as seen in its contract at address `0x6B175474E89094C44Da98b954EedeAC495271d0F`, is:
```solidity
function permit(
    address holder, 
    address spender, 
    uint256 nonce, 
    uint256 expiry,
    bool allowed, 
    uint8 v, 
    bytes32 r, 
    bytes32 s
) external
```

Due to the different parameters (e.g., `nonce` and `allowed` in DAI's `permit`), any interaction using the `ERC20Permit` signature will fail for DAI. This is because DAI requires a nonce and a boolean `allowed` field which are missing in the `ERC20Permit` signature. As a result, attempts to interact with DAI using the incorrect signature will cause the transactions to revert.

## Impact
Any transactions involving the `permit` function with DAI as a redemption asset will fail. This could prevent users from completing swaps or other interactions within the protocol, reducing its usability and reliability.

## Code Snippet
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L150-L170
```solidity
// Example usage of permit in FlashSwapRouter:swapRaforDs
// The function attempts to use permit as defined by ERC20Permit
function permit(
    address owner,
    address spender,
    uint256 value,
    uint256 deadline,
    uint8 v,
    bytes32 r,
    bytes32 s
) public virtual;
```

## Tool Used
Manual Review

## Recommendation
Update the protocol to detect if the DAI token is being used and handle the `permit` function call accordingly by using the correct signature specific to DAI. Alternatively, avoid using the `permit` functionality for DAI and rely on a different method for approving token transfers.
