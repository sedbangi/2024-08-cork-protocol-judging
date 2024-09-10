Raspy Silver Finch

Medium

# Possibility of DoS when issuing new DS

## Summary

Smart contracts can be created both by other contracts and by regular accounts. In both cases, the address for the new contract is computed the same way: as a function of the sender’s own address and a nonce. Every account has an associated nonce: for regular accounts it is increased on every transaction, while for contract accounts it is increased on every contract creation. Nonces cannot be reused, and they must be sequential. This means it is possible to predict the address where the _next_ created contract will be deployed

When new DS is issued via `ModuleCore.issueNewDs()`, the DS and CT token contracts are deployed using the standard `create` opcode. The address of these new contracts are deterministically generated based on `new_address = hash(sender, nonce)`. The CT contract address will be used to create a new AMM pair. If an attempt is made to create a pair for RA and CT that already exists, the transaction will revert, as a pair for these specific token addresses cannot be duplicated.
## Vulnerability Detail

Since Ethereum mainnet has a public mempool, an attacker can monitor when the protocol admin is about to issue a new DS. By calculating the addresses of the newly created CT and DS tokens (which are deterministic based on the `sender` and `nonce`), the attacker can front-run the transaction and create the AMM pair for these tokens before the protocol does. This would cause the line `address ammPair = getAmmFactory().createPair(ra, ct);` to revert, as the pair already exists.

To counter this, the protocol would need to increment its nonce by calling `ModuleCore.initialize()`, which deploys a new Liquidity Vault to change the nonce. However, the attacker can repeat this process indefinitely, creating the pair before the protocol does each time. Since the cost of issuing a new vault and creating a DS is higher than the attacker’s gas cost to create a new pair, the attacker could exploit this repeatedly.

Another scenario involves the attacker preemptively deploying a large number of AMM pairs with potential future DS and CT addresses (based on predicted nonces). This would force the protocol to spend even more gas incrementing the nonce multiple times by initializing new vaults until it reaches a nonce that hasn't been used by the attacker. This significantly increases the gas costs and complexity for the protocol to issue new DS tokens.

`issueNewDs()` uses to AssetFactory's `deploysSwapAssets()`
```solidity
function issueNewDs(Id id, uint256 expiry, uint256 exchangeRates, uint256 repurchaseFeePrecentage)
        external
        override
        onlyConfig
        onlyInitialized(id)
    {
        if (repurchaseFeePrecentage > 5 ether) {
            revert InvalidFees();
        }
        State storage state = states[id];

        address ra = state.info.pair1;

        uint256 prevIdx = state.globalAssetIdx++;
        uint256 idx = state.globalAssetIdx;

        (address ct, address ds) = IAssetFactory(SWAP_ASSET_FACTORY).deploySwapAssets(
            ra, state.info.pair0, address(this), expiry, exchangeRates
        );

		// This would fail if there is already created pair for the RA and CT addresses
        address ammPair = getAmmFactory().createPair(ra, ct);

        PsmLibrary.onNewIssuance(state, ct, ds, ammPair, idx, prevIdx, repurchaseFeePrecentage);

        getRouterCore().onNewIssuance(id, idx, ds, ammPair, 0, ra, ct);

        VaultLibrary.onNewIssuance(state, prevIdx, getRouterCore(), getAmmRouter());

        emit Issued(id, idx, expiry, ds, ct, ammPair);
    }


```

```solidity
function deploySwapAssets(address ra, address pa, address owner, uint256 expiry, uint256 psmExchangeRate)
        external
        override
        onlyOwner
        notDelegated
        returns (address ct, address ds)
    {
        Pair memory asset = Pair(pa, ra);

        // prevent deploying a swap asset of a non existent pair, logically won't ever happen
        // just to be safe
        if (lvs[asset.toId()] == address(0)) {
            revert NotExist(ra, pa);
        }

        string memory pairname = string(abi.encodePacked(Asset(ra).name(), "-", Asset(pa).name()));

        ct = address(new Asset(CT_PREFIX, pairname, owner, expiry, psmExchangeRate));
        ds = address(new Asset(DS_PREFIX, pairname, owner, expiry, psmExchangeRate));

        swapAssets[Pair(pa, ra).toId()].push(Pair(ct, ds));

        deployed[ct] = true;
        deployed[ds] = true;

        emit AssetDeployed(ra, ct, ds);
    }
```
## Impact

This can lead to complete DoS for the system and lock of funds for users who have decided to not redeem their rewards in the expired DS period. As there is no direct incentive for the attacker, the likelihood of this happening is Low, but the impact would be high because of the consequences described in the previous sentence, making the overall severity medium.
## Code Snippet

`ModuleCore.initialNewDS()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/core/ModuleCore.sol#L57
`AssetFactory.deploySwapAssets()`
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/core/assets/AssetFactory.sol#L140
`AssetFactory.deployLv()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/core/assets/AssetFactory.sol#L175
## Tool used

Manual Review
## Recommendation

Consider this scenario that pair may have already been created and use `try/catch` when trying to deploy new pair.