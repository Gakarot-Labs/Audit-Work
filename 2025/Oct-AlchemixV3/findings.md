# Title(Low) --> Incorrect APY calculation in MYTStrategy::_approxAPY() causes underreported yields

## Brief/Intro
`MYTStrategy::_approxAPY()` incorrectly divides the squared APR term by `2 * SECONDS_PER_YEAR`, instead of just 2, effectively nullifying compounding growth in APY approximation. This leads to misreported yield rates across strategies.

## Impact Details
APY and APR become near identical, misleading users and allocators relying on the `snapshotYield()` results for decision making or display.

## References
Link to Code ---> https://github.com/alchemix-finance/v3-poc/blob/b2e2aba046c36ff5e1db6f40f399e93cd2bdaad0/src/MYTStrategy.sol#L230-L234

## Mitigation
To correct the APY miscalculation, the squared APR term should not be divided by SECONDS_PER_YEAR. The denominator should only be 2, restoring the correct compounding approximation formula. Specifically, in `MYTStrategy::_approxAPY()`, replace:

```solidity
-return apr + aprSq / (2 * SECONDS_PER_YEAR);
+return apr + aprSq / 2;

```

## Proof of Concept
Paste this test in file location: v3-poc/src/test/MYTStrategy.t.sol
Runnable Command: MAINNET_RPC_URL="https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY" forge test --mt test_POC_IncorrectAPYCalculation -vvvvv
Note: This PoC runs on a local Foundry mainnet fork using a read only Alchemy RPC key.

```solidity
function test_POC_IncorrectAPYCalculation() public view {
        // Setup a base per-second rate that equals roughly 5% APR
        uint256 ratePerSecWad = 1.585e9; // 0.05 / 31_557_600 * 1e18

        // Simulate what snapshotYield() would call internally
        uint256 secondsPerYear = strategy.SECONDS_PER_YEAR();
        uint256 apr = ratePerSecWad * secondsPerYear;
        uint256 aprSq = (apr * apr) / 1e18;

        // Buggy formula used in strategy
        uint256 buggyApy = apr + (aprSq / (2 * secondsPerYear));

        // Correct compounding approximation
        uint256 correctApy = apr + (aprSq / 2);

        console.log("Proof of Incorrect APY Calculation");
        console.log("Base APR (should be ~5%):", apr);
        console.log("Buggy APY (contract formula):", buggyApy);
        console.log("Correct APY (mathematical):", correctApy);

    }

```

# Title(Low) --> Ownership transfer flow design prevents pending admin from accepting control

## Brief/Intro
The contract’s ownership transfer logic requires the current admin to finalize the transfer instead of allowing the pending admin to accept it. This breaks expected `2 step secure` pattern (propose + accept by new admin) ownership pattern and can temporarily halt governance actions until the current admin completes the process.

## Vulnerability Details
In `AlchemistCurator`, the `acceptAdminOwnership()` function uses the `onlyAdmin` modifier, allowing only the current admin to finalize ownership transfers. As a result, the pending admin cannot accept ownership independently. This deviates from the standard two step transfer pattern and can cause delays or operational lock if the current admin is unavailable to finalize the process.

## Impact Details
The bug disrupts the normal ownership transfer process, temporarily preventing new admins from assuming control. Until the current admin finalizes the transfer, all privileged governance actions such as managing strategies or adjusting vault parameters remain blocked, reducing protocol responsiveness and operational flexibility.

## References
Link to code ----> https://github.com/alchemix-finance/v3-poc/blob/b2e2aba046c36ff5e1db6f40f399e93cd2bdaad0/src/AlchemistCurator.sol#L31-L35

## Mitigation
Allow the pending admin to call `acceptAdminOwnership()` by replacing the `onlyAdmin` modifier with a check like 
```solidity
require(msg.sender == pendingAdmin && pendingAdmin != address(0))
```



## Proof of Concept

- Paste this test in File Location: v3-poc/src/test/AlchemistCurator.t.sol
- Runnable Command: forge test --mt test_OwnershipTransferBug -vvvvv

```solidity
    function test_OwnershipTransferBug() public {
        vm.startPrank(admin);
        address newAdmin = makeAddr("newAdmin"); // New admin address
        mytCuratorProxy.transferAdminOwnerShip(newAdmin); // Transfer ownership to newAdmin
        vm.stopPrank();

        // Try to accept ownership as newAdmin --> will revert
        vm.startPrank(newAdmin);
        vm.expectRevert();
        mytCuratorProxy.acceptAdminOwnership();
        vm.stopPrank();

        // Verify that privileged calls still fail for newAdmin (shows ownership not updated)
        vm.startPrank(newAdmin);
        vm.expectRevert();
        mytCuratorProxy.submitDecreaseAbsoluteCap(address(mytStrategy), 1);
        vm.stopPrank();
    }
```

# Title(Low) --> Stale yield reporting in MYTStrategy::getEstimatedYield() returns static initialization value instead of actual computed yield

## Brief/Intro

The `MYTStrategy::getEstimatedYield()` returns `params.estimatedYield` (constructor only value) instead of the latest snapshotted yield `estApy`, causing permanently stale APY reporting.

## Vulnerability Details

- `params.estimatedYield` is initialized from `_params` in the constructor and never written to again anywhere in the codebase.
- `snapshotYield()` writes runtime yield into `estApy` (estApr/estApy), but `getEstimatedYield()` reads the wrong storage slot `params.estimatedYield` instead of estApy.
- Docstring claims this returns the "current snapshotted estimated yield", but implementation mismatches docs.

## Impact Details

Because `getEstimatedYield()` always returns a static constructor value, the strategy continuously exposes misleading yield data to external systems. This results in incorrect allocation decisions, confused or inaccurate user interfaces, and inefficient capital utilization, as the reported yield no longer matches the strategy’s real on chain performance.

## References

Link to code --> https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/MYTStrategy.sol#L284-L286

## Mitigation

In `MYTStrategy::getEstimatedYield()`, replace:

```solidity
-return params.estimatedYield;
+return estApy;

```

## Proof of Concept

**Paste this test in File Location:** v3-poc/src/test/MYTStrategy.t.sol

**Runnable Command:** MAINNET_RPC_URL="https://eth-mainnet.g.alchemy.com/v2/<YOUR_KEY>" forge test --mt test_poc_StaleEstimatedYieldGetter -vvvvv

```solidity
    function test_poc_StaleEstimatedYieldGetter() public {
        // Confirm initial state
        uint256 initialParamYield = strategy.getEstimatedYield();
        console.log("Initial getEstimatedYield() =", initialParamYield);

        // Snapshot yield (this updates estApy internally)
        vm.startPrank(admin);
        uint256 snapYield = strategy.snapshotYield();
        console.log("Snapshot yield returned =", snapYield);
        (uint256 apr, uint256 apy) = (strategy.estApr(), strategy.estApy());
        console.log("apr:", apr, "apy:", apy);
        vm.stopPrank();

        // Fetch getter value again
        uint256 afterSnapGetter = strategy.getEstimatedYield();
        console.log("After snapshot getEstimatedYield() =", afterSnapGetter);

        // They should be different if getEstimatedYield() reflected estApy,
        // but will be same (stale) because it returns params.estimatedYield
        assertEq(afterSnapGetter, initialParamYield, "getEstimatedYield() is updating unexpectedly");

        // Also check that snapshot yield and getter mismatch proving stale data
        assertTrue(afterSnapGetter != snapYield, "getEstimatedYield() should not equal snapshot yield");
    }

```

