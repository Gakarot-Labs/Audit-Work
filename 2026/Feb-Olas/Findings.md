# Title

Oracle Can Enter Irrecoverable Liveness Failure After Single Price Deviation

## Summary

`BalancerPriceOracle` implements a slippage-bounded update mechanism intended to protect the oracle from manipulated price updates.
The root cause is that `updatePrice()` returns early without updating any state when the current price deviates more than maxSlippage from the stored average price. Because `lastUpdated` is not advanced in this case, all future calls observe the same stale `averagePrice` and repeatedly fail the same slippage check, even if significant time has passed. This creates a state-machine deadlock where the oracle can never converge to the true market price under monotonic price movement.

```solidity
if (
    currentPrice < averagePrice - (averagePrice * maxSlippage / 100) ||
    currentPrice > averagePrice + (averagePrice * maxSlippage / 100)
) {
    return false; // state is not updated, causing permanent stall
}
```
[BalancerPriceOracle::updatePrice](https://github.com/valory-xyz/autonolas-tokenomics/blob/bbec5ac12721a62672fb7a5ffba5c40f5a46d8cb/contracts/oracles/BalancerPriceOracle.sol#L115-L120)

## Impact

The oracle can enter an unrecoverable state under valid market conditions, permanently blocking dependent protocol flows and requiring redeployment or governance intervention to recover.

## POC

This demonstrates that the oracle can enter a deadlocked state where progress is impossible under monotonic price movement.

```solidity
pragma solidity ^0.8.30;

import "forge-std/Test.sol";
import {BalancerPriceOracle} from "../contracts/oracles/BalancerPriceOracle.sol";

interface IVault {
    function getPoolTokens(bytes32 poolId)
        external
        view
        returns (address[] memory tokens, uint256[] memory balances, uint256 lastChangeBlock);
}

contract MockVault is IVault {
    address[] public tokens;
    uint256[] public balances;
    uint256 public lastChangeBlock;

    constructor(address token0, address token1, uint256 bal0, uint256 bal1) {
        tokens = new address[](2);
        tokens[0] = token0;
        tokens[1] = token1;

        balances = new uint256[](2);
        balances[0] = bal0;
        balances[1] = bal1;
    }

    function setBalances(uint256 bal0, uint256 bal1) external {
        balances[0] = bal0;
        balances[1] = bal1;
        lastChangeBlock = block.number;
    }

    function getPoolTokens(bytes32) external view returns (address[] memory, uint256[] memory, uint256) {
        return (tokens, balances, lastChangeBlock);
    }
}

contract OracleFreezeMockTest is Test {
    BalancerPriceOracle oracle;
    MockVault vault;

    address OLAS = address(0x1);
    address WETH = address(0x2);
    bytes32 POOL = bytes32("POOL");

    function setUp() public {
        vault = new MockVault(OLAS, WETH, 100 ether, 100 ether); // price = 1

        oracle = new BalancerPriceOracle(
            OLAS,
            WETH,
            10, // 10% slippage
            900, // min update
            address(vault),
            POOL
        );
    }

    function testPermanentFreeze() public {
        uint256 p0 = oracle.getPrice();
        assertEq(p0, 1e18);

        // store initial timestamp
        (, uint256 last0,) = oracle.snapshotHistory();

        // big jump
        vault.setBalances(100 ether, 150 ether);

        vm.warp(block.timestamp + 901);
        bool ok = oracle.updatePrice();
        assertFalse(ok);

        // warp again
        vm.warp(block.timestamp + 2000);
        ok = oracle.updatePrice();
        assertFalse(ok);

        // check still stuck
        (, uint256 last1,) = oracle.snapshotHistory();
        assertEq(last1, last0);
        // oracle cannot recover even after multiple update windows

    }
}

```
# Title
Spot price is overwritten by TWAP, allowing oracle deviation check bypass

## Summary

In `LiquidityManagerCore::checkPoolAndGetCenterPrice`, the spot price is accidentally overwritten with the TWAP price before the deviation is calculated. **As a result, the contract ends up comparing the TWAP price with itself instead of comparing spot vs TWAP, causing the deviation check to always evaluate to zero**.

This makes the oracle protection ineffective: even if the pool price is manipulated, liquidity operations continue as if the price were safe.

Also this check is intended to protect protocol owned liquidity from external price manipulation, but becomes ineffective due to the overwrite

## Root Cause

In `checkPoolAndGetCenterPrice`, the variable that initially stores the spot sqrt price is reused to store the TWAP sqrt price, overwriting the original value:

```solidity
(centerSqrtPriceX96, observationIndex) = _getPriceAndObservationIndexFromSlot0(pool);
```
Later, when the TWAP is fetched, the same variable is reused:

```solidity
(twapPrice, centerSqrtPriceX96) = abi.decode(returnData, (uint256, uint160));
```
Because `centerSqrtPriceX96` now contains the TWAP value, the `instant` price is computed from TWAP instead of spot:

```solidity
uint256 instantPrice = mulDiv(uint256(centerSqrtPriceX96), uint256(centerSqrtPriceX96), (1 << 64));
```
This causes the deviation calculation to compare TWAP against TWAP, always resulting in zero deviation and bypassing the intended protection.

## Impact

Oracle based slippage protection is rendered ineffective, allowing liquidity minting, burning and rebalancing to proceed at manipulated prices, potentially leading to loss of protocol owned liquidity or incorrect OLAS burns.

## POC

Paste this test in: autonolas-tokenomics/test/LiquidityManagerETH.t.sol

```solidity
    function testPriceDeviationBypass() public {
        //initial V3 position
        int24[] memory tickShifts = new int24[](2);
        tickShifts[0] = -25000;
        tickShifts[1] = 15000;

        liquidityManager.convertToV3(
            TOKENS,
            PAIR_V2_BYTES32,
            FEE_TIER,
            tickShifts,
            0,
            true
        );

        //manipulate price heavily
        address pool = IFactory(FACTORY_V3).getPool(TOKENS[0], TOKENS[1], uint24(FEE_TIER));

        // record spot price before
        (uint160 sqrtBefore,,,,,,) = IUniswapV3(pool).slot0();

        //pool price is nuked by large swap
        uint256 olasAmount = 200_000 ether;
        deal(OLAS, address(this), olasAmount);
        IToken(OLAS).approve(ROUTER_V3, olasAmount);

        IRouterV3.ExactInputSingleParams memory params = IRouterV3.ExactInputSingleParams({
            tokenIn: OLAS,
            tokenOut: WETH,
            fee: uint24(FEE_TIER),
            recipient: address(this),
            amountIn: olasAmount,
            amountOutMinimum: 1,
            sqrtPriceLimitX96: 0
        });

        IRouterV3(ROUTER_V3).exactInputSingle(params);

        // record spot price after
        (uint160 sqrtAfter,,,,,,) = IUniswapV3(pool).slot0();

        //price must move significantly
        uint256 beforeX96 = FullMath.mulDiv(sqrtBefore, sqrtBefore, FixedPoint96.Q96);
        uint256 afterX96 = FullMath.mulDiv(sqrtAfter, sqrtAfter, FixedPoint96.Q96);
        uint256 impact = beforeX96 > afterX96
            ? ((beforeX96 - afterX96) * 1e18) / beforeX96
            : ((afterX96 - beforeX96) * 1e18) / beforeX96;

        require(sqrtBefore != sqrtAfter, "price unchanged");


        //fund LM and try to use manipulated price
        deal(OLAS, address(liquidityManager), 10_000 ether);
        deal(WETH, address(liquidityManager), 10 ether);

        // Expected: revert due to MAX_ALLOWED_DEVIATION
        // Actual: call succeeds
        liquidityManager.increaseLiquidity(TOKENS, FEE_TIER, 0);
    }
```

## Mitigation

Use separate variables for spot and TWAP prices and ensure the deviation is calculated between them, avoiding overwriting the spot price with the TWAP value.

``` solidity
(uint160 spotSqrtPriceX96, uint16 observationIndex) = _getPriceAndObservationIndexFromSlot0(pool);

(uint256 twapPrice, uint160 twapSqrtPriceX96) = abi.decode(returnData, (uint256, uint160));
```
Then compute the instant price from spotSqrtPriceX96 and compare it against twapPrice.

# Title

Incorrect TWAP calculation due to double counting of elapsed time in BalancerPriceOracle

## Summary

The TWAP update logic double counts elapsed time by applying both the previous average price and the current spot price to the same interval.
This result as mathematically invalid TWAP and breaks oracle correctness even under honest market conditions.


## Root Cause

The TWAP calculation in `BalancerPriceOracle.updatePrice()` is mathematically incorrect because the elapsed time interval is double counted when updating the average price.

Specifically, the contract first correctly adds the previous average price over the elapsed time to the cumulative price:

```solidity
snapshot.cumulativePrice += snapshot.averagePrice * elapsedTime;
```

However, when computing the new average price, the same `elapsedTime` is added again using the current spot price:

```solidity
uint256 averagePrice = (snapshot.cumulativePrice + (currentPrice * elapsedTime))
    / ((snapshot.cumulativePrice / snapshot.averagePrice) + elapsedTime);
```

At this point, `snapshot.cumulativePrice` already includes the contribution of `elapsedTime` at the previous price, but the formula incorrectly assumes an additional interval of `elapsedTime` at `currentPrice`.
As a result, the same time window is counted twice: once at the old average price and once at the new spot price.

## Impact

Incorrect TWAP calculation enables price skew in oracle dependent operations, potentially leading to mispriced liquidity moves and buybacks.

## POC

Paste this test in: test/LiquidityManagerBase.t.sol

```solidity
    function testOracleTWAPDoubleCountsElapsedTime() public {
        // Make slippage so update never fails
        BalancerPriceOracle oracle = oracleV2;

        //initial price
        uint256 p0 = oracle.getPrice();

        // Advance time beyond minUpdateTimePeriod
        vm.warp(block.timestamp + minUpdateTimePeriod + 1);
        oracle.updatePrice();

        //Perform a real Balancer swap to move price
        deal(WETH, address(this), 1 ether);
        IToken(WETH).approve(BALANCER_VAULT, 1 ether);

        IBalancer.SingleSwap memory s = IBalancer.SingleSwap({
            poolId: POOL_V2_BYTES32,
            kind: IBalancer.SwapKind.GIVEN_IN,
            assetIn: WETH,
            assetOut: OLAS,
            amount: 1 ether,
            userData: ""
        });

        IBalancer.FundManagement memory f = IBalancer.FundManagement({
            sender: address(this),
            fromInternalBalance: false,
            recipient: payable(address(this)),
            toInternalBalance: false
        });

        IBalancer(BALANCER_VAULT).swap(s, f, 0, block.timestamp);

        uint256 p1 = oracle.getPrice();

        // Advance time again
        uint256 dt = minUpdateTimePeriod + 1;
        vm.warp(block.timestamp + dt);
        oracle.updatePrice();

        // Read oracle average
        (,, uint256 oracleAvg) = oracle.snapshotHistory();

        //Correct TWAP calculation
        // interval1: p0 for dt
        // interval2: p1 for dt
        uint256 correct = (p0 * dt + p1 * dt) / (2 * dt);

        // Oracle must be wrong due to double counting elapsedTime
        assertTrue(oracleAvg != correct);
    }
```

# Title
UniswapPriceOracle uses self referential TWAP, causing slippage check to always pass

## Summary
`UniswapPriceOracle::validatePrice()` calculates TWAP using the current price, so TWAP and spot price are always the same.
Because of this, the slippage check always returns true and never blocks bad prices.
The oracle gives a false sense of safety and does not protect against manipulation.

## Root Cause
The oracle tries to calculate a TWAP, but it uses the current spot price to build it.
Because of this, the TWAP always becomes equal to the current price, so the slippage check is useless.

In validatePrice(), the cumulative price is built like this:
```solidity
uint256 tradePrice = getPrice();
uint256 cumulativePrice = cumulativePriceLast + (tradePrice * elapsedTime);
uint256 timeWeightedAverage =
    (cumulativePrice - cumulativePriceLast) / elapsedTime;
```

This simplifies to:
`timeWeightedAverage = tradePrice`

Since TWAP and spot price are always the same, the difference is always zero:
```solidity
return derivation <= slippage;
```
So validatePrice() always returns true and never blocks bad prices.

## Impact
Any price is accepted as valid because the slippage check always passes.
This allows manipulated or incorrect prices to be used by downstream contracts without protection.

## POC

Paste this test in: test/LiquidityManagerETH.t.sol

```solidity
/// validatePrice() always returns true because TWAP == spot price
    function testOracleValidatePriceAlwaysTrue() public {
        //oracle should return some price
        uint256 spotPrice = oracleV2.getPrice();
        assertGt(spotPrice, 0, "spot price must be non-zero");

        // use minimum slippage
        uint256 slippage = 1;

        // call validation
        bool valid = oracleV2.validatePrice(slippage);

        // this must always be true, regardless of price movement
        assertTrue(valid, "validatePrice() always returns true due to broken TWAP");
    }
```
