Proper Turquoise Goldfish

Medium

# Missing Vault expiry check in `transferRedemptionRights()`

### Summary

In the `Vault.sol` contract, the `transferRedemptionRights()` function allows users to transfer their redemption rights to another address.

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/Vault.sol#L90
```solidity
  
    /**
     * @notice Transfer redemption rights of a given vault at expiry
     * @param id The Module id that is used to reference both psm and lv of a given pair
     * @param to The address of the new owner of the redemption rights
     * @param amount The amount of the user locked LV token to be transferred
     */
    function transferRedemptionRights(Id id, address to, uint256 amount) external override {
        State storage state = states[id];
        state.transferRedemptionRights(_msgSender(), to, amount); 
        emit RedemptionRightTransferred(id, _msgSender(), to, amount);
    }
```

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L218-L232
```solidity
    function transferRedemptionRights(State storage self, address from, address to, uint256 amount) external {
        uint256 initialOwneramount = self.vault.pool.withdrawEligible[from];

        if (initialOwneramount == 0) {
            revert Unauthorized(msg.sender);
        }

        if (initialOwneramount < amount) {
            revert InsufficientBalance(from, amount, initialOwneramount);
        }

        self.vault.pool.withdrawEligible[to] += amount;
        self.vault.pool.withdrawEligible[from] -= amount;
    }
```

However, there is **no check** to ensure that the vault has **expired** before allowing the transfer, which contradicts the function's NatSpec documentation.

>Transfer redemption rights of a given vault **at expiry**

The NatSpec states that the function transfers redemption rights "at expiry," but in practice, this function is callable at any time, including before the vault expires.

There is a possibility that the protocol had in mind that there should be a check not if it expired but before it expired. Because that's the logic in `requestRedemption()`. But in both cases, there is no such check.

### Root Cause

**Missing vault expiry check** in the `transferRedemptionRights()` function, which contradicts the function’s intended behavior as described in the documentation.

### Internal pre-conditions

No

### External pre-conditions

No

### Attack Path

- A user with locked LV tokens and redemption rights calls `transferRedemptionRights()` before the vault has expired.
- Since no expiry check is implemented, the transfer is allowed even though the NatSpec documentation suggests this should only happen **after** the vault has expired.
- The recipient of the redemption rights can now control them prematurely, potentially disrupting the intended redemption process or liquidity allocation of the vault.

### Impact

Users can transfer their redemption before/after(depending on the protocol what they had in mind) the vault has expired, which contradicts the intended redemption process. In the NatSpec documentation it is written that there should be such a check.


### PoC

_No response_

### Mitigation

Introduce a vault expiry check in the `transferRedemptionRights()` function.