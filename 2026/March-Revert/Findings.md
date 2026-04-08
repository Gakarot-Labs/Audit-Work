# Title

Transform replacement missing _removeTokenFromOwner breaks vault ownership invariant and creates phantom tokens

## Summary 

The transform replacement flow registers the new tokenId via `_addTokenToOwner` but fails to remove the replaced `oldTokenId` from the owner enumeration. As a result, the vault retains a stale token entry, creating phantom tokens in `ownedTokens` and **breaking the vault’s ownership accounting invariant between internal state and actual NFT custody**.

## Root Cause

During a transform that replaces an NFT(oldTokenId --> tokenId), the vault registers the new token in the owner’s enumeration but fails to remove the replaced token from the owner’s token list.

In the `onERC721Received` transform replacement branch, the contract adds the new token to the owner but does not mirror the removal of the old token from `ownedTokens`.

```solidity
address owner = tokenOwner[oldTokenId];

// set transformed token to new one
transformedTokenId = tokenId;

uint256 debtShares = loans[oldTokenId].debtShares;

// copy debt to new token
loans[tokenId] = Loan(debtShares);

_addTokenToOwner(owner, tokenId);
emit Add(tokenId, owner, oldTokenId);

// remove debt from old loan
_cleanupLoan(oldTokenId, debtExchangeRateX96, lendExchangeRateX96);
```
Here `_addTokenToOwner(owner, tokenId)` registers the new token, but `_removeTokenFromOwner(owner, oldTokenId)` is never called.

Because of this, the old token remains in:
```solidity
ownedTokens[owner]
tokenOwner[oldTokenId]
```
even though the underlying NFT has been replaced or burned by the transformer.

As a result, the vault’s internal ownership registry becomes desynchronized from the actual NFT custody, leaving stale entries that appear as valid loans in `loanCount()` and `loanAtIndex()` enumeration.

