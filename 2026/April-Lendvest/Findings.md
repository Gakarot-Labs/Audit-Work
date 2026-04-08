# Title
Greedy order matching allows deterministic epoch start reversion leading to Protocol DoS

## Summary
The order matching engine performs greedy borrower lender matching without enforcing collateral lender capacity during the matching process. As a result, oversized borrower and lender orders can be matched even when insufficient collateral lender liquidity exists. 

The capacity check occurs only after matching, causing `startEpoch()` to revert. An attacker can repeatedly submit such orders to deterministically trigger epoch start failures, disrupting protocol liveness.

## Root Cause
The root cause lies in the order matching logic inside `tryMatchOrders()`, where borrower and lender orders are greedily matched before verifying whether sufficient collateral lender liquidity exists.

During the matching loop, the protocol continuously increases the total borrow amount matched:
```solidity
borrowAmountMatched += scaledAmountToBorrow;
```
This occurs without enforcing the collateral lender capacity constraint during the matching process. Than oversized borrower orders can be matched even when the system lacks sufficient collateral lender backing.

The collateral lender capacity check is only performed after the entire matching loop completes:
```solidity
if (((borrowAmountMatched * 1e18) / epochStartRedemptionRate) / inverseBorrowCLAmount > totalCollateralLenderCT) {
    revert VaultLib.InsufficientFunds();
}
```
Because this validation happens post matching, the system may fully match borrower and lender orders that exceed the available collateral lender liquidity. When this condition is detected, `startEpoch()` reverts via `tryMatchOrders()`, preventing the epoch from starting.

Thus, the mismatch between greedy order matching and late collateral capacity validation allows attacker controlled orders to deterministically trigger epoch start failures, resulting in a protocol liveness disruption.

