Blurry Blush Mouse

Medium

# Malicious actor will frontrun user redeeming early permit call and DoS the user from withdrawing, being problematic whenever it's close to expiry

### Summary

[VaultLib::redeemEarly()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L639), [PsmLib::redeemRaWithCtDs()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L194) and [PsmLib::redeemWithDs()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L364) allow users to redeem `lv` and `Ra`, respectively, before expiry. These functions implement permit functionality to allow users to redeem in 1 transaction. However, the permit may be frontrun by an attacker monitoring transactions who will call the permit function itself in the `Asset` contract, DoSing the redeem call.

Both these functions are time sensitive because they can only be performed before the expiry date of the `Ds` token. This means that if the attack occurs close to the expiry date, the user would not be able to redeem the assets in this way and would have to [redeem](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L448) with `ct` or [VaultLib::redeemExpired()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L514), taking losses.

### Root Cause

In `VaultLib:651` and `PsmLib:376,207,208`, the permit may be frontrun and DoSed right before expiry.

### Internal pre-conditions

1. User calls [VaultLib::redeemEarly()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L639), [PsmLib::redeemRaWithCtDs()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L194) and [PsmLib::redeemWithDs()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L364) a few blocks before expiry.

### External pre-conditions

None.

### Attack Path

1. User calls [VaultLib::redeemEarly()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L639), [PsmLib::redeemRaWithCtDs()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L194) and [PsmLib::redeemWithDs()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L364) a few blocks before expiry.
2. Attacker frontruns and spends the permit, DoSing the functions.
3. The user would not be able to redeem with these functions, taking losses.

### Impact

User is DoSed from redeeming and takes losses.

### PoC

Check the links to confirm the permit is always called, which may revert if the attacker frontruns it.

### Mitigation

Implement a try catch in the permit such that if an attacker maliciously frontruns with a permit call, the transaction would still go through.