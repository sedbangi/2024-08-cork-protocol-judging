# Issue H-1: Lack of slippage protection leads to loss of protocol funds 

Source: https://github.com/sherlock-audit/2024-08-cork-protocol-judging/issues/66 

## Found by 
0x6a70, 0x73696d616f, 0xjoi, 0xnbvc, 4b, 4gontuk, Abhan1041, MohammedRizwan, Pheonix, alphacipher, boringslav, ivanonchain, sakshamguruji, vinica\_boy
## Summary

There is no slippage protection while removing liquidity and swap tokens from AMM.

## Vulnerability Detail

There are 2 intances where slippage protection is missing which are as below:

1. When LV token holder redeem before expiry `vaultLib::redeemEarly` function is called in which `_liquidateLpPartial` function and in that  `_redeemCtDsAndSellExcessCt` is called. In `_redeemCtDsAndSellExcessCt` function CT tokens are swapped for RA tokens in AMM as below:

```solidity
...
ra += ammRouter.swapExactTokensForTokens(ctSellAmount, 0, path, address(this), block.timestamp)[1];
...
```
As stated above, `swapExactTokensForTokens` function's 2nd parameter is 0 which shows that there is no slippage protection for this 
swap and also deadline is block.timestamp.

2. In `vaultLib::_liquidateLpPartial` function `__liquidateUnchecked` is called in which liquidity is removed from AMM of RA-CT token pair by burning LP tokens of protocol as below:

```solidity
...
(raReceived, ctReceived) =
            ammRouter.removeLiquidity(raAddress, ctAddress, lp, 0, 0, address(this), block.timestamp);
...
```
As stated above, `removeLiquidity` function's 4th & 5th parameter is 0 which shows that there is no slippage protection for this swap and also deadline is block.timestamp.

In such cases, an attacker can frontrun the transaction by seeing it in the mempool and manipulate the price such that protocol transaction have to bear heavy slippage which will leads to loss of protocol funds.

Also, there is block.timestamp as deadline so malicious node can prevent transaction to execute temporary and  execute the transaction when there is high slippage which will also leads to loss of protocol funds.

## Impact

Loss of protocol funds which will reduce the yield of users.

## Code Snippet

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L282

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L345

## Tool used

Manual Review

## Recommendation

Protocol should implement slippage protection and set deadline while removing liquidity and also swap from AMM.



## Discussion

**ziankork**

Yes this is a valid issue, we've already fixed this prior to our trading competition except for the `deadline` vulnerability.

# Issue H-2: FlashSwapRouter::emptyReserve()  and FlashSwapROuter::emptyReservePartial() functions return incorrect values 

Source: https://github.com/sherlock-audit/2024-08-cork-protocol-judging/issues/68 

## Found by 
0x73696d616f, 0xNirix, KupiaSec, dimulski, nikhil840096, sakshamguruji, vinica\_boy
### Summary