[Code Reference](https://github.com/revert-finance/lend/blob/2bb022ee862c0f7f2010505f4d697f33827312ef/src/V3Vault.sol#L393-L466)
[Code reference](https://github.com/revert-finance/lend/blob/2bb022ee862c0f7f2010505f4d697f33827312ef/docs/agents/protocol-primer.md?plain=1#L62)

## Impact
A token replacement during `transform()` registers the new NFT but fails to remove the replaced token from the owner’s enumeration, leaving a ghost collateral entry in the vault’s ownership state. As a result, the vault reports collateral that no longer exists, causing `loanCount()` and `loanAtIndex()` to return invalid token IDs and permanently desynchronizing vault accounting from the real collateral held by the protocol

**Likelihood:** High because replacement transforms are a supported and normal workflow, allowing any user performing such a transform to reliably trigger the issue.

**Severity:** High because the bug corrupts the vault’s core collateral ownership accounting, violating a fundamental protocol invariant and leaving persistent ghost entries in protocol state.

## Mitigation
When a position NFT is replaced during a transform, the vault must atomically migrate ownership state by removing the replaced token (oldTokenId) from the owner's enumeration before registering the new token (tokenId).

```solidity
function onERC721Received(
    {
                ....................

                address owner = tokenOwner[oldTokenId];

                // set transformed token to new one
                transformedTokenId = tokenId;

                uint256 debtShares = loans[oldTokenId].debtShares;

                // copy debt to new token
                loans[tokenId] = Loan(debtShares);

>@              //FIX: remove old token from enumeration
                _removeTokenFromOwner(owner, oldTokenId);

                _addTokenToOwner(owner, tokenId);
                emit Add(tokenId, owner, oldTokenId);

                ......................
    }
```

## POC
Add this test in lend/test/integration/aerodrome/V3VaultTransformPlanTests.t.sol
Runnable Command: forge test mt testTransformReplacementCreatesGhostTokenInEnumeration -vvvvv

```solidity
    function testTransformReplacementCreatesGhostTokenInEnumeration() public {
        MockTokenIdMigrator migrator = new MockTokenIdMigrator(npm);
        vault.setTransformer(address(migrator), true);

        uint256 tokenId = createPosition(alice, address(usdc), address(dai), 500, -100, 100, 1000e18);

        vm.startPrank(alice);
        npm.approve(address(vault), tokenId);
        vault.create(tokenId, alice);
        vault.approveTransform(tokenId, admin, true);
        vm.stopPrank();

        oracle.setMaxPoolPriceDifference(type(uint16).max);

        bytes memory transformData = abi.encodeWithSelector(MockTokenIdMigrator.migrate.selector, tokenId);

        vm.prank(admin);
        uint256 newTokenId = vault.transform(tokenId, address(migrator), transformData);

        // sanity checks
        assertTrue(newTokenId != tokenId);
        assertEq(vault.ownerOf(newTokenId), alice);

        // Invariant check: enumeration should only contain the new token
        uint256 count = vault.loanCount(alice);

        // Expected: only 1 active token after replacement
        // Actual Bug: oldTokenId remains i.e count = 2
        assertEq(count, 2, "replacement leaves ghost token in enumeration");

        uint256 token0 = vault.loanAtIndex(alice, 0);
        uint256 token1 = vault.loanAtIndex(alice, 1);

        assertTrue(token0 == tokenId || token0 == newTokenId, "unexpected token at index 0");
        assertTrue(token1 == tokenId || token1 == newTokenId, "unexpected token at index 1");
    }
```

# Title
Protocol compound reward fee cannot be increased after being lowered due to incorrect validation check

## Summary
The `setCompoundReward` function incorrectly validates the new reward fee against the current fee instead of the defined maximum limit. As a result, once the owner lowers the protocol reward fee, it becomes impossible to increase it again, permanently locking the protocol into the lowest previously set fee value.

## Root Cause
The setter checks the new fee against the current state variable rather than the intended ceiling constant.
```solidity
    function setCompoundReward(uint64 _totalRewardX64) external override onlyOwner {
        if (_totalRewardX64 > totalRewardX64) {
            revert InvalidConfig();
        }
        totalRewardX64 = _totalRewardX64;
        emit CompoundRewardUpdated(msg.sender, _totalRewardX64);
    }
```

Here the validation condition:
```solidity
if (_totalRewardX64 > totalRewardX64) {
    revert InvalidConfig();
}
```
forces the fee to only monotonically decrease, even though a maximum limit (MAX_REWARD_X64) is defined.

Because of this, any attempt to increase the fee after reducing it will revert.

[Code Reference](https://github.com/revert-finance/lend/blob/2bb022ee862c0f7f2010505f4d697f33827312ef/src/GaugeManager.sol#L292-L298)

## Impact
If governance reduces the compound reward fee(i.e. temporarily lowering protocol fees), the protocol cannot restore the fee later, even if the new value is still below the intended maximum.
This permanently traps the protocol fee at the lowest historically set value.

**Likelihood:** Medium
The setter is a normal governance function and can be triggered by routine parameter updates.

**Severity:** Medium
No direct loss of user funds, but protocol revenue can become permanently degraded.

## Mitigation
Validate the new fee against the defined ceiling instead of the current value:
```solidity
function setCompoundReward(uint64 _totalRewardX64) external override onlyOwner {
    if (_totalRewardX64 > MAX_REWARD_X64) {
        revert InvalidConfig();
    }

    totalRewardX64 = _totalRewardX64;
    emit CompoundRewardUpdated(msg.sender, _totalRewardX64);
}
```
This allows governance to both increase and decrease the fee within the intended bounds.

## POC
Paste this test in lend/test/unit/GaugeManager.t.sol
Runnable Command: forge test --mt testSetCompoundRewardCannotIncreaseAfterDecrease -vvvvv

```solidity
        function testSetCompoundRewardCannotIncreaseAfterDecrease() external {
            // read current protocol fee
            uint64 initialFee = gaugeManager.totalRewardX64();
            assertGt(initialFee, 0);

            // owner reduces fee
            uint64 reducedFee = initialFee / 2;
            gaugeManager.setCompoundReward(reducedFee);

            assertEq(gaugeManager.totalRewardX64(), reducedFee);

            // attempt to increase fee again should logically be allowed up to MAX_REWARD_X64
            uint64 newHigherFee = reducedFee + 1;

            vm.expectRevert();
            gaugeManager.setCompoundReward(newHigherFee);

            // verify fee stuck at reduced value
            assertEq(gaugeManager.totalRewardX64(), reducedFee);
        }
```

# Title
Daily Lend Increase Limit Not Rescaled After Lender Haircut Allowing Circuit Breaker Bypass

## Summary
The vault enforces a daily lend increase circuit breaker based on the total lent assets to limit rapid pool growth. 
During liquidation events that cause lender loss, the protocol reduces `lendExchangeRateX96`, which lowers the effective lent asset base. But `dailyLendIncreaseLimitLeft` is not rescaled after this haircut. As a result, attackers can deposit amounts exceeding the intended percentage cap, bypassing the circuit breaker and allowing disproportionate pool growth immediately after a liquidation event.

## Root Cause
During bad debt liquidations the vault reduces `lendExchangeRateX96`, which effectively shrinks the lender asset base(socialized loss). But the contract only rescales the daily debt limit, while `dailyLendIncreaseLimitLeft` remains unchanged, leaving it calibrated to the pre haircut pool size.

`V3Vault::_handleReserveLiquidation`
```solidity
uint256 preHaircutDailyDebtCap = _calculateDailyIncreaseLimit(
    newLendExchangeRateX96, dailyDebtIncreaseLimitMin, MAX_DAILY_DEBT_INCREASE_X32
);

// haircut reduces lender base
newLendExchangeRateX96 = (totalLent - missing) * newLendExchangeRateX96 / totalLent;
lastLendExchangeRateX96 = newLendExchangeRateX96;

// recompute debt cap
uint256 postHaircutDailyDebtCap = _calculateDailyIncreaseLimit(
    newLendExchangeRateX96, dailyDebtIncreaseLimitMin, MAX_DAILY_DEBT_INCREASE_X32
);

if (
    dailyDebtIncreaseLimitLastReset == uint32(block.timestamp / 1 days)
        && postHaircutDailyDebtCap < preHaircutDailyDebtCap
) {
    uint256 capReduction = preHaircutDailyDebtCap - postHaircutDailyDebtCap;
    dailyDebtIncreaseLimitLeft =
        capReduction >= dailyDebtIncreaseLimitLeft ? 0 : dailyDebtIncreaseLimitLeft - capReduction;
}
```
Because the lender exchange rate changes:
`lendExchangeRateX96 = newLendExchangeRateX96;`
the total lent base decreases, but `dailyLendIncreaseLimitLeft` is not recalculated, allowing deposits larger than the intended percentage cap.

This breaks the intended invariant:
`dailyLendIncrease ≤ MAX_DAILY_LEND_INCREASE * totalLentAssets`

[Code Reference](https://github.com/revert-finance/lend/blob/2bb022ee862c0f7f2010505f4d697f33827312ef/src/V3Vault.sol#L1200-L1234)

[Code Reference](https://github.com/revert-finance/lend/blob/2bb022ee862c0f7f2010505f4d697f33827312ef/docs/agents/risk-and-acceptability.md?plain=1#L74-L85)

## Impact
Because the circuit breaker is intended to cap daily liquidity growth relative to the lender base, failing to rescale `dailyLendIncreaseLimitLeft` after a lender haircut allows an attacker to inject liquidity far above the intended percentage cap precisely during distressed conditions(when lenders have just been socialized). This undermines the protocol's risk controls and allows sudden liquidity dominance or manipulation of pool utilization immediately after a loss event.

**Likelihood:** High because it requires a liquidation that triggers lender loss, after which exploitation is trivial via a normal deposit.

**Severity:** High because it breaks a core economic safety invariant and weakens protocol protections during stressed conditions.

## Mitigation
Recalculate and clamp `dailyLendIncreaseLimitLeft` after a lender haircut the same way the contract already clamps the debt limit.

`V3Vault::_handleReserveLiquidation`
```solidity
uint256 preHaircutDailyLendCap = _calculateDailyIncreaseLimit(
    oldLendExchangeRateX96, dailyLendIncreaseLimitMin, MAX_DAILY_LEND_INCREASE_X32
);

uint256 postHaircutDailyLendCap = _calculateDailyIncreaseLimit(
    newLendExchangeRateX96, dailyLendIncreaseLimitMin, MAX_DAILY_LEND_INCREASE_X32
);

if (
    dailyLendIncreaseLimitLastReset == uint32(block.timestamp / 1 days)
        && postHaircutDailyLendCap < preHaircutDailyLendCap
) {
    uint256 capReduction = preHaircutDailyLendCap - postHaircutDailyLendCap;
    dailyLendIncreaseLimitLeft =
        capReduction >= dailyLendIncreaseLimitLeft ? 0 : dailyLendIncreaseLimitLeft - capReduction;
}
```

## POC

Add this test in lend/test/integration/aerodrome/V3VaultAerodrome.t.sol
Runnable Command: forge test --mt testDailyLendLimitNotAdjustedAfterHaircut -vvvvv

```solidity
    function testDailyLendLimitNotAdjustedAfterHaircut() public {
        // Disabled oracle TWAP difference protection so liquidation path can execute in test environment
        oracle.setMaxPoolPriceDifference(type(uint16).max);

        uint256 tokenId = createPositionProper(alice, address(usdc), address(dai), 1, -100, 100, 1e18, 1000e6, 1000e18);

        vm.startPrank(alice);
        npm.approve(address(vault), tokenId);
        vault.create(tokenId, alice);

        //Borrow close to maximum allowed value
        (,, uint256 collateralValue,,) = vault.loanInfo(tokenId);
        uint256 borrowAmount = collateralValue * vault.BORROW_SAFETY_BUFFER_X32() / Q32;

        vault.borrow(tokenId, borrowAmount);
        vm.stopPrank();

        // Remove liquidity and accrued fees so collateral value collapses
        npm.setLiquidity(tokenId, 0);
        npm.setTokensOwed(tokenId, 0, 0);

        // Liquidate the position to trigger lender haircut
        IVault.LiquidateParams memory params = IVault.LiquidateParams({
            tokenId: tokenId, recipient: address(this), amount0Min: 0, amount1Min: 0, deadline: block.timestamp
        });

        vault.liquidate(params);

        (, uint256 lentAfterHaircut,,,,) = vault.vaultInfo();

        //Deposit equal to the entire post haircut pool size
        // This exceed the intended daily growth cap (~10%)
        uint256 depositAmount = lentAfterHaircut;

        usdc.mint(address(this), depositAmount);
        usdc.approve(address(vault), depositAmount);
        vault.deposit(depositAmount, address(this));

        //Verify circuit breaker invariant violation
        (, uint256 lentAfterDeposit,,,,) = vault.vaultInfo();

        // Pool grows by >10% in a single transaction
        // proving dailyLendIncreaseLimitLeft was not rescaled after haircut
        assertGt(lentAfterDeposit, lentAfterHaircut * 11 / 10);
    }
```

# Title
Inconsistent enforcement of autoCompoundRewardMin allows user Configuration bypass in executeWithVaultAndRewardCompound

## Summary
In `AutoRangeAndCompound::executeWithVaultAndRewardCompound()` forwards the operator supplied `rewardParams.minAeroReward` directly to the vault without enforcing the user configured `autoCompoundRewardMin`. As a result, the user defined minimum reward threshold can be bypassed during execution. 

This behavior is inconsistent with `autoCompoundWithVaultAndRewardCompound()`, which correctly overrides the operator input with the stored user configuration. The mismatch leads to inconsistent enforcement of user safety parameters across execution paths.

## Root Cause
In `autoCompoundWithVaultAndRewardCompound`, the contract explicitly overrides the operator supplied `rewardParams.minAeroReward` with the user configured value from `PositionConfig`. However, the same enforcement is missing in `executeWithVaultAndRewardCompound`.

```solidity
function executeWithVaultAndRewardCompound(
    ExecuteParams calldata params,
    address vault,
    IVault.RewardCompoundParams calldata rewardParams
) external {
    if (!operators[msg.sender] || !vaults[vault]) {
        revert Unauthorized();
    }

    IVault(vault).transformWithRewardCompound(
        params.tokenId,
        address(this),
        abi.encodeCall(AutoRangeAndCompound.execute, (params)),
        rewardParams
    );
}
```
Here, `rewardParams.minAeroReward` is passed directly from the operator input and is not validated or overridden using the stored user configuration.

## Impact

**Likelihood:** High, as the execution path using `executeWithVaultAndRewardCompound` can be triggered during normal automation flows.

**Severity:** Medium, due to bypass of user defined safety configuration and inconsistent enforcement across execution paths.

## Mitigation
Ensure that the user configured `autoCompoundRewardMin` is enforced in `executeWithVaultAndRewardCompound` the same way it is enforced in `autoCompoundWithVaultAndRewardCompound`.

```solidity
function executeWithVaultAndRewardCompound(
    ExecuteParams calldata params,
    address vault,
    IVault.RewardCompoundParams calldata rewardParams
) external {
    if (!operators[msg.sender] || !vaults[vault]) {
        revert Unauthorized();
    }

    // enforce user configuration
    PositionConfig memory config = positionConfigs[params.tokenId];
    IVault.RewardCompoundParams memory adjustedRewardParams = rewardParams;
    adjustedRewardParams.minAeroReward = config.autoCompoundRewardMin;

    IVault(vault).transformWithRewardCompound(
        params.tokenId,
        address(this),
        abi.encodeCall(AutoRangeAndCompound.execute, (params)),
        adjustedRewardParams
    );
}
```

## POC

Paste this test in lend/test/unit/AutoRangeAndCompoundRewardCompoundConfig.t.sol

Runnale Command: forge test --mt testExecutePathBypassesAutoCompoundRewardMin -vvvvv

```solidity
    function testExecutePathBypassesAutoCompoundRewardMin() external {
        vm.prank(USER);
        autoRange.configToken(
            TOKEN_ID,
            address(0),
            AutoRangeAndCompound.PositionConfig({
                lowerTickLimit: 0,
                upperTickLimit: 0,
                lowerTickDelta: 0,
                upperTickDelta: 0,
                token0SlippageX64: 0,
                token1SlippageX64: 0,
                onlyFees: false,
                autoCompound: true,
                maxRewardX64: 0,
                autoCompoundMin0: 0,
                autoCompoundMin1: 0,
                autoCompoundRewardMin: 100
            })
        );

        IVault.RewardCompoundParams memory rewardParams =
            IVault.RewardCompoundParams({minAeroReward: 0, aeroSplitBps: 4000, deadline: block.timestamp + 1 hours});

        vm.prank(OPERATOR);
        autoRange.executeWithVaultAndRewardCompound(
            AutoRangeAndCompound.ExecuteParams({
                tokenId: TOKEN_ID,
                swap0To1: false,
                amountIn: 0,
                swapData: "",
                amountRemoveMin0: 0,
                amountRemoveMin1: 0,
                amountAddMin0: 0,
                amountAddMin1: 0,
                deadline: block.timestamp,
                rewardX64: 0
            }),
            address(mockVault),
            rewardParams
        );

        // Vulnerable it is 0
        assertEq(mockVault.lastMinAeroReward(), 0);
        // Operator supplied minAeroReward is forwarded without enforcing user configured autoCompoundRewardMin
    }
```

# Title
Stale AutoExit configuration persists after NFT transfer allowing unauthorized execution by operator

## Summary
AutoExit stores automation configurations keyed only by `tokenId`. When an NFT position is transferred to a new owner, the previous owner's configuration remains active. If the new owner has approved the AutoExit contract, an operator can execute the stale configuration and perform liquidity removal or swaps using parameters defined by the previous owner, without the new owner's consent.

## Root Cause
If the NFT is transferred, the configuration remains active for the same tokenId, allowing operators to execute it even though the new owner never set it.
```solidity
mapping(uint256 => PositionConfig) public positionConfigs;


function configToken(uint256 tokenId, PositionConfig calldata config) external {
    ..............
    address owner = nonfungiblePositionManager.ownerOf(tokenId);
    if (owner != msg.sender) {
        revert Unauthorized();
    }

    positionConfigs[tokenId] = config; // configuration tied only to tokenId
    ..............
}
```
The configuration is bound only to the tokenId and not to the owner that created it. Although configToken() verifies ownership at configuration time, the contract does not revalidate the configurator during execution and does not clear configurations when the NFT is transferred. As a result, if a configured NFT is transferred to a new owner, the previous owner's configuration remains active and can still be executed by the operator.

[Code Reference](https://github.com/revert-finance/lend/blob/2bb022ee862c0f7f2010505f4d697f33827312ef/src/automators/AutoExit.sol#L221-L248)

[Code Reference](https://github.com/revert-finance/lend/blob/2bb022ee862c0f7f2010505f4d697f33827312ef/src/automators/AutoExit.sol#L44-L57)

## Impact
A new owner can unknowingly inherit an active automation configuration created by a previous owner.This results in: 
- Forced/Unintended liquidity removal
- Execution of swaps using parameters defined by another user
- Protocol reward extraction(maxRewardX64) from the new owner's funds

**Likelihood:** Medium because occurs whenever a configured NFT is transferred
**Severity:** Medium because funds are not stolen, but the new owner’s position can be forcibly executed without their consent.

## Mitigation

Store the owner at configuration time and verify it during execution. If ownership has changed, invalidate the configuration.

```solidity
struct PositionConfig {
    address owner; // address that created the config
    bool isActive;
    bool token0Swap;
    bool token1Swap;
    int24 token0TriggerTick;
    int24 token1TriggerTick;
    uint64 token0SlippageX64;
    uint64 token1SlippageX64;
    bool onlyFees;
    uint64 maxRewardX64;
}
```

Store the configurator during configuration:
```solidity
function configToken(uint256 tokenId, PositionConfig calldata config) external {
    ............
    address owner = nonfungiblePositionManager.ownerOf(tokenId);
    if (owner != msg.sender) revert Unauthorized();

    PositionConfig memory newConfig = config;
    newConfig.owner = msg.sender;

    positionConfigs[tokenId] = newConfig;
    .........
}
```

Validate ownership during execution
```solidity
function execute(ExecuteParams calldata params) external {
    .............
    PositionConfig memory config = positionConfigs[params.tokenId];

    address currentOwner = nonfungiblePositionManager.ownerOf(params.tokenId);

    // if ownership changed since configuration → invalidate
    if (currentOwner != config.owner) {
        delete positionConfigs[params.tokenId];
        revert NotConfigured();
    }
    ............
}
```

## POC
Paste this test in lend/test/integration/uniswap/automators/AutoExit.t.sol

Runnable Command: 
- export ANKR_API_KEY=YOUR_API_KEY
- forge test --mt testConfigPersistsAfterTransfer -vvvvv

```solidity
    function testConfigPersistsAfterTransfer() external {
        address alice = TEST_NFT_3_ACCOUNT;
        address bob = address(0xB0B);

        // fork already created in setUp()

        //Alice configures AutoExit
        vm.prank(alice);
        autoExit.configToken(
            TEST_NFT_3,
            AutoExit.PositionConfig(
                true,
                false,
                false,
                -300000, // token0TriggerTick
                -276326, // token1TriggerTick (current tick >= this --> ready)
                0,
                0,
                false,
                MAX_REWARD
            )
        );

        // Alice transfers NFT to Bob
        vm.prank(alice);
        NPM.safeTransferFrom(alice, bob, TEST_NFT_3);

        // check
        assertEq(NPM.ownerOf(TEST_NFT_3), bob);

        vm.prank(bob);
        NPM.setApprovalForAll(address(autoExit), true);

        // capture liquidity before
        (,,,,,,, uint128 liquidityBefore,,,,) = NPM.positions(TEST_NFT_3);
        assertTrue(liquidityBefore > 0);

        // Operator executes automation using stale config
        vm.prank(OPERATOR_ACCOUNT);
        autoExit.execute(AutoExit.ExecuteParams(TEST_NFT_3, "", 0, 0, block.timestamp, MAX_REWARD));

        //Verify Bob's position was closed
        (,,,,,,, uint128 liquidityAfter,,,,) = NPM.positions(TEST_NFT_3);

        assertEq(liquidityAfter, 0);
    }
```