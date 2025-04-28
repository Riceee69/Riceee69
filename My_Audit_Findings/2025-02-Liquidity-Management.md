# Liquidity Management - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Wrong Fee Refund Logic During Withdrawal and Cancel Flow Leads to Improper Fee Refunds](#H-01)
    - ### [H-02. Deposit during an active 1x leveraged long position can result in a complete DoS of the protocol due to a Stalling Flow State](#H-02)
    - ### [H-03. Burning user deposit information before verifying their eligibility for a fee refund will always result in no fees being refunded in PerpetualVault::__handleReturn](#H-03)

- ## Low Risk Findings
    - ### [L-01. PerpetualVault withdrawals are affected by global parameter updates](#L-01)
    - ### [L-02. Cancelling a Flow after a Position Is Created Might Result in Inflation/Deflation of Shares](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Gamma

### Dates: Feb 11th, 2025 - Feb 25th, 2025

[View Full Report Here](https://codehawks.cyfrin.io/c/2025-02-gamma/results?lt=contest&page=1&sc=reward&sj=reward&t=report)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 3
- Medium: 0
- Low: 2


# High Risk Findings

## <a id='H-01'></a>H-01. Wrong Fee Refund Logic During Withdrawal and Cancel Flow Leads to Improper Fee Refunds            



## Summary

The `PerpetualVault` contract includes logic for processing withdrawals or to cancel an onging withdrawal flow and refund unused execution fees to users. However, the parameters used for `IGmxProxy(gmxProxy).refundExecutionFee` to refund fees leads to execution fees refunded to the wrong user and also results in incorrect refund amounts.

## Vulnerability Details

In the withdrawal flow and during cancelling a withdrawal flow, the contract attempts to refund execution fees, but it incorrectly references `depositInfo[counter]` instead of `depositInfo[depositId]` / `depositInfo[flowData]` respectively. Since `counter` might point to the  latest depositor Id (unless last action was a `cancelFlow` during deposit), this leads to:

* Execution fees being refunded to the wrong user.
* Incorrect refund amounts being issued.

#### Affected Code:

In `_handleReturn` function

```solidity
if (refundFee) {
    uint256 usedFee = callbackGasLimit * tx.gasprice;
    if (depositInfo[depositId].executionFee > usedFee) {
        try IGmxProxy(gmxProxy).refundExecutionFee(
            depositInfo[counter].owner,                           <@
            depositInfo[counter].executionFee - usedFee           
        ) {} catch {}
    }
}
```

In `_cancelFlow` function

```solidity
else if (flow == FLOW.WITHDRAW) {
    try IGmxProxy(gmxProxy).refundExecutionFee(depositInfo[counter].owner, depositInfo[counter].executionFee) {}     <@
        catch {}
}
```

## Impact

* Users will not receive their execution fee refunds.
* Refunds may be incorrectly allocated to other users.

## Tools Used

* Manual Review

## Recommendations

Update the contract to ensure that the correct `depositId` is used when referencing execution fee refunds.

### Suggested Fix:

```diff
if (refundFee) {
    uint256 usedFee = callbackGasLimit * tx.gasprice;
    if (depositInfo[depositId].executionFee > usedFee) {
        try IGmxProxy(gmxProxy).refundExecutionFee(
+           depositInfo[depositId].owner,
+           depositInfo[depositId].executionFee - usedFee
        ) {} catch {}
    }
}
```

```diff
else if (flow == FLOW.WITHDRAW) {
+   try IGmxProxy(gmxProxy).refundExecutionFee(depositInfo[flowData].owner, depositInfo[flowData].executionFee) {}
        catch {}
}
```

## <a id='H-02'></a>H-02. Deposit during an active 1x leveraged long position can result in a complete DoS of the protocol due to a Stalling Flow State            



## Summary

The Perpetual Vault contract enables 1x Long positions by swapping the collateral token deposited by users into the index token. These swaps are executed using ParaSwap and GMX. To facilitate these transactions, the `PerpetualVault` contract includes a `_runSwap` function that takes a bytes array containing swap data, the swap direction, and the current prices of the market tokens. The swap data is generated off-chain and may contain only ParaSwap data, GMX swap data, or both as per the code natspec. 

## Vulnerability Issues

The problem arises when a user deposits collateral while a 1x Long position is active. The protocol is designed to add the new collateral to the existing position. This process is managed via a keeper mechanism.  The `nextAction.selector` is set to`INCREASE_ACTION`, then the keeper calls `runNextAction`, the function invokes `_runSwap` to swap the newly added collateral into the index token.

The issue occurs if the swap data includes only ParaSwap data. The collateral swap is executed using `_doDexSwap`, but the flow variable is not reset after the transaction, leading to an inconsistent state.

#### Faulty Code Block:

[PerpetualVault.sol#L1003-L1006](https://github.com/CodeHawks-Contests/2025-02-gamma/blob/84b9da452fc84762378481fa39b4087b10bab5e0/contracts/PerpetualVault.sol#L1003-L1006)

#### Steps to Reproduce

1. The vault has an active long position with 1x leverage.
2. A user deposits collateral into the vault, flow is set to DEPOSIT. 
3. The keeper generates swap data that only involves swapping via `_doDexSwap`.
4. The swap takes place, but the flow does not get reset as intended.
5. This leads to the loss of major protocol functionality.

## Impact

This vulnerability results in a complete denial-of-service (DoS) for critical protocol functions such as: 

* Deposits
* Withdrawals
* Opening / Closing positions&#x20;
* Claiming collateral rebates

## Proof of Concept (PoC)

The following test case demonstrates the issue using the provided ParaSwap swap data in the test suite:

```solidity
function test_Run_Paraswap_during_active_position_deposit() external {
    address keeper = PerpetualVault(vault).keeper();
    address alice = makeAddr("alice");
    depositFixture(alice, 1e10);

    // Alice deposits collateral, and the vault opens a 1x Long Position
    MarketPrices memory prices = mockData.getMarketPrices();
    bytes memory paraSwapData = mockData.getParaSwapData(vault);
    bytes[] memory swapData = new bytes[](1);
    swapData[0] = abi.encode(PROTOCOL.DEX, paraSwapData);
    vm.prank(keeper);
    PerpetualVault(vault).run(true, true, prices, swapData);

    address bob = makeAddr("bob");
    deal(bob, 1 ether); // Cover execution fees
    depositFixture(bob, 1e10);

    // Bob's deposit sets nextAction to INCREASE_ACTION, and the keeper calls runNextAction
    prices = mockData.getMarketPrices();
    paraSwapData = mockData.getParaSwapData(vault);
    swapData = new bytes[](1);
    swapData[0] = abi.encode(PROTOCOL.DEX, paraSwapData);
    vm.prank(keeper);
    PerpetualVault(vault).runNextAction(prices, swapData);

    assertEq(uint8(PerpetualVault(vault).flow()), 1); // The DEPOSIT flow is never reset
}
```

This test case confirms that after the swap is completed, the flow state remains `DEPOSIT` (`1`), preventing further protocol actions.

```solidity
enum FLOW {
    NONE,
    DEPOSIT,  // 1
    SIGNAL_CHANGE,
    WITHDRAW,
    COMPOUND,
    LIQUIDATION
}
```

## Tools Used

* Manual Review
* Foundry

## Recommendations

To resolve this issue, call the `_finalize()` function at the end of the deposit flow during a swap that takes place via `_runSwap` when ParaSwap is the sole DEX used. This resets the global state variables and allows the protocol to continue operating as intended.

### Suggested Fix:

```diff
function _runSwap(bytes[] memory metadata, bool isCollateralToIndex, MarketPrices memory prices)
    internal
    returns (bool completed)
{
    [...]
    else {
        if (metadata.length != 1) {
            revert Error.InvalidData();
        }
        (PROTOCOL _protocol, bytes memory data) = abi.decode(metadata[0], (PROTOCOL, bytes));
        if (_protocol == PROTOCOL.DEX) {
            uint256 outputAmount = _doDexSwap(data, isCollateralToIndex);

            // update global state
            if (flow == FLOW.DEPOSIT) {
                // last `depositId` equals with `counter` because another deposit is not allowed before previous deposit is completely processed
                _mint(counter, outputAmount + swapProgressData.swapped, true, prices);
+               _finalize(hex"");
            } 
        [...]
        }
    }   
}
```

## <a id='H-03'></a>H-03. Burning user deposit information before verifying their eligibility for a fee refund will always result in no fees being refunded in PerpetualVault::__handleReturn            



## Vulnerability Details

When processing withdrawals, `_handleReturn` is executed at the end of the withdrawal flow to transfer collateral back to the user proportionally to their share. This function includes a `bool refundFee` parameter to determine whether the user is eligible for a refund on execution fees paid during the `withdraw` function call.

However, before verifying the refund eligibility, the function prematurely burns the `depositorInfo`. As a result, even if a user qualifies for a fee refund, they will never receive it. This occurs because `if (depositInfo[depositId].executionFee > usedFee)` will always evaluate to `false` since the relevant deposit information has already been deleted resulting in `depositInfo[depositId].executionFee` to always be zero.

```solidity
  function _handleReturn(uint256 withdrawn, bool positionClosed, bool refundFee) internal {
    [...]
    emit Burned(depositId, depositInfo[depositId].recipient, depositInfo[depositId].shares, amount);
    _burn(depositId);                     <@

    if (refundFee) {
      uint256 usedFee = callbackGasLimit * tx.gasprice;
      if (depositInfo[depositId].executionFee > usedFee) {
        try IGmxProxy(gmxProxy).refundExecutionFee(depositInfo[counter].owner, depositInfo[counter].executionFee - usedFee) {} catch {}
      }
    }

    // update global state
    delete swapProgressData;
    delete flowData;
    delete flow;
  }
```

## Impact

If the user was supposed to get a refund on the execution fees they paid, for example, during withdrawing from a 1x leveraged long position, they ultimately end up losing the entire execution fees for that transaction. 

## Tools Used

Manual Review 

## Recommendations

`_burn` should be called after the refund functionality

```diff
function _handleReturn(uint256 withdrawn, bool positionClosed, bool refundFee) internal {

        [...]

        emit Burned(depositId, depositInfo[depositId].recipient, depositInfo[depositId].shares, amount);
-        _burn(depositId);

        if (refundFee) {
            uint256 usedFee = callbackGasLimit * tx.gasprice;
            if (depositInfo[depositId].executionFee > usedFee) {
                try IGmxProxy(gmxProxy).refundExecutionFee(
                    depositInfo[counter].owner,
                    depositInfo[counter].executionFee - usedFee
                ) {} catch {}
            }
        }

+        _burn(depositId);

        // update global state
        delete swapProgressData;
        delete flowData;
        delete flow;
}
```

    


# Low Risk Findings

## <a id='L-01'></a>L-01. PerpetualVault withdrawals are affected by global parameter updates            



### Summary

If the protocol changes the `lockTime`, it should only apply to new deposits and not affect existing ones. If the lock period is extended, users might be forced to keep their funds locked for a longer time than originally expected, preventing planned timely withdrawals during losses or profits. Conversely, if the lock period is shortened, early withdrawals from older deposits could disrupt trading strategies, leading to forced liquidations or premature position closures.

### Impact

* **Extended Lock Period:** Users unable to withdraw during desired periods, potentially leading to forced losses or reduced profits.
* **Reduced Lock Period:** Unplanned withdrawals might disrupt vault strategies, causing position liquidations or premature position closures.

### Recommendations

* Implement logic to ensure `lockTime` changes only apply to future deposits.\
  add a `uint256 lockTime` variable to the `DepositInfo` struct
  and check for lock durations using each deposits respective lockTime and not the global parameter.

```solidity
function withdraw(address recipient, uint256 depositId) public payable nonReentrant {
        _noneFlow();
        flow = FLOW.WITHDRAW;
        flowData = depositId;

        if (recipient == address(0)) {
            revert Error.ZeroValue();
        }
        //Use local lockTime parameter
        if (depositInfo[depositId].timestamp + depositInfo[depositId].lockTime >= block.timestamp) {
            revert Error.Locked();
        }
        [...]
}
```

## <a id='L-02'></a>L-02. Cancelling a Flow after a Position Is Created Might Result in Inflation/Deflation of Shares            



## Vulnerability Details

In `PerpetualVault` a deposit or withdrawal flow can be cancelled via Keepers calling the `cancelFlow` function. The `cancelFlow` has the `gmxLock` modifier meaning that `cancelFlow` can be called anytime but not when gmx order request is on-going. Before \_createIncrease/DecreasePosition and after it's executed fully.

But this could lead to situations where a flow is cancelled after a order is executed fully. And that might lead to some unintended results.

* During a deposit flow if the flow is cancelled after the execution of `_createIncreasePosition` it could result in a inflation in the value of shares if the vault has enough collateral tokens acquired form the last action. Since cancelling the flow in a situation like that would result in a refund of the user's collateral but it does not decrease the `totalShares` value. Meaning that although the user is now not a part of the vault the shares minted to them still remain in the `totalShares`. Resulting in inflation in the value of shares.
* During a withdrawal flow if the flow is cancelled after the execution of `_createDecreasePosition`, it could result in a situation where the position is not entirely closed and the vault now has the user's collateral that is to be returned plus any collateral amount resulting from the last action / settle action. But if the user calls `withdraw` again in this situation, the withdraw flow will execute resulting in another  position of the withdrawer's amount being closed from the active position, and then the amount calculated in `_handleReturn` would result in a withdrawal amount a bit higher than during the previous withdrawal flow. Because the current `balanceBeforeWithdrawal` would not only include any collateral generated from last action and settle but also the collateral withdrawn during the previously cancelled flow.

## Impact

During a deposit flow it could result in a inflation in share value  and a deflation in share value during a withdrawal flow

## PoC

### During a deposit flow, the flow of action would be something like this:

```solidity
    function test_CancelDeposit_Call_After_Pos_Increases() external {
        //setup
        address alice = makeAddr("alice");
        payable(alice).transfer(1 ether);
        depositFixture(alice, 1e12);

        MarketPrices memory prices = mockData.getMarketPrices();
        bytes[] memory data = new bytes[](1);
        data[0] = abi.encode(3380000000000000);
        address keeper = PerpetualVault(vault).keeper();
        vm.prank(keeper);
        PerpetualVault(vault).run(true, false, prices, data);//1x SHORT 
        GmxOrderExecuted(true);
        vm.prank(keeper);
        PerpetualVault(vault).runNextAction(prices, data);
        emit log_named_decimal_uint("Vault shares before cancel flow", PerpetualVault(vault).totalShares(), 8);
        //

        //A withdraw action before the deposit action causes enough indexTokens to enter the vault due to settle/ADL action

        //Someone deposits to the open position after that 
        deal(alice, 1 ether);
        depositFixture(alice, 1e8);
        vm.prank(keeper);
        PerpetualVault(vault).runNextAction(prices, data);
        deal(address(PerpetualVault(vault).collateralToken()), vault, 1e10);//indexTokens Swapped to collateral during INCREASE_ACTION
        GmxOrderExecuted(true); //increase position completed

        (PerpetualVault.NextActionSelector selector,) = PerpetualVault(vault).nextAction();
        assertEq(uint8(selector), 6);

        bool isLock = PerpetualVault(vault).isLock();
        assertEq(isLock, false);

        uint256 ethBalBefore = alice.balance;
        //keeper calls cancelFlow after a position is executed 
        //Keeper calls cancel flow after the next deposit has already increased the position, it passes because the collateral tokens generated in vault from the last withdraw action are enough to suffice the deposit amount of the new deposit
        vm.prank(keeper);
        PerpetualVault(vault).cancelFlow();
        assertTrue(ethBalBefore < alice.balance);

        (selector,) = PerpetualVault(vault).nextAction();
        assertEq(uint8(selector), 6);
        // uint8 isNextAction = uint8(PerpetualVault(vault).isNextAction());
        // assertEq(isNextAction, 6);
        uint8 flow = uint8(PerpetualVault(vault).flow());
        assertEq(flow, 5);
        emit log_named_decimal_uint("Vault Shares after cancel flow", PerpetualVault(vault).totalShares(), 8);
    }
```

```diff
Logs:
Vault shares before cancel flow: 1000000000000.00000000
Vault Shares after cancel flow: 1000099186888.80175813
```

We notice that even though the deposit was cancelled successfully the vault generates extra shares, this would result in inflation in share value during future withdrawals since we withdraw pro-rata to the shares owned `shares / totalShares` leaving unclaimable collateral becuase of the extra deposit amount that came in even though the flow was cancelled.

### During Withdrawal flow :

```Solidity
    function test_CancelWithdraw_Call_After_Pos_Decreases() external {
        console.log("Alice Deposits to the Vault"); 
        IERC20 collateralToken = PerpetualVault(vault).collateralToken();
        address keeper = PerpetualVault(vault).keeper();
        address alice = makeAddr("alice");
        depositFixture(alice, 1e12);
        (,uint256 aliceShares,,,,) = PerpetualVault(vault).depositInfo(1);
        emit log_named_decimal_uint("Shares Minted when no pos open", aliceShares, 8);
        emit log_named_decimal_uint("Total Collateral In Vault Initially", collateralToken.balanceOf(vault), 6); 
        

        MarketPrices memory prices = mockData.getMarketPrices();
        bytes[] memory data = new bytes[](1);
        data[0] = abi.encode(3380000000000000);
        vm.prank(keeper);
        PerpetualVault(vault).run(true, false, prices, data);//1x SHORT POSITION
        PerpetualVault.FLOW flow = PerpetualVault(vault).flow();
        assertEq(uint8(flow), 2);
        GmxOrderExecuted(true);
        vm.prank(keeper);
        PerpetualVault(vault).runNextAction(prices, new bytes[](0));

        ////Bob also deposits ///////////////////////////////////////////////
        console.log("Bob deposits during an open position");
        address bob = makeAddr("bob");
        deal(bob, 1 ether); //to cover for execution fees
        depositFixture(bob, 1e12);
        (PerpetualVault.NextActionSelector selector,) = PerpetualVault(vault).nextAction();
        assertEq(uint8(selector), 1);//INCREASE ACTION
        //keeper for nextAction call 
        vm.prank(keeper);
        PerpetualVault(vault).runNextAction(prices, data);
        GmxOrderExecuted(true);//callback
        (selector,) = PerpetualVault(vault).nextAction();
        assertEq(uint8(selector), 6);//FINALIZE ACTION
        //keeper for nextAction call 
        vm.prank(keeper);
        PerpetualVault(vault).runNextAction(prices, new bytes[](0));
        ////////////////////////////////////////////////////////////////////////////////
        //ALice Withdraws
        console.log("Alice tries to withdraw");
        uint256 balanceBeforeWithdrawal = collateralToken.balanceOf(vault);
        uint256 totalShares = PerpetualVault(vault).totalShares();
        uint256[] memory depositIds = PerpetualVault(vault).getUserDeposits(alice);
        uint256 executionFee = PerpetualVault(vault).getExecutionGasLimit(false);

        uint256 lockTime = 1;
        PerpetualVault(vault).setLockTime(lockTime);
        vm.warp(block.timestamp + lockTime + 1);
        payable(alice).transfer(1 ether);
        vm.prank(alice);
        PerpetualVault(vault).withdraw{value: executionFee * tx.gasprice}(alice, depositIds[0]);

        GmxOrderExecuted(true);//settle callback

        (selector,) = PerpetualVault(vault).nextAction();
        assertEq(uint8(selector), 3);//WITHDRAW ACTION

        bytes[] memory swapData = new bytes[](2);
        swapData[0] = abi.encode(3390000000000000);
        vm.prank(keeper);
        PerpetualVault(vault).runNextAction(prices, swapData);

        GmxOrderExecuted(true);//market decrease callback
        uint256 withdrawAmount = collateralToken.balanceOf(vault) - balanceBeforeWithdrawal;

        bool isLock = PerpetualVault(vault).isLock();
        assertEq(isLock, false);
        console.log("Keeper cancels withdraw after the market position is decreased");
        //keeper calls cancelFlow after a position is executed 
        vm.prank(keeper);
        PerpetualVault(vault).cancelFlow();
        vm.prank(keeper);
        PerpetualVault(vault).runNextAction(prices, new bytes[](0));//finalize
        //emit log_named_decimal_uint("Total Collateral In Vault When withdraw flow cancelled after pos decrased", collateralToken.balanceOf(vault), 6); 
        uint256 refundAmountInCancelFlow = withdrawAmount + balanceBeforeWithdrawal * aliceShares / totalShares;
        emit log_named_decimal_uint("Refund Amount for Alice When Flow Cancelled", refundAmountInCancelFlow, 6);       

        //Alice withdraws again since the withdraw was cancelled 
        console.log("Alice Withdraws again after her withdraw was cancelled");
        balanceBeforeWithdrawal = collateralToken.balanceOf(vault);
        vm.prank(alice);
        PerpetualVault(vault).withdraw{value: executionFee * tx.gasprice}(alice, depositIds[0]);

        GmxOrderExecuted(true);//settle callback
        (selector,) = PerpetualVault(vault).nextAction();
        assertEq(uint8(selector), 3);//WITHDRAW ACTION

        vm.prank(keeper);
        PerpetualVault(vault).runNextAction(prices, swapData);

        GmxOrderExecuted(true);//market decrease callback
        withdrawAmount = collateralToken.balanceOf(vault) - balanceBeforeWithdrawal;
        //emit log_named_decimal_uint("Total Collateral In Vault after second withdraw call", collateralToken.balanceOf(vault), 6);
        uint256 refundAmount= withdrawAmount + balanceBeforeWithdrawal * aliceShares / totalShares;
        emit log_named_decimal_uint("Refund Amount When calling Withdrawal again", refundAmount, 6);  
        vm.prank(keeper);
        PerpetualVault(vault).runNextAction(prices, new bytes[](0));//FINALIZE
    }
```

```diff
Logs:
Alice Deposits to the Vault
Shares Minted when no pos open: 1000000000000.00000000
Total Collateral In Vault Initially: 1000000.000000
Bob deposits during an open position
Alice tries to withdraw
Keeper cancels withdraw after the market position is decreased
Refund Amount for Alice When Flow Cancelled: 994108.906352
Alice Withdraws again after her withdraw was cancelled
Refund Amount When calling Withdrawal again: 995377.193831
```

Here we notice, if the withdrawal flow is cancelled after a position has been decreased, and then the withdrawer  tries to withdraw again they receive more collateral for the same amount of tokens. Resulting in deflation of share value.

## Tools Used

Manual  Review\
Foundry

## Recommendations

The only recommendation I could think of is to allow keepers to only call `cacelFlow` before `_createIncreasePosition` / `_createDecreasePosition` is executed and not after that. This could be done by adding a flag variable such as `bool positionExecuted` that keeps track of the position execution during a single flow and allowing the keepers to enter `cancelFlow` only when `positionExecuted` is `false`

#### Note :- I consider this a valid issue because it does not stem from a scenario where the keeper loses functionality or behaves maliciously. The keeper is executing its role as intended. Instead, the root cause lies in a fundamental logical flaw in how `cancelFlow` operates. This design flaw can lead to unintended consequences, regardless of the keeper's trustability (hence why the low), making it a issue that needs to be addressed.