The protocol deposits RA and CT tokens to an AMM pair, from fees or when users call the [depositLv()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/Vault.sol#L33-L37) function. The CT and DS tokens issued by the protocol have an expiration, after the first DS and CT tokens for a pair have been issued and expired, each next time the protocol tries to issue new DS and CT tokens for an existing pair of RA and PA tokens via calling the [issueNewDs()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/ModuleCore.sol#L57-L86) function, the [VaultLib::onNewIssuance()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L92-L108) function will be called. The [VaultLib::onNewIssuance()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L92-L108) function will then call the [VaultLib::_liquidatedLp()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L349-L393) function, which internally calls the [FlashSwapRouter::emptyReserve()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L69-L72) function which will empty the whole reserve, and then return 0. 
```solidity
    function emptyReserve(ReserveState storage self, uint256 dsId, address to) internal returns (uint256 reserve) {
        reserve = emptyReservePartial(self, dsId, self.ds[dsId].reserve, to);
    }

    function emptyReservePartial(ReserveState storage self, uint256 dsId, uint256 amount, address to)
        internal
        returns (uint256 reserve)
    {
        self.ds[dsId].ds.transfer(to, amount);

        self.ds[dsId].reserve -= amount;
        reserve = self.ds[dsId].reserve;
    }
```
When we go back to the [VaultLib::_liquidatedLp()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L349-L393) function
```solidity
    function _liquidatedLp(
        State storage self,
        uint256 dsId,
        IUniswapV2Router02 ammRouter,
        IDsFlashSwapCore flashSwapRouter
    ) internal {
        ...
         // the following things should happen here(taken directly from the whitepaper) :
        // 1. The AMM LP is redeemed to receive CT + RA
        // 2. Any excess DS in the LV is paired with CT to redeem RA
        // 3. The excess CT is used to claim RA + PA in the PSM
        // 4. End state: Only RA + redeemed PA remains
        uint256 reservedDs = flashSwapRouter.emptyReserve(self.info.toId(), dsId);

        uint256 redeemAmount = reservedDs >= ctAmm ? ctAmm : reservedDs;
        PsmLibrary.lvRedeemRaWithCtDs(self, redeemAmount, dsId);

        // if the reserved DS is more than the CT that's available from liquidating the AMM LP
        // then there's no CT we can use to effectively redeem RA + PA from the PSM
        uint256 ctAttributedToPa = reservedDs >= ctAmm ? 0 : ctAmm - reservedDs;

        uint256 psmPa;
        uint256 psmRa;

        if (ctAttributedToPa != 0) {
            (psmPa, psmRa) = PsmLibrary.lvRedeemRaPaWithCt(self, ctAttributedToPa, dsId);
        }

        psmRa += redeemAmount;

        self.vault.pool.reserve(self.vault.lv.totalIssued(), raAmm + psmRa, psmPa);
    }
```
As can be seen from the 2 comment in the function any excess CT and DS tokens should be paired and redeemed for RA, however since the [FlashSwapRouter::emptyReserve()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L69-L72) function will always return 0, so the [PsmLib::lvRedeemRaWithCtDs()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L125-L128) function will always redeem 0 RA tokens and not burn the CT and DS tokens. As we can see from the above code snippet we will go directly to [PsmLib::lvRedeemRaPaWithCt()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L130-L143) function, which will try to redeem RA + PA tokens, with all of the CT tokens that were returned from the UniV2 pair when the LP tokens of the protocol were liquidated. 

 The second case where a problem occurs is when a user tries to redeem his LV tokens by calling the [redeemEarlyLv()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/Vault.sol#L187-L198) function which internally calls the [VaultLib::redeemEarly()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L639-L665) function and after a couple of other internal calls the [VaultLib::_redeemCtDsAndSellExcessCt()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L318-L347) function is called where the [FlashSwapRouter::emptyReservePartial()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L74-L82) function is called:
```solidity
    function _redeemCtDsAndSellExcessCt(
        State storage self,
        uint256 dsId,
        IUniswapV2Router02 ammRouter,
        IDsFlashSwapCore flashSwapRouter,
        uint256 ammCtBalance
    ) internal returns (uint256 ra) {
        uint256 reservedDs = flashSwapRouter.getLvReserve(self.info.toId(), dsId);

        uint256 redeemAmount = reservedDs >= ammCtBalance ? ammCtBalance : reservedDs;

        reservedDs = flashSwapRouter.emptyReservePartial(self.info.toId(), dsId, redeemAmount);

        ra += redeemAmount;
        PsmLibrary.lvRedeemRaWithCtDs(self, redeemAmount, dsId);

        uint256 ctSellAmount = reservedDs >= ammCtBalance ? 0 : ammCtBalance - reservedDs;

        DepegSwap storage ds = self.ds[dsId];
        address[] memory path = new address[](2);
        path[0] = ds.ct;
        path[1] = self.info.pair1;

        ERC20(ds.ct).approve(address(ammRouter), ctSellAmount);

        if (ctSellAmount != 0) {
            // 100% tolerance, to ensure this not fail
            ra += ammRouter.swapExactTokensForTokens(ctSellAmount, 0, path, address(this), block.timestamp)[1];
        }
    }
```
When the last LV tokens are being redeemed the **reservedDs** will be equal or very close to **ammCtBalance**, and when the  [FlashSwapRouter::emptyReservePartial()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L74-L82) function is called, it will return the DS reserve after the redeemAmount has been subtracted, which will be either 0, or much less than **redeemAmount**. For this example consider it is 0. When the **ctSellAmount** is calculated it will be much bigger than the actual reserves of CT token in the contract, and when the function tries to transfer the CT tokens to the AMM in order to swap them for RA tokens, the call will revert, and the user redeeming his LV token won't be able to redeem it and receive RA tokens back, thus locking funds in the contract. 

### Root Cause

The root cause is that the [FlashSwapRouter::emptyReserve()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L69-L72) and [FlashSwapROuter::emptyReservePartial()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L74-L82) functions returns the reserve that is left after the redeemAmount has been subtracted. 

### Internal pre-conditions

1. Users mint LV tokens via the [depositLv()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/Vault.sol#L33-L37) function
2. There are a couple of LV tokens that haven't been redeemed yet, and a user decides to redeem them by calling the [VaultLib::redeemEarly()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L639-L665) function

### External pre-conditions

_No response_

### Attack Path

_No response_

### Impact

When it comes to [FlashSwapRouter::emptyReserve()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L69-L72), instead of the excess DS in the LV being paired with CT to redeem RA, all of the CT returned from the liquidation of LP will be used to claim RA + PA in the PSM, this is contrary of what is expected from the function according to the docs, and the comments, and may result in [VaultLib::_liquidatedLp()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L349-L393) function claiming much more PA tokens than it should, and distributing them to LV holders. In the case of [FlashSwapROuter::emptyReservePartial()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L74-L82), the last users to withdraw won't be able to do so.  The last user that tries to redeem his LV tokens won't be able to do so, and he won't receive his RA tokens back, locking the RA tokens in the contract.

### PoC

[Gist](https://gist.github.com/AtanasDimulski/3f9bfc84c63e1c977b877613b644c0e2)

After following the steps in the above mentioned [gist](https://gist.github.com/AtanasDimulski/3f9bfc84c63e1c977b877613b644c0e2) add the following test to the ``AuditorTests.t.sol`` contract:
```solidity
    function test_IncorrectEmptyReserveReturnedValue() public {
        vm.startPrank(alice);
        WETH.mint(alice, 10e18);
        WETH.approve(address(moduleCore), type(uint256).max);
        moduleCore.depositLv(id, 10e18);
        Asset(lvAddress).approve(address(moduleCore), type(uint256).max);
        vm.expectRevert(bytes("TransferHelper::transferFrom: transferFrom failed"));
        moduleCore.redeemEarlyLv(id, alice, 10e18);
        vm.stopPrank();
    }
```

To run the test use: ``forge test -vvv --mt test_IncorrectEmptyReserveReturnedValue``

### Mitigation

A lot of things have to be considered when fixing this problems, simply returning the amount that was redeemed may introduce other problems. Returning the amount that was redeemed seems to be okay when it comes to the [FlashSwapRouter::emptyReserve()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L69-L72) function.



## Discussion

**ziankork**

this is a valid issue, it should return the how much amount is emptied, we did fix this prior to our trading competition and will provide links later

# Issue H-3: Lack of Slippage Protection for Reserve during swaps 

Source: https://github.com/sherlock-audit/2024-08-cork-protocol-judging/issues/97 

## Found by 
StraawHaat
### Summary

The `swapRaforDs()` function allows users to swap Redemption Asset (RA) for Depeg Swap (DS) tokens:

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L98-L138
`swapRaforDs()` -> `_swapRaforDs()`:

```solidity
    function _swapRaforDs(
        ReserveState storage self,
        AssetPair storage assetPair,
        Id reserveId,
        uint256 dsId,
        uint256 amount,
        uint256 amountOutMin
    ) internal returns (uint256 amountOut) {
        uint256 borrowedAmount;

        // calculate the amount of DS tokens attributed
        (amountOut, borrowedAmount,) = assetPair.getAmountOutBuyDS(amount);

        // calculate the amount of DS tokens that will be sold from LV reserve
        uint256 amountSellFromReserve =
            amountOut - MathHelper.calculatePrecentageFee(self.reserveSellPressurePrecentage, amountOut);
        // sell all tokens if the sell amount is higher than the available reserve
        amountSellFromReserve = assetPair.reserve < amountSellFromReserve ? assetPair.reserve : amountSellFromReserve;

        // sell the DS tokens from the reserve if there's any
        if (amountSellFromReserve != 0) {
            // decrement reserve
            assetPair.reserve -= amountSellFromReserve;

            // sell the DS tokens from the reserve and accrue value to LV holders
            uint256 vaultRa = __swapDsforRa(assetPair, reserveId, dsId, amountSellFromReserve, 0); //@audit - no slippage
            IVault(owner()).provideLiquidityWithFlashSwapFee(reserveId, vaultRa);

  
            // recalculate the amount of DS tokens attributed, since we sold some from the reserve
            (amountOut, borrowedAmount,) = assetPair.getAmountOutBuyDS(amount);
        }

        if (amountOut < amountOutMin) {
            revert InsufficientOutputAmount();
        }

        // trigger flash swaps and send the attributed DS tokens to the user

        __flashSwap(assetPair, assetPair.pair, borrowedAmount, 0, dsId, reserveId, true, amountOut);
    }
```

If a user wants to have slippage protection, he can set it as a parameter, and at the end of the function, we have a check.

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L124

```solidity
        if (amountOut < amountOutMin) {
            revert InsufficientOutputAmount();
        }
```

However, when the protocol executes the swap for its **reserve**, there is no slippage protection in place, exposing the protocol to unfavorable market conditions. Specifically, the reserve sells **DS** tokens without ensuring a minimum acceptable amount of **RA** in return, potentially resulting in significant financial losses for the protocol.

```solidity
uint256 vaultRa = __swapDsforRa(assetPair, reserveId, dsId, amountSellFromReserve, 0);
```

In this line, the reserve executes the swap without any slippage protection (the slippage is set to `0`), meaning the reserve will accept any amount of **RA** in return for the **DS** tokens it sells. This makes the protocol vulnerable to market volatility or manipulation.

There are three variants here:

1. A regular user sets a slippage for him, but in the swap protocol loses tokens because there is no slippage protection for the reserve. 
2. A regular user decides he doesn't need slippage protection and leaves it at 0. The protocol takes a huge loss because of this user decision.
3. Self-sandwich Attack. I will explain in more detail in the Attack Path. 

### Root Cause

The lack of slippage protection when the reserve sells its **DS** tokens in the `_swapRaforDs()` function:

```solidity
uint256 vaultRa = __swapDsforRa(assetPair, reserveId, dsId, amountSellFromReserve, 0);
```



### Internal pre-conditions

Мalicious or normal users need to set `amountOutMin` to 0. In the Attack Path section, I have explained both variants. 
- If a normal user calls `swapRaforDs()` with slippage protection, the loss to the protocol is minimal - Low Severity.
- If a normal user calls `swapRaforDs()` without slippage protection, the loss to the protocol is significant - High Severity.
- Self-sandwich Attack - High Severity

### External pre-conditions

No external pre-conditions

### Attack Path

#### Normal Attack Path
- - A user calls the `swapRaforDs()` function to swap RA for DS tokens and does not want slippage protection for **himself**.
- The function internally calls the `_swapRaforDs()` function.
- Inside `_swapRaforDs()`, the protocol attempts to sell DS tokens from the reserve using the following code:
```solidity
uint256 vaultRa = __swapDsforRa(assetPair, reserveId, dsId, amountSellFromReserve, 0);
```
- The **reserve** is selling DS tokens with **0 slippage protection**.
- The **reserve** could suffer significant losses
***
#### Self-sandwich Attack Path

In this scenario, the attacker uses two accounts (let's call them **Account A** and **Account B**) to manipulate the protocol’s lack of slippage protection for personal profit.

- The attacker controls two accounts: **Account A** and **Account B**.
- **Account A** will make a trade that shifts the market price. 
- **Account B** will execute the swap using `swapRaforDs()` with **zero slippage** protection to exploit the price difference caused by **Account A**.
- **Account A** places a transaction that manipulates the price of **DS** tokens, for example, by buying a large amount of **DS** or swapping another token to increase **DS** price or decrease liquidity. This transaction front-runs **Account B’s** swap, ensuring that when **Account B**'s transaction executes, it does so at a manipulated and unfavorable price.
- Since there is no slippage protection, the protocol will complete the trade at a much worse price than it should have, causing the reserve to lose value.
- After **Account B** has executed its swap and the protocol has lost value, **Account A** back-runs the transaction by reversing the initial trade or taking advantage of the new price levels.
- **Account A** sells **DS** tokens at the inflated price, securing a profit.

### Impact

The **protocol** suffers losses when the **reserve** sells **DS** tokens without any slippage protection. In case of market volatility or manipulation, the protocol will receive less **RA** in return than the actual value of the **DS** tokens being sold.

### PoC

_No response_

### Mitigation

Introduce Slippage Protection for the Reserve. If a user decides they don't need slippage protection, then you can set a standard base percentage to protect the protocol.



## Discussion

**ziankork**

The slippage protection for user is implemented(will provide the link later), and we need to fix the base slippage protection for the protocol(may also make sense to cancel the trade from the protocol if's below slippage protection)

# Issue H-4: Incorrect redeemAmount Is Accounted Due To Not Accounting For The  Exchange Rate 

Source: https://github.com/sherlock-audit/2024-08-cork-protocol-judging/issues/119 

## Found by 
0x73696d616f, 0xNirix, Ace-30, KupiaSec, Pheonix, korok, oxelmiguel, sakshamguruji, vinica\_boy
## Summary

When Liquidating LP , DS and CT are paired , then that amount is used to redeem RA . But the accounting for RA has been done incorrectly since it does not account for exchange rate.

## Vulnerability Detail

1.) Inside Liquidate LP we empty out the DS reserve and pair it up with the CT amount returned from the AMM ->

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L376

This is the amount of CtDs being redeemed.

2.) This same amount has been accounted for the increment in RA ->

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L390

But this is incorrect , this is because `redeemAmount` is an amount in Ct/Ds not in RA , to make it into RA we need to apply the exchange rate over it(exchange rate is how many Redemption Assets you need to deposit to receive 1 Cover Token + 1 Depeg Swap and how many Redemption Assets you receive when redeeming 1 Pegged Asset + 1 Depeg Swap , read more in the Dealing with non-rebasing Pegged Assets section -> https://corkfi.notion.site/Cork-Protocol-Litepaper-f21a57d5c19d48209dfa0f0c2ab776c4).

3.) Therefore incorrect RA amount has been accounted and incorrect amount of RA would be reserved ->

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L392

meaning , incorrect amount of RA attributed to be withdrawn/redeemed.

4.) This inconsistency is found at multiple places , and instead of making separate reports  im listing them here ->

a.) https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L122

The amount here is in RA and we are issuing DS/CT with it.

b.) https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L368

dsAttributed is in DS and we are depositing RA.

c.) https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L331

redeemAmount is in Ct/Ds here
## Impact

## Code Snippet

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L390

## Tool used

Manual Review

## Recommendation

Account for the exchange rate correctly.



## Discussion

**ziankork**

This is valid, will fix

# Issue H-5: Incoming Redemption Assets not being tracked when repurchase is called 

Source: https://github.com/sherlock-audit/2024-08-cork-protocol-judging/issues/126 

## Found by 
0x73696d616f, 0xNirix, Ace-30, KupiaSec, Pheonix, dimulski, sakshamguruji, steadyman, vinica\_boy
## Summary
``repurchase()`` function take redemption asset and gives back depeg swap along with pegged. However, the incoming redemption asset is not being tracked via ``lockFrom`` but there's a direct transfer of ra via ``lockUnchecked()`` causing mismatch in ra accounting. 

## Vulnerability Detail
Users deposit redemption asset via deposit to get back Cover Token + DepegSwap which can be redeemed back using various combinations.
1. Redeem with CoverToken + DepegSwap ( only before expiry of DS )
2. Redeem with DepegSwap + Pegged Asset ( only before expiry of DS ) 
3. Redeem with CoverToken ( only after expiry ) 

Is users use 2. as a means to redeem deposited RA, they can repurchase DepegSwap + Pegged Asset  back using ``repurchase()``
The ``repurchase()`` function is designed for getting back a combination of DepegSwap + Pegged Asset by giving Redemption Asset.

Incoming and outgoing RA is tracked via PsmRedemptionAssetManager struct attached with State struct. 
Consider the system before expiry. Let's look at all the places where RA can come and go and how internal tracking is changing respectively. 
1. deposit() : ra increased
2. redeem with DS + PA : ra decreased
3. redeem with CT + DS : ra decreased
Everytime there is incoming of ra, the contracts increases internal state variable ( ``locked``)  with that amount. 
```Solidity
struct PsmRedemptionAssetManager {
    address _address;
    uint256 locked;
    uint256 free;
}
```
But this is not increased  when ra comes with ``repurchase()`` . 
This can be problematic after time of expiry. 
After expiry, when users redeems with ct by calling ``redeemWithCt()`` it internally invokes ``_separateLiquidity()``. 
```Solidity
    function _separateLiquidity(State storage self, uint256 prevIdx) internal {
        if (self.psm.liquiditySeparated.get(prevIdx)) {
            return;
        }


        DepegSwap storage ds = self.ds[prevIdx];
        Guard.safeAfterExpired(ds);


        uint256 availableRa = self.psm.balances.ra.convertAllToFree();
        uint256 availablePa = self.psm.balances.paBalance;


        self.psm.poolArchive[prevIdx] = PsmPoolArchive(availableRa, availablePa, IERC20(ds.ct).totalSupply());


        // reset current balances
        self.psm.balances.ra.reset();
        self.psm.balances.paBalance = 0;


        self.psm.liquiditySeparated.set(prevIdx);
    }
```
In this function all the accumulated ra goes to pool archive. This is achieved by zeroing out the variable that was tracking incoming and outgoing ra ( by ``reset()``)  and storing it in PoolArchive. 

But since repurchase does not change this internal variable, at the time of  ``_separateLiquidity()`` , actual ra in the system will be much more than what is expected. 

## Impact
Since protocol's internal tracking assumes less ra in the system than the actual amount, the remaining amount will be stuck in the contract causing direct loss of funds.

## Code Snippet
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L293-L322


## Tool used

Manual Review

## Recommendation
Increase locked ra amount whenever repurchase is called by calling ``lockFrom`` instead of ``lockUnchecked()``



## Discussion

**ziankork**

this is valid, although we already fixed this for the trading competition. will provide links later

# Issue H-6: Users will steal excess funds from the Vault due to `VaultPoolLib::redeem()` not always decreasing `self.withdrawalPool.raBalance` and `self.withdrawalPool.paBalance` 

Source: https://github.com/sherlock-audit/2024-08-cork-protocol-judging/issues/144 

The protocol has acknowledged this issue.

## Found by 
0x73696d616f, sakshamguruji
### Summary

[Vault::redeemExpiredLv()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/Vault.sol#L128) calls [VaultLib::redeemExpired()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L514), which allows users to withdraw funds after expiry, even if they have not requested a redemption. This redemption happens in [VaultPoolLib::redeem()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L549), when [userEligible < amount](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L83), it calls internally [__redeemExcessFromAmmPool()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L89), where only [self.ammLiquidityPool.balance](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L167) is reduced, but not `self.withdrawalPool.raBalance` and `self.withdrawalPool.paBalance`. As such, when calculing the withdrawal pool balance in the next issuance on [VaultPoolLibrary::reserve()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L12), it will double count all the already [withdraw self.withdrawalPool.raBalance](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L17) and [self.withdrawalPool.paBalance](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L26), allowing users to withdraw the same funds twice.

### Root Cause

In `VaultPoolLib::__redeemExcessFromAmmPool()`, `self.withdrawalPool.raBalance` and `self.withdrawalPool.paBalance` are not decreased, but `ra` and `pa` are also withdrawn from the withdrawal pool when the user has partially requested redemption.

### Internal pre-conditions

1. User requests redemption of an amount smaller than the total withdrawn in `VaultLib::redeemExpired()`, that is, `userEligible < amount`.

### External pre-conditions

None.

### Attack Path

1. User calls `Vault::redeemExpiredLv()`, withdrawing from the withdrawal pool, but `self.withdrawalPool.raBalance` and `self.withdrawalPool.paBalance` are not decreased.
2. A new issuance starts, and in `VaultPoolLib::reserve()`, the funds are double counted as not all withdrawals were reduced.
3. As such, `self.withdrawalPool.raExchangeRate` and `self.withdrawalPool.paExchangeRate` will be inflated by double the funds and users will redeem more funds than they should, leading to the insolvency of the Vault.

### Impact

Users steal funds while unaware users will not be able to withdraw.

### PoC

`__tryRedeemExcessFromAmmPool()` does not decrease the withdrawn `self.withdrawalPool.raBalance` and `self.withdrawalPool.paBalance`.
```solidity
function __tryRedeemExcessFromAmmPool(VaultPool storage self, uint256 amountAttributed, uint256 excessAmount)
    internal
    view
    returns (uint256 ra, uint256 pa, uint256 withdrawnFromAmm)
{
    (ra, pa) = __tryRedeemfromWithdrawalPool(self, amountAttributed);

    withdrawnFromAmm =
        MathHelper.calculateRedeemAmountWithExchangeRate(excessAmount, self.withdrawalPool.raExchangeRate); //@audit PA is ignored here

    ra += withdrawnFromAmm;
}
```

### Mitigation

Replace `__tryRedeemfromWithdrawalPool()` with `__redeemfromWithdrawalPool()`.
```solidity
function __tryRedeemExcessFromAmmPool(VaultPool storage self, uint256 amountAttributed, uint256 excessAmount)
    internal
    view
    returns (uint256 ra, uint256 pa, uint256 withdrawnFromAmm)
{
    (ra, pa) = __redeemfromWithdrawalPool(self, amountAttributed);

    withdrawnFromAmm =
        MathHelper.calculateRedeemAmountWithExchangeRate(excessAmount, self.withdrawalPool.raExchangeRate); //@audit PA is ignored here

    ra += withdrawnFromAmm;
}
```



## Discussion

**ziankork**

This is a valid issue, though we removed the functionality of expiry redemption on the LV, so this would have no impact on the updated version of the protocol. still valid nonetheless 

# Issue H-7: Attackers will steal the reserve from the `Vault` by receiving `ra` in `FlashSwapRouter::__swapDsforRa()` 

Source: https://github.com/sherlock-audit/2024-08-cork-protocol-judging/issues/161 

## Found by 
0x73696d616f, KupiaSec, Smacaud, oxelmiguel
### Summary

[FlashSwapRouter::__swapDsforRa()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L288) is called as part of [FlashSwapRouter::_swapRaforDs()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L98) [whenever](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L124) the reserve is sold and the resulting `Ra` is used to provide liquidity to the `Vault` by calling [Vault::provideLiquidityWithFlashSwapFee()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L125). 

However, if we follow the code, `FlashSwapRouter::__swapDsforRa()` calls [FlashSwapRouter::__flashSwap()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L301), which calls [UniswapV2Pair::swap()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L335). Then, the Uniswap pair calls [uniswapV2Call()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L338), which calls [FlashSwapRouter::__afterFlashswapSell()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L353) and sends the `Ra` to the [caller](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L404).

Thus, the vault is providing a fee, but does not use the acquired funds to fund it, but the already existing `Ra` in the contract. The `Ra` meant for the liquidity fee is sent to the user calling `FlashSwapRouter::_swapRaforDs()`.

### Root Cause

In `FlashSwapRouter.sol:124`, `FlashSwapRouter::__swapDsforRa()` is called, which ultimately sends the `Ra` to the `msg.sender`.

### Internal pre-conditions

None.

### External pre-conditions

None.

### Attack Path

1. User calls `swapRaforDs()` and ends up receiving `Ra` which was intended for the `Vault`.

### Impact

Users can steal all reserve from the `Vault`.

### PoC

The following code snippets show how the `Ra` ends up in the `caller`, that is, the user that calls `swapRaforDs()`.

```solidity
function __flashSwap(...) internal {
    ...
    bytes memory data = abi.encode(reserveId, dsId, buyDs, msg.sender, extraData);

    univ2Pair.swap(amount0out, amount1out, address(this), data);
}

function uniswapV2Call(address sender, uint256 amount0, uint256 amount1, bytes calldata data) external {
    (Id reserveId, uint256 dsId, bool buyDs, address caller, uint256 extraData) =
        abi.decode(data, (Id, uint256, bool, address, uint256));
    ...
    if (buyDs) {
        ...
    } else {
        uint256 amount = amount0 == 0 ? amount1 : amount0;

        __afterFlashswapSell(self, amount, reserveId, dsId, caller, extraData);
    }
}

function __afterFlashswapSell(...) internal {
    ...
    IERC20(ra).safeTransfer(caller, raAttributed);
    ...
}
```

### Mitigation

The `FlashSwapRouter::__flashSwap()` has to be modified to accept a caller argument, where it accepts either the `msg.sender` or `owner()`. In the flow of selling the reserve, it should send the `Ra` to the `owner`, which is the `Vault`.



## Discussion

**ziankork**

This is a valid issue, though already fixed this prior to our trading competition. will provide links later

# Issue H-8: Users redeeming early will withdraw `Ra` without decreasing the amount locked, which will lead to stolen funds when withdrawing after expiry 

Source: https://github.com/sherlock-audit/2024-08-cork-protocol-judging/issues/166 

## Found by 
0x73696d616f, 0xNirix, KupiaSec, Matrox, dimulski, hunter\_w3b, nikhil840096, oxelmiguel, vinica\_boy
### Summary

[VaultLib::redeemEarly()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L639) is called when users redeem early via [Vault::redeemEarlyLv()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/Vault.sol#L168), which allows users to redeem `Lv` for `Ra` and pay a fee. 

In the process, the `Vault` burns `Ct` and `Ds` in [VaultLib::_redeemCtDsAndSellExcessCt()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L318) for `Ra`, by calling [PsmLib::PsmLibrary.lvRedeemRaWithCtDs()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L332). However, it never calls [RedemptionAssetManagerLib::decLocked()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/RedemptionAssetManagerLib.sol#L57) to decrease the tracked locked `Ra`, but the `Ra` leaves the `Vault` for the user redeeming.

This means that when a new `Ds` is [issued](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L43) in the `PsmLib` or users call [PsmLib::redeemWithCt()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L448), [PsmLib::_separateLiquidity()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L65) will be called and it will calculated the exchange rate to withdraw `Ra` and `Pa` as if the `Ra` amount withdrawn earlier was still there. When it calls [self.psm.balances.ra.convertAllToFree()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L73), it [converts](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/RedemptionAssetManagerLib.sol#L42) the locked amount to free and assumes these funds are available, when in reality they have been withdrawn earlier. As such, the `Ra` and `Pa` [checkpoint](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L76) will be incorrect and users will redeem more `Ra` than they should, such that the last users will not be able to withdraw and the first ones will profit.

### Root Cause

In `PsmLib.sol:125`, `self.psm.balances.ra.decLocked(amount);` is not called.

### Internal pre-conditions

None.

### External pre-conditions

None.

### Attack Path

1. User calls `Vault::redeemEarlyLv()` or `ModuleCore::issueNewDs()` is called by the admin.

### Impact

Users withdraw more funds then they should via `PsmLib::redeemWithCt()` meaning the last users can not withdraw.

### PoC

`PsmLib::lvRedeemRaWithCtDs()` does not reduce the amount of `Ra` locked.
```solidity
function lvRedeemRaWithCtDs(State storage self, uint256 amount, uint256 dsId) internal {
    DepegSwap storage ds = self.ds[dsId];
    ds.burnBothforSelf(amount);
}
```

### Mitigation

`PsmLib::lvRedeemRaWithCtDs()` must reduce the amount of `Ra` locked.
```solidity
function lvRedeemRaWithCtDs(State storage self, uint256 amount, uint256 dsId) internal {
    self.psm.balances.ra.decLocked(amount);
    DepegSwap storage ds = self.ds[dsId];
    ds.burnBothforSelf(amount);
}
```



## Discussion

**ziankork**

dup of #44 and related #156 . will fix

# Issue H-9: `VaultPoolLib::reserve()` will store the `Pa` not attributed to user withdrawals incorrectly and leave in untracked once it expires again 

Source: https://github.com/sherlock-audit/2024-08-cork-protocol-judging/issues/191 

## Found by 
0x73696d616f, 0xNirix, dimulski, sakshamguruji
### Summary

[VaultPoolLib::reserve()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L12) stores the `Pa` attributed to withdrawals in [self.withdrawalPool.stagnatedPaBalance](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L31) instead of storing the amount `attributedToAmm`. Additionally, this amount of `Pa`, the one attributed to the `Amm` is never dealt with and leads to stuck `PA`.

The comment in the code mentions 
```solidity
    // FIXME : this is only temporary, for now
    // we trate PA the same as RA, thus we also separate PA
    // the difference is the PA here isn't being used as anything
    // and for now will just sit there until rationed again at next expiry.
```
But it is incorrect as it is never rationed again, just forgotten. The [VaultPoolLib::rationedToAmm()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L170) function only uses the `Ra` [balance](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L171), not the `Pa`, which is effectively left untracked.

### Root Cause

In `VaultPoolLib:170`, the leftover non attributed `Pa` is not dealt with.

### Internal pre-conditions

None.

### External pre-conditions

None.

### Attack Path

1. `VaultPoolLib::reserve()` is called when liquidating the lp position of the `Vault` via `VaultLib::_liquidatedLP()`, triggered by users when redeeming expired liquidity vault shares or on the admin trigerring a new issuance.

### Impact

The `Pa` in the `Vault` is stuck.

### PoC

`VaultPoolLib::rationedToAmm()` does not deal with the `Pa`.
```solidity
function rationedToAmm(VaultPool storage self, uint256 ratio) internal view returns (uint256 ra, uint256 ct) {
    uint256 amount = self.ammLiquidityPool.balance;

    (ra, ct) = MathHelper.calculateProvideLiquidityAmountBasedOnCtPrice(amount, ratio);
}
```

### Mitigation

Distributed the `Pa` to users based on their `LV` shares or redeem the `Pa` for `Ra` and add liquidity to the new issued `Ds` or similar.



## Discussion

**ziankork**

valid. will fix

# Issue H-10: The `VaultLib.__liquidateUnchecked()` function unnecessarily reorders the already correctly ordered values of `raReceived` and `ctReceived` from `UniswapV2Router` 

Source: https://github.com/sherlock-audit/2024-08-cork-protocol-judging/issues/214 

## Found by 
KupiaSec
## Summary

The `__liquidateUnchecked()` function removes liquidity and receives the corresponding `RA` and `CT` from UniswapV2, returning the amounts of `RA` and `CT` received. When removing liquidity, the `removeLiquidity()` function of the `UniswapV2Router` returns `raReceived` and `ctReceived`, which are the exact amounts of `RA` and `CT` corresponding to the removed liquidity, regardless of the order of tokens `RA` and `CT` in the Uniswap pool. However, in the `__liquidateUnchecked()` function, these values are reordered as if they represent the received amounts of `token0` and `token1`, assuming that `token0 = CT` and `token1 = RA`.

## Vulnerability Detail

As noted in line 282 of the `__liquidateUnchecked()` function, the `ammRouter.removeLiquidity()` function returns two values: `raReceived` and `ctReceived`. These values represent the exact amounts of `RA` and `CT` received from the Uniswap pool, even in the case where `token0 = CT` and `token1 = RA` (see `UniswapV2Router02.sol`). Therefore, there is no need to reorder these values. However, at line 284, the function reorders them, assuming `token0 = CT` and `token1 = RA`. As a result, when `token0 = CT` and `token1 = RA`, the values of `raReceived` and `ctReceived` will be interchanged. This leads to incorrect behavior in the [_liquidatedLp()](https://github.com/sherlock-audit/2024-08-cork-protocol/tree/main/Depeg-swap/contracts/libraries/VaultLib.sol#L370) function, which processes expired states when [issuing new DS](https://github.com/sherlock-audit/2024-08-cork-protocol/tree/main/Depeg-swap/contracts/libraries/VaultLib.sol#L104). Consequently, an incorrect `RA` amount is reserved in the [reserve pool](https://github.com/sherlock-audit/2024-08-cork-protocol/tree/main/Depeg-swap/contracts/libraries/VaultLib.sol#L392), potentially leading to a loss of `RA` for users.

```solidity
VaultLib.sol

    function __liquidateUnchecked(
        State storage self,
        address raAddress,
        address ctAddress,
        IUniswapV2Router02 ammRouter,
        IUniswapV2Pair ammPair,
        uint256 lp
    ) internal returns (uint256 raReceived, uint256 ctReceived) {
        ammPair.approve(address(ammRouter), lp);

        // amountAMin & amountBMin = 0 for 100% tolerence
        (raReceived, ctReceived) =
282         ammRouter.removeLiquidity(raAddress, ctAddress, lp, 0, 0, address(this), block.timestamp);

284     (raReceived, ctReceived) = MinimalUniswapV2Library.reverseSortWithAmount224(
            ammPair.token0(), ammPair.token1(), raAddress, ctAddress, raReceived, ctReceived
        );

        self.vault.config.lpBalance -= lp;
    }

----------------------------

UniswapV2Router02.sol

    function removeLiquidity(
        address tokenA,
        address tokenB,
        uint liquidity,
        uint amountAMin,
        uint amountBMin,
        address to,
        uint deadline
    ) public virtual override ensure(deadline) returns (uint amountA, uint amountB) {
        address pair = UniswapV2Library.pairFor(factory, tokenA, tokenB);
        IUniswapV2Pair(pair).transferFrom(msg.sender, pair, liquidity); // send liquidity to pair
        (uint amount0, uint amount1) = IUniswapV2Pair(pair).burn(to);
        (address token0,) = UniswapV2Library.sortTokens(tokenA, tokenB);
@>      (amountA, amountB) = tokenA == token0 ? (amount0, amount1) : (amount1, amount0);
        require(amountA >= amountAMin, 'UniswapV2Router: INSUFFICIENT_A_AMOUNT');
        require(amountB >= amountBMin, 'UniswapV2Router: INSUFFICIENT_B_AMOUNT');
    }
```

## Impact

Loss of `RA` for users.

## Code Snippet

https://github.com/sherlock-audit/2024-08-cork-protocol/tree/main/Depeg-swap/contracts/libraries/VaultLib.sol#L270-L289

https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router02.sol#L116

## Tool used

Manual Review

## Recommendation

Eliminate the reordering.

```diff
    function __liquidateUnchecked(
        State storage self,
        address raAddress,
        address ctAddress,
        IUniswapV2Router02 ammRouter,
        IUniswapV2Pair ammPair,
        uint256 lp
    ) internal returns (uint256 raReceived, uint256 ctReceived) {
        ammPair.approve(address(ammRouter), lp);

        // amountAMin & amountBMin = 0 for 100% tolerence
        (raReceived, ctReceived) =
            ammRouter.removeLiquidity(raAddress, ctAddress, lp, 0, 0, address(this), block.timestamp);

-       (raReceived, ctReceived) = MinimalUniswapV2Library.reverseSortWithAmount224(
-           ammPair.token0(), ammPair.token1(), raAddress, ctAddress, raReceived, ctReceived
-       );

        self.vault.config.lpBalance -= lp;
    }
```



## Discussion

**ziankork**

yes this is valid and have been fixed prior to our trading competition. will provide links later

# Issue M-1: Premature LV Redemption Leading to Asset Loss For user 

Source: https://github.com/sherlock-audit/2024-08-cork-protocol-judging/issues/27 

The protocol has acknowledged this issue.

## Found by 
0xNirix, dimulski, hunter\_w3b, oxelmiguel
### Summary

 According to the Cork protocol's litepaper, Liquidity Vault tokenholders can submit a withdrawal request prior to expiry, which will be processed at expiry and can be claimed by burning the Liquidity Vault Token. However, the current implementation allows users to call `VaultCore::redeemExpiredLv` before expiry, resulting in the complete loss of user funds.

### Root Cause

In `VaultLibrary::redeemExpired`, if a user calls the function before the expiry and their userEligible balance is greater than or equal to the requested amount, the transaction proceeds without invoking `_liquidatedLp(self, dsId, ammRouter, flashSwapRouter)`. 

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L514

```solidity
 {
            uint256 dsId = self.globalAssetIdx;
            DepegSwap storage ds = self.ds[dsId];

            
            uint256 userEligible = self.vault.pool.withdrawEligible[owner];

            if (userEligible == 0 && !ds.isExpired()) {
                revert Unauthorized(owner);
            }

            // user can only redeem up to the amount they requested, when there's a DS active
            // if there's no DS active, then there's no cap on the amount of LV that can be redeemed
            if (!ds.isExpired() && userEligible < amount) {
                revert InsufficientBalance(owner, amount, userEligible);
            }

            if (ds.isExpired() && !self.vault.lpLiquidated.get(dsId)) {
                _liquidatedLp(self, dsId, ammRouter, flashSwapRouter);
                assert(self.vault.balances.ra.locked == 0);
            }
        }
```

Since this function (`_liquidatedLp`) is responsible for setting the exchange rates (`withdrawalPool.raExchangeRate` and `withdrawalPool.paExchangeRate`), these rates remain uninitialized (set to 0).

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L392


As a result, when calculating the user's allocated Redemption Asset (RA) and Pegged Asset (PA), the allocation is zero. 

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L137

The function burns Liquidity Vault (LV) tokens, decreases userEligible, but sends 0 RA and PA to the user, leading to a complete loss of funds.


### Impact

Users who attempt to call `redeemExpiredLv` before the Depeg Swap (DS) expiry risk losing all their funds.

### PoC

Run the poc in test/contracts/LvCore.ts

```ts
 describe("LossFund", function () {
    it("call redeemExpiredLv before expiry ", async function () {

      const { Id, dsId } = await issueNewSwapAssets(expiry);

      let paBalance = await fixture.pa.read.balanceOf(
        [secondSigner.account.address],
      );
      let rabalance = await fixture.ra.read.balanceOf([
        secondSigner.account.address,
      ]);

      console.log("Ra balance of user before deposit ",rabalance);


      await moduleCore.write.depositLv([Id, depositAmount]);
      await moduleCore.write.depositLv([Id, depositAmount], {
        account: secondSigner.account,
      });

      const msgPermit = await helper.permit({
        amount: depositAmount,
        deadline,
        erc20contractAddress: fixture.lv.address!,
        psmAddress: moduleCore.address,
        signer: secondSigner,
      });

      await moduleCore.write.requestRedemption(
        [Id, depositAmount, msgPermit, deadline],
        {
          account: secondSigner.account,
        }
      );

      let lvBalance = await moduleCore.read.lockedLvfor(
        [Id, secondSigner.account.address],
        {
          account: secondSigner.account,
        }
      );

      console.log("LV locked for user before request redemption ",lvBalance);
    
      await moduleCore.write.redeemExpiredLv(
        [Id, secondSigner.account.address, depositAmount],
        {
          account: secondSigner.account,
        }
      );

      paBalance = await fixture.pa.read.balanceOf(
        [secondSigner.account.address],
      );
      rabalance = await fixture.ra.read.balanceOf([
        secondSigner.account.address,
      ]);

      lvBalance = await moduleCore.read.lockedLvfor(
        [Id, secondSigner.account.address],
        {
          account: secondSigner.account,
        }
      );

      console.log("LV locked for user after redemption ", lvBalance);
      
      console.log("PA balance of user after redemption ", paBalance);
      console.log("RA balance of user after redemption ", rabalance);
    });
  });
```

### Output

![Screenshot from 2024-09-04 18-47-09](https://github.com/user-attachments/assets/f150cb67-7a2b-4cfa-9adf-6c72cfbd2063)


### Mitigation

The `redeemExpired` function is intended to be called at DS expiry. To prevent misuse, a simple fix would be to add a check to ensure it reverts when called before expiry. The `redeemEarlyLv` function already handles LV redemption before expiry. However, if the current behavior is intended, the internal function logic should be refactored to properly calculate and set the exchange rates even before expiry to avoid zero allocations and loss of user funds. 



## Discussion

**ziankork**

This is a valid bug, but we decided to remove this functionality altogether in favor of redeemEarly being the default, implication of this is that user can remove their liquidity at any moment with no fee and this simplifies the LV withdrawals logic

# Issue M-2: If all LV tokens are requested for redeem, issuing new CT and DS tokens will revert 

Source: https://github.com/sherlock-audit/2024-08-cork-protocol-judging/issues/114 

The protocol has acknowledged this issue.

## Found by 
dimulski
### Summary

The Corc protocol creates a pair for RA:PA tokens, and then can issue CT and DS tokens for that pair, which can be used for different purposes such as hedging against PA tokens deepening from the RA token. The CT and DS tokens have an expiration. Once they have expired the protocol admins can issue new CT and DS tokens via the [ModuleCore::issueNewDs()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/ModuleCore.sol#L57-L86) function. However in some cases that won't be possible. The [ModuleCore::issueNewDs()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/ModuleCore.sol#L57-L86) function calls a lot of functions that have to do with deploying asset contracts and setting up parameters, but the problem arises in the [VaultLib::onNewIssuance()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L92-L108) function, which calls the [VaultLib::__provideAmmLiquidityFromPool()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L174-L189) function:
```solidity
    function __provideAmmLiquidityFromPool(
        State storage self,
        IDsFlashSwapCore flashSwapRouter,
        address ctAddress,
        IUniswapV2Router02 ammRouter
    ) internal {
        uint256 dsId = self.globalAssetIdx;

        uint256 ctRatio = __getAmmCtPriceRatio(self, flashSwapRouter, dsId);

        (uint256 ra, uint256 ct) = self.vault.pool.rationedToAmm(ctRatio);

        __provideLiquidity(self, ra, ct, flashSwapRouter, ctAddress, ammRouter, dsId);

        self.vault.pool.resetAmmPool();
    }
```
There are several overengineered calls that first separate the liquidity for the previously expired CT and DS tokens, based on how much LV tokens users minted, and how much of those LV tokens were requested for withdrawal. If all of the users that minted LV tokens request to withdraw them, the internal call to [VaultPoolLib::rationedToAmm()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L170-L174) function will return 0. And when the contract tries to deposit 0 RA and 0 CT tokens to a newly created UniV2 Pair, the contract will revert due to underflow. 
```solidity
    function mint(address to) external lock returns (uint liquidity) {
        ...
        uint _totalSupply = totalSupply; // gas savings, must be defined here since totalSupply can update in _mintFee
        if (_totalSupply == 0) {
            liquidity = Math.sqrt(amount0.mul(amount1)).sub(MINIMUM_LIQUIDITY);
            _mint(address(0), MINIMUM_LIQUIDITY); // permanently lock the first MINIMUM_LIQUIDITY tokens
        } else {
            liquidity = Math.min(amount0.mul(_totalSupply) / _reserve0, amount1.mul(_totalSupply) / _reserve1);
        }
        ...
    }
```
All users requesting to redeem their LV in the same time frame is fairly possible scenario. Keep in mind that there is one LV token for a RA:PA pair, and several CT and DS tokens can be issued for a pair of RA:PA tokens. So it just may happens that at the 3rd issuing of CT and DS tokens, all users have requested to redeem their LV tokens. 

### Root Cause

In the [VaultLib::__provideAmmLiquidityFromPool()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L174-L189) function there is no  check whether there is liquidity that can be provided to the newly created UniV2 pair.

### Internal pre-conditions
1. All the LV tokens that were minted by the system are requested for redemption.

### External pre-conditions

_No response_

### Attack Path

_No response_

### Impact

The [ModuleCore::issueNewDs()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/ModuleCore.sol#L57-L86) function is most critical function for the protocol, without it new DS and CT tokens can't be issued and the protocol becomes obsolete, simply redeploying the protocol won't fix this issue, as there can be other pairs for RA:PA tokens, and there can be tokens that users are still to reclaim, keep in mind that once CT and DS tokens have expired RA tokens that were deposited by users can still be withdrawn from users by calling the [Psm::redeemWithCT()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/Psm.sol#L198-L210) function and depositing CT tokens, that have already expired. Thus the medium severity.

### PoC

[Gist](https://gist.github.com/AtanasDimulski/3f9bfc84c63e1c977b877613b644c0e2)

After following the steps in the above mentioned [gist](https://gist.github.com/AtanasDimulski/3f9bfc84c63e1c977b877613b644c0e2) add the following test to the ``AuditorTests.t.sol`` file:

```solidity
    function test_IssuingNewDsWillRevertWhenNoLiquidityCanBeProvided() public {
        vm.startPrank(alice);
        WETH.mint(alice, 10e18);
        WETH.approve(address(moduleCore), type(uint256).max);
        moduleCore.depositLv(id, 10e18);
        Asset(lvAddress).approve(address(moduleCore), type(uint256).max);
        moduleCore.requestRedemption(id, 10e18);
        vm.stopPrank();

        vm.startPrank(bob);
        WETH.mint(bob, 10e18);
        WETH.approve(address(moduleCore), type(uint256).max);
        moduleCore.depositPsm(id, 10e18);
        vm.stopPrank();

        vm.startPrank(tom);
        WETH.mint(tom, 10e18);
        WETH.approve(address(moduleCore), type(uint256).max);
        moduleCore.depositLv(id, 10e18);
        Asset(lvAddress).approve(address(moduleCore), type(uint256).max);
        moduleCore.requestRedemption(id, 10e18);
        vm.stopPrank();

        /// @notice skip 1100 seconds so that the CT and DS tokens are expired
        skip(1100);

        vm.startPrank(owner);
        vm.expectRevert(bytes("ds-math-sub-underflow"));
        corkConfig.issueNewDs(id, block.timestamp + expiry, 1e18, 5e18);
        vm.stopPrank();
    }
```

To run the test use: ``forge test -vvv --mt test_IssuingNewDsWillRevertWhenNoLiquidityCanBeProvided``

### Mitigation

In the [VaultLib::__provideAmmLiquidityFromPool()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L174-L189) function first check whether there is liquidity that can be provided.



## Discussion

**ziankork**

This is a valid issue, though there's an update in which we removed all the functionality related to the expiry redemption in the LV(all of withdrawal is done through redeemEarly). so won't fix

# Issue M-3: Wrong accounting of locked RA when repurchasing DS+PA with RA 

Source: https://github.com/sherlock-audit/2024-08-cork-protocol-judging/issues/155 

## Found by 
0x73696d616f, KupiaSec, dimulski, sakshamguruji, vinica\_boy
## Summary

Users have the option to repurchase DS + PA by providing RA to the PSM. A portion of the RA provided is taken as a fee, and this fee is used to mint CT + DS for providing liquidity to the AMM pair.
## Vulnerability Detail

NOTE: Currently `PsmLib.sol` incorrectly uses `lockUnchecked()` for the amount of RA provided by the user. As discussed with sponsor it should be `lockFrom()` in order to account for the RA provided.

After initially locking the RA provided, part of this amount is used to provide liquidity to the AMM via `VaultLibrary.provideLiquidityWithFee()`

```solidity
function repurchase(
        State storage self,
        address buyer,
        uint256 amount,
        IDsFlashSwapCore flashSwapRouter,
        IUniswapV2Router02 ammRouter
    ) internal returns (uint256 dsId, uint256 received, uint256 feePrecentage, uint256 fee, uint256 exchangeRates) {
        DepegSwap storage ds;

        (dsId, received, feePrecentage, fee, exchangeRates, ds) = previewRepurchase(self, amount);

        // decrease PSM balance
        // we also include the fee here to separate the accumulated fee from the repurchase
        self.psm.balances.paBalance -= (received);
        self.psm.balances.dsBalance -= (received);

        // transfer user RA to the PSM/LV
        // @audit-issue shouldnt it be lock checked, deposit -> redeemWithDs -> repurchase - locked would be 0
        self.psm.balances.ra.lockUnchecked(amount, buyer);

        // transfer user attrubuted DS + PA
        // PA
        (, address pa) = self.info.underlyingAsset();
        IERC20(pa).safeTransfer(buyer, received);

        // DS
        IERC20(ds._address).transfer(buyer, received);

        // Provide liquidity
        VaultLibrary.provideLiquidityWithFee(self, fee, flashSwapRouter, ammRouter);
    }
```

`provideLiqudityWithFee` internally uses `__provideLiquidityWithRatio()` which calculates the amount of RA that should be used to mint CT+DS in order to be able to provide liquidity.
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

`__provideLiquidity()` uses `PsmLibrary.unsafeIssueToLv()` to account the RA locked.
```solidity
function __provideLiquidity(
        State storage self,
        uint256 raAmount,
        uint256 ctAmount,
        IDsFlashSwapCore flashSwapRouter,
        address ctAddress,
        IUniswapV2Router02 ammRouter,
        uint256 dsId
    ) internal {
        // no need to provide liquidity if the amount is 0
        if (raAmount == 0 && ctAmount == 0) {
            return;
        }

        PsmLibrary.unsafeIssueToLv(self, ctAmount);

        __addLiquidityToAmmUnchecked(self, raAmount, ctAmount, self.info.redemptionAsset(), ctAddress, ammRouter);

        _addFlashSwapReserve(self, flashSwapRouter, self.ds[dsId], ctAmount);
    }
```

RA locked is incremented with the amount of CT tokens minted.
```solidity
function unsafeIssueToLv(State storage self, uint256 amount) internal {
        uint256 dsId = self.globalAssetIdx;

        DepegSwap storage ds = self.ds[dsId];

        self.psm.balances.ra.incLocked(amount);

        ds.issue(address(this), amount);
    }

```

Consider the following scenario:
For simplicity the minted values in the example may not be accurate, but the idea is to show the wrong accounting of locked RA.

1. PSM has 1000 RA locked.
2. Alice repurchase 100 DS+PA with providing 100 RA and we have fee=5% making the fee = 5
3. PSM will have 1100 RA locked and ra.locked would be 1100 also.
5. In `__provideLiquidity()` let say 3 of those 5 RA are used to mint CT+DS.
6. `PsmLibrary.unsafeIssueToLv()` would add 3 RA to the locked amount, making the psm.balances.ra.locked = 1103 while the real balance would still be 1100.

## Impact

Wrong accounting of locked RA would lead to over-distribution of rewards for users + after time last users to redeem might not be able to redeem as there wont be enough RA in the contract due to previous over-distribution. This breaks a core functionality of the protocol and the likelihood of this happening is very high, making the overall severity High.
## Code Snippet

`PsmLib.repurchase()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/PsmLib.sol#L293
## Tool used

Manual Review
## Recommendation

Consider either not accounting for the fee used to mint CT+DS as it is already accounted or do not initially account for the fee when users provide RA.



## Discussion

**ziankork**

This is a valid issue, will fix by not accounting the fee when users provide RA

# Issue M-4: Admin new issuance or user calling `Vault::redeemExpiredLv()` after `Psm::redeemWithCt()` will lead to stuck funds when trying to withdraw 

Source: https://github.com/sherlock-audit/2024-08-cork-protocol-judging/issues/156 

## Found by 
0x73696d616f
### Summary

[VaultLib::_liquidatedLp()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L349) calls [PsmLib::lvRedeemRaWithCtDs()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L377), which redeems `ra` with `ct` and `ds`. However, if [PsmLib::_separateLiquidity()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L65) has already been called, this will lead to an incorrect tracking of funds as `PsmLib::_separateLiquidity()` [checkpointed](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/PsmLib.sol#L76) the total supply of `ct`, but the `Vault` will redeem some of it using `PsmLib::lvRedeemRaWithCtDs()`, leading to some `pa` that will never be withdrawn and the vault withdraws too many `Ra`, which means users will not be able to redeem their `ct` for `ra` and `pa` as `ra` has been withdrawn already and it reverts.



### Root Cause

In `VaultLib.sol:377`, it calls `PsmLib::lvRedeemRaWithCtDs()` even if `PsmLib::_separateLiquidity()` has already been called. It should skip it in this case.

### Internal pre-conditions

1. `PsmLib::_separateLiquidity()` needs to be called before `VaultLib::_liquidatedLp()`, which may be done on a new issuance when the admin calls [ModuleCore::issueNewDs()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/ModuleCore.sol#L57) or by users calling `Psm::redeemWithCT()` before `Vault::redeemExpiredLv()`.

### External pre-conditions

None.

### Attack Path

1. Admin calls `ModuleCore::issueNewDs()`.
Or users call `Psm::redeemWithCT()` before `Vault::redeemExpiredLv()`.

### Impact

As the `Vault` withdraws `Ra` after the checkpoint and burns the corresponding `Ct` tokens, it will withdraw too many `ra` and not withdraw the `pa` it was entitled to.

### PoC

`ModuleCore::issueNewDs()` calls `PsmLib::onNewIssuance()` before `VaultLib::onNewIssuance()` always triggering this bug.
```solidity
function issueNewDs(Id id, uint256 expiry, uint256 exchangeRates, uint256 repurchaseFeePrecentage)
    external
    override
    onlyConfig
    onlyInitialized(id)
{
    ...
    PsmLibrary.onNewIssuance(state, ct, ds, ammPair, idx, prevIdx, repurchaseFeePrecentage);

    getRouterCore().onNewIssuance(id, idx, ds, ammPair, 0, ra, ct);

    VaultLibrary.onNewIssuance(state, prevIdx, getRouterCore(), getAmmRouter());
    ...
}
```

`PsmLib::_separateLiquidity()` checkpoints `ra` and `pa` based on `ct` supply:
```solidity
function _separateLiquidity(State storage self, uint256 prevIdx) internal {
    ...
    self.psm.poolArchive[prevIdx] = PsmPoolArchive(availableRa, availablePa, IERC20(ds.ct).totalSupply());
    ...
}
```

`VaultLib::_liquidatedLp()` redeems `ra` with `ct` and `ds` when it should have skipped it has liquidity has already been checkpointed in `PsmLib::_separateLiquidity()`.
```solidity
function _liquidatedLp(
    State storage self,
    uint256 dsId,
    IUniswapV2Router02 ammRouter,
    IDsFlashSwapCore flashSwapRouter
) internal {
    ...
    PsmLibrary.lvRedeemRaWithCtDs(self, redeemAmount, dsId);

    // if the reserved DS is more than the CT that's available from liquidating the AMM LP
    // then there's no CT we can use to effectively redeem RA + PA from the PSM
    uint256 ctAttributedToPa = reservedDs >= ctAmm ? 0 : ctAmm - reservedDs;

    uint256 psmPa;
    uint256 psmRa;

    if (ctAttributedToPa != 0) {
        (psmPa, psmRa) = PsmLibrary.lvRedeemRaPaWithCt(self, ctAttributedToPa, dsId);
    }

    psmRa += redeemAmount;

    self.vault.pool.reserve(self.vault.lv.totalIssued(), raAmm + psmRa, psmPa);
}
```

### Mitigation

If the liquidity has been separated, skip redeeming `ra` for `ct` and `ds`.
```solidity
function _liquidatedLp(
    State storage self,
    uint256 dsId,
    IUniswapV2Router02 ammRouter,
    IDsFlashSwapCore flashSwapRouter
) internal {
    ...
    if (!self.psm.liquiditySeparated.get(prevIdx)) {
        PsmLibrary.lvRedeemRaWithCtDs(self, redeemAmount, dsId);
    }
    ...
}
```



## Discussion

**ziankork**

related to #44, the function that's problematic here is `PsmLibrary::lvRedeemRaWithCtDs`. will fix

# Issue M-5: Possibility of DoS when issuing new DS 

Source: https://github.com/sherlock-audit/2024-08-cork-protocol-judging/issues/180 

The protocol has acknowledged this issue.

## Found by 
0x73696d616f, vinica\_boy
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



## Discussion

**ziankork**

As of now, we've updated the contracts and removed expiry redemption for LV. in this case, CT user can still withdraws their share of RA so no problem there. while this is valid and following our contract update, we won't fix this

# Issue M-6: Admin will not be able to only pause deposits in the `Vault` due to incorrect check leading to DoSed withdrawals 

Source: https://github.com/sherlock-audit/2024-08-cork-protocol-judging/issues/182 

## Found by 
0x73696d616f, 404Notfound, 4gontuk, Abhan1041, KupiaSec, Pro\_King, Smacaud, Trooper, durov, hunter\_w3b, korok, mladenov, octeezy, ravikiran.web3, tinnohofficial, tmotfl, ydlee, yovchev\_yoan
### Summary

The modifier [LVDepositNotPaused](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/ModuleState.sol#L108) in [Vault::depositLv()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/Vault.sol#L33) checks [states[id].vault.config.isWithdrawalPaused](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/core/ModuleState.sol#L109) instead of [states[id].vault.config.isDepositPaused](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/State.sol#L107), which means deposits will only be paused if withdrawals are paused, DoSing withdrawals.

### Root Cause

In `ModuleState:109`, it checks `states[id].vault.config.isWithdrawalPaused` when it should check `states[id].vault.config.isDepositPaused`.

### Internal pre-conditions

1. Admin pauses deposits.

### External pre-conditions

None.

### Attack Path

1. Admin sets deposits paused, but deposits are not actually paused due to the incorrect modifier.
2. Admin either leaves deposits unpaused or pauses both deposits and withdrawals, DoSing withdrawals.

### Impact

Admin is not able to pause deposits alone which would lead to loss of funds as this is an emergency mechanism. If the admin wants to pause deposits, withdrawals would also have to be paused, DoSing withdrawals.

### PoC

`ModuleState::LVDepositNotPaused()` is incorrect:
```solidity
modifier LVDepositNotPaused(Id id) {
    if (states[id].vault.config.isWithdrawalPaused) { //@audit isDepositPaused
        revert LVDepositPaused();
    }
    _;
}
```

### Mitigation

`ModuleState::LVDepositNotPaused()` should be:
```solidity
modifier LVDepositNotPaused(Id id) {
    if (states[id].vault.config.isDepositPaused) {
        revert LVDepositPaused();
    }
    _;
}
```



## Discussion

**ziankork**

will fix

# Issue M-7: Admin will not be able to upgrade the smart contracts, breaking core functionality and rendering the upgradeable contracts useless 

Source: https://github.com/sherlock-audit/2024-08-cork-protocol-judging/issues/185 

## Found by 
0x73696d616f, MadSisyphus, dimulski
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



## Discussion

**ziankork**

yes this is valid, we already removed this prior to our trading competition. will provide links later

# Issue M-8: Withdrawing all `lv` before expiry will lead to lost funds in the Vault 

Source: https://github.com/sherlock-audit/2024-08-cork-protocol-judging/issues/211 

## Found by 
0x73696d616f, NoOne, dimulski
### Summary

[VaultLib:redeemEarly()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L639) redeems users' liquidity vault positions, `lv` for `Ra`, [before](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L649) expiry. After expiry, it is not possible to deposit into the vault or redeem early.

Whenever all users redeem early, when it gets to the expiry date, [VaultLib::_liquidatedLp()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L349) is called to remove the lp position from the `AMM` into `Ra` and `Ct` (redeem from the `PSM` into more `Ra` and `Pa`) and split among all `lv` holders.

However, as the total supply of `lv` is `0` due to users having redeemed all their positions via `VaultLib::redeemEarly()`, when it gets to [VaultPoolLib::reserve()](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultLib.sol#L392), it [reverts](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/VaultPoolLib.sol#L19) due to a division by `0` [error](https://github.com/sherlock-audit/2024-08-cork-protocol/blob/main/Depeg-swap/contracts/libraries/MathHelper.sol#L134), never allowing the `Vault::_liquidatedLp()` call to go through. 

As the `Ds` has expired, it is also not possible to deposit into it to increase the `lv` supply, so all funds are forever stuck.

### Root Cause

In `MathHelper:134`, the `ratePerLv` reverts due to division by 0. It should calculate the rate after the return guard that checks if the `totalLvIssued == 0`.

### Internal pre-conditions

1. All users must redeem early.

### External pre-conditions

None.

### Attack Path

1. All users redeem early via `Vault::redeemEarlyLv()`.
2. The `Ds` expires and there is no `lv` tokens, making all funds stuck.

### Impact

All funds are stuck.

### PoC

The `MathHelper` separates liquidity by calculating first the `ratePerLv`, which will trigger a division by 0 revert.
```solidity
function separateLiquidity(uint256 totalAmount, uint256 totalLvIssued, uint256 totalLvWithdrawn)
    external
    pure
    returns (uint256 attributedWithdrawal, uint256 attributedAmm, uint256 ratePerLv)
{
    // with 1e18 precision
    ratePerLv = ((totalAmount * 1e18) / totalLvIssued);

    // attribute all to AMM if no lv issued or withdrawn
    if (totalLvIssued == 0 || totalLvWithdrawn == 0) {
        return (0, totalAmount, ratePerLv);
    }
    ...
}
```

### Mitigation

The `MathHelper` should place the return guard first:
```solidity
function separateLiquidity(uint256 totalAmount, uint256 totalLvIssued, uint256 totalLvWithdrawn)
    external
    pure
    returns (uint256 attributedWithdrawal, uint256 attributedAmm, uint256 ratePerLv)
{
    // attribute all to AMM if no lv issued or withdrawn
    if (totalLvIssued == 0 || totalLvWithdrawn == 0) {
        return (0, totalAmount, 0);
    }
    ...
    // with 1e18 precision
    ratePerLv = ((totalAmount * 1e18) / totalLvIssued);
}
```



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**tsvetanovv** commented:
> Borderline Low/Medium



**ziankork**

This is valid. will fix

# Issue M-9: Rebasing tokens are not supported contrary to the readme and will lead to loss of funds 

Source: https://github.com/sherlock-audit/2024-08-cork-protocol-judging/issues/235 

The protocol has acknowledged this issue.

## Found by 
0x73696d616f, Kirkeelee, dimulski, sakshamguruji
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



## Discussion

**ziankork**

this is intended as there's no easy way to sync balance between psm and vault since when we sync we have to somehow calculate the virtual reserves of each PSM and Vault from the total shares that the contract holds. so won't fix

# Issue M-10: Providing liquidity to the AMM does not check the return value of actually provided tokens leading to locked funds. 

Source: https://github.com/sherlock-audit/2024-08-cork-protocol-judging/issues/240 

## Found by 
0x73696d616f, vinica\_boy
## Summary

When providing liquidity to an AMM pair, the protocol specifies both the desired amount of tokens to be provided and a minimum amount to be accepted. Any difference between the two—meaning the amount not used by the AMM—should be properly accounted for within the protocol, as it is not taken by the AMM.

## Vulnerability Detail

The `__addLiquidityToAmmUnchecked` function is used to provide liquidity to the RA:CT AMM pair. In the current implementation, `raTolerance` and `ctTolerance` are calculated based on the reserves of the pair during the current transaction with 1% slippage tolerance. The amounts to be provided are determined by the current price ratio in the pair, which ensures that the amounts are almost always exactly what the AMM expects to maintain the `X * Y = K` constant product formula.
```solidity
function __addLiquidityToAmmUnchecked(
        State storage self,
        uint256 raAmount,
        uint256 ctAmount,
        address raAddress,
        address ctAddress,
        IUniswapV2Router02 ammRouter
    ) internal {
        (uint256 raTolerance, uint256 ctTolerance) =
            MathHelper.calculateWithTolerance(raAmount, ctAmount, MathHelper.UNIV2_STATIC_TOLERANCE);

        ERC20(raAddress).approve(address(ammRouter), raAmount);
        ERC20(ctAddress).approve(address(ammRouter), ctAmount);

        (address token0, address token1, uint256 token0Amount, uint256 token1Amount) =
            MinimalUniswapV2Library.sortTokensUnsafeWithAmount(raAddress, ctAddress, raAmount, ctAmount);
        (,, uint256 token0Tolerance, uint256 token1Tolerance) =
            MinimalUniswapV2Library.sortTokensUnsafeWithAmount(raAddress, ctAddress, raTolerance, ctTolerance);

        (,, uint256 lp) = ammRouter.addLiquidity(
            token0, token1, token0Amount, token1Amount, token0Tolerance, token1Tolerance, address(this), block.timestamp
        );

        self.vault.config.lpBalance += lp;
    }

```

The current implementation does not check the actual amounts used by the AMM when providing liquidity. As a result, small differences (1-2 wei of the corresponding token) between the provided amount and the actual amount used by the AMM may remain locked in the contract. These differences arise from rounding in the RA:CT 
price ratio calculations and the corresponding amounts that should be provided. Over time, these small discrepancies could accumulate, leading to higher amount of locked tokens in the contract.

PoC:

Adjust the `__addLiquidityToAmmUnchecked()` function to:

```solidity 
function __addLiquidityToAmmUnchecked(
        State storage self,
        uint256 raAmount,
        uint256 ctAmount,
        address raAddress,
        address ctAddress,
        IUniswapV2Router02 ammRouter
    ) internal {
        (uint256 raTolerance, uint256 ctTolerance) =
            MathHelper.calculateWithTolerance(raAmount, ctAmount, MathHelper.UNIV2_STATIC_TOLERANCE);

        ERC20(raAddress).approve(address(ammRouter), raAmount);
        ERC20(ctAddress).approve(address(ammRouter), ctAmount);

        (address token0, address token1, uint256 token0Amount, uint256 token1Amount) =
            MinimalUniswapV2Library.sortTokensUnsafeWithAmount(raAddress, ctAddress, raAmount, ctAmount);
        (,, uint256 token0Tolerance, uint256 token1Tolerance) =
            MinimalUniswapV2Library.sortTokensUnsafeWithAmount(raAddress, ctAddress, raTolerance, ctTolerance);
        
        uint256 lp;
        // add one more block to avoid stack too deep errors
        {
            uint256 actual0;
            uint256 actual1;
            (actual0, actual1, lp) = ammRouter.addLiquidity(
                token0, token1, token0Amount, token1Amount, token0Tolerance, token1Tolerance, address(this), block.timestamp
            );
            if(actual0 != token0Amount || actual1 != token1Amount){
                revert();
            }
        }
        self.vault.config.lpBalance += lp;
    }
```

Running the tests with the following function would result in some tests failing due to this difference in provided and used amounts.
## Impact

The impact of these small amounts of locked funds is not significant on their own, but due to the compound effect over time and the high likelihood of this happening with each liquidity provision, the overall severity of the issue should be considered Medium.
## Code Snippet

`VaultLib.__addLiquidityToAmmUnchecked()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L55
## Tool used

Manual Review
## Recommendation

To handle the small differences between the provided and actual amounts used by the AMM, the return values of the `addLiquidity()` function should be checked, as shown in the adjusted `__addLiquidityToAmmUnchecked()` function. This allows the protocol to detect any discrepancies and take appropriate action.

Depending on the protocol's decision, these leftover funds can either be:
*  Returned to users to prevent token loss, ensuring they are not penalized by the rounding differences.
* Accounted for by the protocol and later used for liquidity provision or distributed as rewards, thereby ensuring the funds are not wasted and remain within the protocol’s ecosystem.



## Discussion

**ziankork**

yes this is valid and have been fixed, will provide the fix links later

# Issue M-11: Malicious actor will frontrun user redeeming early permit call and DoS the user from withdrawing, being problematic whenever it's close to expiry 

Source: https://github.com/sherlock-audit/2024-08-cork-protocol-judging/issues/246 

## Found by 
0x73696d616f
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



## Discussion

**ziankork**

valid. will fix