[Code Reference](https://github.com/Lendvest/lendvest-smart-staking/blob/81df1a46811776cbca4db2b37bd401da1f2b4483/src/LVLidoVault.sol#L294-L487)

## Impact
The malicious orders remain in the orderbook, subsequent attempts to start the epoch will continue to revert, preventing the protocol from progressing to a new epoch.

This enables a repeatable DoS where the attacker can repeatedly block epoch initialization, disrupting protocol operations and preventing legitimate user orders from being processed until the malicious orders are removed or additional collateral lender liquidity is introduced.

## Mitigation
Instead of performing a post loop validation, the protocol should calculate the maximum borrow capacity backed by collateral lenders before matching begins and ensure the matching process never exceeds this limit.
```solidity
uint256 maxBorrowCapacity =
    (totalCollateralLenderCT * inverseBorrowCLAmount * epochStartRedemptionRate) / 1e18;
```
Then enforce this constraint inside the matching loop:
```solidity
if (borrowAmountMatched + scaledAmountToBorrow > maxBorrowCapacity) {
    break; // stop matching once CL capacity is reached
}
```
This approach ensures that:
- Matching stops once collateral lender capacity is reached.
- `startEpoch()` can proceed without reverting.
- Excess borrower and lender orders remain in the orderbook for future epochs instead of blocking the protocol.

By making the matching logic capacity aware rather than revert based, the protocol prevents deterministic epoch start failures and eliminates the griefing vector.

## POC
Paste this test in `lendvest-smart-staking/test/mainnet/E2EEpochLifecycleMainnet.t.sol`

Runnable Command: `forge test --match-test test_MyTest --fork-url $RPC_URL -vvvvv`

```solidity

    function test_MyTest() public {

        // close current epoch
        vm.prank(address(vaultUtil));
        vault.end_epoch();

        // bypass cooldown
        vm.warp(block.timestamp + 2 days);

        // small CL liquidity
        _fundCollateralLender(collateralLender1, 0.01 ether);

        // attacker orders
        _fundBorrower(borrower1, 0.2 ether);
        _fundLender(lender1, 0.2 ether);

        console.log("CL:", vault.totalCollateralLenderCT());

        vm.expectRevert();

        vm.prank(owner);
        vault.startEpoch();
    }
```

# Title
Incorrect collateral injection allows multiple protection tranches to be marked consumed while only one is injected

## Summary
The vault’s collateral protection logic incorrectly accounts for tranche usage during price drawdowns. When multiple thresholds are crossed in a single upkeep cycle, the system marks multiple collateral lender tranches as consumed but injects collateral equivalent to only a single tranche.

This creates a mismatch between recorded protection and actual collateral added. As a result, protection layers can be prematurely exhausted, **exposing the vault to liquidation even though unused collateral lender funds remain available.**

## Root Cause
The vulnerability originates from a mismatch between tranche accounting and collateral injection logic inside the automation function `performUpkeep()` in `LVLidoVaultUtil`.

When the market price drops significantly within a single block, the system may detect that multiple collateral protection thresholds have been crossed. This causes the contract to calculate `tranchesToTrigger`, representing how many collateral protection layers should be activated.

However, the contract only injects collateral equivalent to a single tranche, while updating the state as if multiple tranches were used.

```solidity
// Vulnerable Code
uint256 remainingTranches = MAX_TRANCHES - LVLidoVault.collateralLenderTraunche();
if (remainingTranches == 0) remainingTranches = 1;

uint256 collateralToAddToPreventLiquidation =
    LVLidoVault.totalCLDepositsUnutilized() / remainingTranches;

if (collateralToAddToPreventLiquidation > 0) {
    LVLidoVault.avoidLiquidation(collateralToAddToPreventLiquidation);
    LVLidoVault.setCollateralLenderTraunche(
        LVLidoVault.collateralLenderTraunche() + tranchesToTrigger
    );
    LVLidoVault.setCurrentRedemptionRate(newRedemptionRate);
}
```

Why the Bug Happens:

The function correctly calculates how many thresholds were crossed:
`tranchesToTrigger`

But the collateral injected is calculated using:
`uint256 collateralToAddToPreventLiquidation = LVLidoVault.totalCLDepositsUnutilized() / remainingTranches;`
which represents only one tranche worth of collateral.

Meanwhile, the state update increments the tranche counter by:
`LVLidoVault.setCollateralLenderTraunche( LVLidoVault.collateralLenderTraunche() + tranchesToTrigger);`

As a result, the vault may prematurely exhaust its protection layers, leading to potential liquidation even though unused collateral lender funds still remain in the system.

[Code Reference](https://github.com/Lendvest/lendvest-smart-staking/blob/81df1a46811776cbca4db2b37bd401da1f2b4483/src/LVLidoVaultUtil.sol#L212-L302)

## Impact
- Multiple tranches can be marked as consumed while only one tranche worth of collateral is injected.
- The vault may become underprotected and get liquidated even though collateral lender funds still remain unused. Borrowers and collateral lenders may suffer losses due to unnecessary liquidation triggered by insufficient injected collateral.
- An attacker observing a rapid price movement could wait for automation to execute the faulty tranche logic and then trigger liquidation on Ajna while protection funds remain unused.

## Mitigation
Inject collateral proportional to the number of triggered tranches.

```solidity
uint256 remainingTranches = MAX_TRANCHES - LVLidoVault.collateralLenderTraunche();
if (remainingTranches == 0) remainingTranches = 1;

uint256 trancheSize =
    LVLidoVault.totalCLDepositsUnutilized() / remainingTranches;

// Inject collateral proportional to triggered tranches
uint256 collateralToAddToPreventLiquidation =
    trancheSize * tranchesToTrigger;

if (collateralToAddToPreventLiquidation > 0) {
    LVLidoVault.avoidLiquidation(collateralToAddToPreventLiquidation);
    LVLidoVault.setCollateralLenderTraunche(LVLidoVault.collateralLenderTraunche() + tranchesToTrigger);
    LVLidoVault.setCurrentRedemptionRate(newRedemptionRate);
}
```

Additional Safety:
Add a validation check to prevent accounting mismatch.
```solidity
require(
    collateralToAddToPreventLiquidation <= LVLidoVault.totalCLDepositsUnutilized(),
    "Invalid tranche collateral calculation"
);
```
## POC
Make a new file in `lendvest-smart-staking/test/mainnet/TrancheBug.t.sol`

Runnable Command: `forge test --fork-url $RPC_URL --match-test test_TrancheAccountingBug -vvvvv`

**The following PoC demonstrates that when multiple tranches are triggered during a single upkeep cycle, the vault records multiple tranches as consumed while injecting collateral equivalent to only one tranche.**

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "forge-std/console.sol";

import "./BaseMainnetTest.sol";

contract TrancheBugForkTest is BaseMainnetTest {
    function test_TrancheAccountingBug() public {

        require(vault.epochStarted(), "epoch not started");
        
        // setup liquidity
        _fundLender(lender1, 10 ether);
        _fundBorrower(borrower1, 5 ether);

        // large collateral lender pool
        _fundCollateralLender(collateralLender1, 300 ether);

        uint256 trancheBefore = vault.collateralLenderTraunche();
        uint256 clBefore = vault.totalCLDepositsUnutilized();

        console.log("Tranche before:", trancheBefore);
        console.log("CL deposits before:", clBefore);

        // force artificial price drop
        // increase redemption rate so market appears down
        vm.prank(address(vaultUtil));
        vault.setCurrentRedemptionRate(2e18);

        // check automation
        (bool upkeepNeeded, bytes memory performData) = vaultUtil.checkUpkeep("");

        console.log("Upkeep needed:", upkeepNeeded);

        require(upkeepNeeded, "automation not triggered");

        //execute upkeep
        vm.prank(vaultUtil.s_forwarderAddress());
        vaultUtil.performUpkeep(performData);

        uint256 trancheAfter = vault.collateralLenderTraunche();
        uint256 clAfter = vault.totalCLDepositsUnutilized();

        uint256 collateralInjected = clBefore - clAfter;

        console.log("Tranche after:", trancheAfter);
        console.log("Collateral injected:", collateralInjected);

        // expected tranche size
        uint256 MAX_TRANCHES = vaultUtil.MAX_TRANCHES();

        uint256 remainingTranches = MAX_TRANCHES - trancheBefore;

        uint256 trancheSize = clBefore / remainingTranches;

        uint256 logicalTranchesUsed = collateralInjected / trancheSize;

        uint256 recordedTranchesUsed = trancheAfter - trancheBefore;

        console.log("Logical tranches:", logicalTranchesUsed);
        console.log("Recorded tranches:", recordedTranchesUsed);

        // Bug
        assertTrue(
            recordedTranchesUsed > logicalTranchesUsed,
            "Bug: multiple tranches consumed but collateral injected only for one tranche"
        );
    }
}

```

# Title
Missing state update in getRate() allows repeated performTask() calls to spam chainlink requests and drain LINK subscription

## Summary
The `getRate()` function sends a Chainlink Functions request but does not update `updateRateNeeded` after submission. As a result, the upkeep condition remains true until the oracle response arrives. Because `performTask()` is permissionless, attackers can repeatedly trigger it to submit multiple oracle requests in the same state. This enables LINK subscription draining and can halt protocol automation when the subscription balance is exhausted.

## Root Cause
The vulnerability arises because `getRate()` does not update `updateRateNeeded` after submitting a Chainlink Functions request, leaving the upkeep condition valid. Since `performTask()` is permissionless and internally bypasses the `onlyForwarder` restriction, attackers can repeatedly trigger `performUpkeep()` and submit multiple oracle requests before the previous request is fulfilled.

In `checkUpkeep()`, the protocol determines whether a rate update is required when the term has ended and debt remains:
```solidity
if (updateRateNeeded && block.timestamp > (LVLidoVault.epochStart() + LVLidoVault.termDuration()) && debt > 0) {
    upkeepNeeded = true;
    performData = abi.encode(221); // Task ID 221: Get new rate
}
```
Since `updateRateNeeded` remains true, the upkeep condition continues to return true.

When the upkeep executes, `performUpkeep()` triggers the rate request:
```solidity
if (taskId == 221 && updateRateNeeded && t1Debt > 0) {
    getRate();
    emit VaultLib.TermEnded(LVLidoVault.epochStart() + LVLidoVault.termDuration());
    return;
}
```

The `getRate()` function sends the Chainlink request but fails to disable the update flag after submission:
```solidity
function getRate() internal {
    s_requestCounter = s_requestCounter + 1;

    try i_router.sendRequest(
        s_subscriptionId,
        s_requestCBOR,
        FunctionsRequest.REQUEST_DATA_VERSION,
        s_fulfillGasLimit,
        VaultLib.donId
    ) returns (bytes32 requestId_) {
        s_lastRequestId = requestId_;
        emit RequestSent(requestId_);
    }
}
```
Because `updateRateNeeded` is only set to false in the oracle callback(fulfillRequest) or in error cases, the system remains in a state where checkUpkeep() continues to return true until the asynchronous response arrives.

[Code Reference](https://github.com/Lendvest/lendvest-smart-staking/blob/81df1a46811776cbca4db2b37bd401da1f2b4483/src/LVLidoVaultUtil.sol#L106-L211)

[Code Reference](https://github.com/Lendvest/lendvest-smart-staking/blob/81df1a46811776cbca4db2b37bd401da1f2b4483/src/LVLidoVaultUtil.sol#L212-L302)

[Code Reference](https://github.com/Lendvest/lendvest-smart-staking/blob/81df1a46811776cbca4db2b37bd401da1f2b4483/src/LVLidoVaultUtil.sol#L367-L386)

## Impact
### Attacker Cost vs Protocol Loss

From the PoC execution:
Gas used = 1,350,259 gas for 3 calls
Current gas price ≈ 0.096 gwei
ETH price ≈ $2050
Chainlink price ≈ $9
Chainlink Functions request cost = 0.4157 LINK per request

**Attacker Gas Cost(per call):**
Gas per call = 1,350,259 / 3 = 450,086 gas  
Gas cost = 450,086 × 0.096 gwei = 43,208 gwei
Convert to ETH: 43,208 gwei = 0.0000432 ETH
Convert to USD: 0.0000432 × 2050 ≈ $0.088
Attacker cost per call ≈ $0.088
[Gas Reference](https://etherscan.io/gastracker)

**Protocol Loss (Chainlink Request cost per call):**
Each oracle request costs: 0.4157 LINK
Convert to USD: 0.4157 × $9 ≈ $3.74
Protocol loss per request ≈ $3.74
[Chainlink Reference](https://docs.chain.link/chainlink-functions/resources/billing)


| Spam Calls | Attacker Cost (USD) | LINK Burned | Protocol Loss (USD) |
|------------|---------------------|-------------|---------------------|
| 1 call     |       $0.09         | 0.4157 LINK |      $3.74          |
| 10 calls   |       $0.90         | 4.157 LINK  |      $37.4          |
| 100 calls  |        $9           | 41.57 LINK  |       $374          |
| 500 calls  |        $45          | 207.85 LINK |      $1,870         |


**Cost Amplification:**
Protocol loss / attacker cost = 3.74 / 0.09 ≈ 41X
So the protocol loses ~41X more value than the attacker spends.

## Mitigation
Update the state before sending the request so the upkeep condition becomes false while the oracle request is pending.
```solidity
function getRate() internal {
    s_requestCounter = s_requestCounter + 1;

    try i_router.sendRequest(
        s_subscriptionId,
        s_requestCBOR,
        FunctionsRequest.REQUEST_DATA_VERSION,
        s_fulfillGasLimit,
        VaultLib.donId
    ) returns (bytes32 requestId_) {

        updateRateNeeded = false; // lock after successful submission
        s_lastRequestId = requestId_;
        emit RequestSent(requestId_);

    } catch {
        updateRateNeeded = true;
    }
}
```

## POC

Paste this test in `lendvest-smart-staking/test/mainnet/PublicPerformTaskMainnet.t.sol`

Runnable Command `forge test --fork-url $RPC_URL --match-test test_RequestSpamPossible -vvvv`

```solidity
    function test_RequestSpamPossible() public {

        vm.warp(vault.epochStart() + vault.termDuration() + 1);

        // Read initial counter
        uint256 beforeCounter = vaultUtil.s_requestCounter();

        console.log("Initial request counter:", beforeCounter);

        uint256 gasStart = gasleft();

        // Simulate multiple attackers calling performTask
        address attacker = makeAddr("attacker1");

        vm.prank(attacker);
        try vaultUtil.performTask() {} catch {}

        vm.prank(attacker);
        try vaultUtil.performTask() {} catch {}

        vm.prank(attacker);
        try vaultUtil.performTask() {} catch {}

        uint256 gasUsed = gasStart - gasleft();

        uint256 afterCounter = vaultUtil.s_requestCounter();

        console.log("Final request counter:", afterCounter);

        if (afterCounter > beforeCounter + 1) {
            console.log("Multiple requests triggered from same state");
        }
        console.log("Total gas used:", gasUsed);
    }
```

# Title
