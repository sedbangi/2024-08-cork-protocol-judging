Trendy Tin Turtle

Medium

# Permit functions are allowed to be called by the permit creator only.

## Summary
Functions using permit to deposit eg `swapRaforDs` are allowed to be called by the permit creator only. Not any other contracts will be able to execute these function on behalf of signer.

## Vulnerability Detail
The  function's using permit are allowed to provide a signed permit in order to receive, approve and deposit funds.
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/Psm.sol#L123-L140
it passes `mgs.sender` as the `owner`  while calling  `state:redeemWithDs`, and   then inside `state:redeemWithDs` it calls `DepegSwapLibrary:permit` and pass `msg.sender` as `owner` to that function.

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/Psm.sol#L134-L135

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L375-L377

This means, that the signer can be only the same person that called the function.

However, the purpose of the permit is to allow someone to sign the approve signature, so that this signature can be used by another contract to call some function on behalf of signer.

In this case, anyone should be able to sign permit for the vault, and vault should check that _receiver is same who signed permit.

There are many functions which are using `msg.sender` for `permit` 


## Code Snippet
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L375-L377
## Tool used

Manual Review

## Recommendation

Use `_receiver` instead of `msg.sender`.

