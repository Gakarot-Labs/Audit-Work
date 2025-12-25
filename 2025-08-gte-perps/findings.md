# Finding-1-Low
# Title---->Misaligned rewards in Distributor:addRewards due to positional parameters

---

## Root
`Distributor:addRewards(address token0, address token1, uint128 amount0, uint128 amount1)` accepts token pairs in any order, and uses positional amounts that correspond directly to the token order passed. Although the function contains logic to detect which token is the pool’s launchAsset and swaps internal variables accordingly, **there is no protection or explicit API that prevents callers from supplying amounts that are the wrong way round relative to their intent.** The function’s forgiving “accept tokens in any order” behavior plus positional amount0/amount1 leads to easy human / integration error: a sponsor can (accidentally or maliciously via a bad UI) pass amounts in the wrong order and thus fund the wrong token amounts.

The issue is not merely frontend/UX related the contract API itself is misleading. **It claims to support order-agnostic inputs but still binds amounts positionally,** creating a protocol-level ambiguity. This makes the likelihood non-negligible, and combined with the high impact (fund loss).

---

## Impact
- **Direct financial loss for reward sponsors:** Sponsor ends up transferring more of one token and less of the other than intended.  
- **Unexpected gain/loss for stakers:** Stakers may receive more of an asset than the sponsor intended (overpayment) and less of the other (underpayment). If the overpaid asset has a much lower market value `(e.g., receiving 200 tokens worth 1 dollar instead of 100 tokens worth 10dollar), the staker also suffers an economic loss. ` Once rewards are credited to the wrong pool, stakers can immediately claim them. These assets are unrecoverable, leading to permanent loss for the sponsor.
- **Distorted incentives:** Misaligned rewards break the intended incentive structure, potentially discouraging staking if participants consistently receive rewards in a less valuable token.  
- **Severity:** This is immediate, non-recoverable value transfer when `addRewards` succeeds. It breaks trust in rewards accounting and can be exploited.Because the protocol itself promises order-agnostic behavior and fails to uphold it, this is not a frontend issue but a **protocol guarantee failure. It results in permanent, unrecoverable loss of sponsor funds. This is a High severity issue because it represents a protocol-level guarantee failure that leads to irreversible misallocation of assets and direct financial loss.**


---

## Detailed description

**Bug / Problem:**
- **The contract API is positional:** `(token0, token1, amount0, amount1)`. The amounts are applied to whichever token is passed in that same position. This is error prone.
- The `Distributor:addRewards` function attempts to be order-agnostic by checking whether token0 corresponds to a created reward pool and, if not, switching to token1.**But it does not validate that caller intent (which token they meant as base vs quote) matches the amounts supplied. If the caller supplies the amounts reversed relative to their intent, the pool state will reflect reversed funding.**

**Expected behaviour:**
Sponsoring X of base token and Y of quote token should always result in exactly X being credited to the base pool and Y to the quote pool regardless of how the caller arranges the tokens/parameters (or — better — the API should require a single canonical order and be unambiguous).
The contract should either (A) force a canonical explicit ordering (e.g., addRewards(launchAsset, quoteAsset, baseAmount, quoteAmount)), or (B) accept token→amount mappings (struct/array) so that amounts cannot be mis-associated by position.


File location: 2025-08-gte-perps/contracts/launchpad/Distributor.sol
```solidity
  function addRewards(address token0, address token1, uint128 amount0, uint128 amount1) external {
        (address launchAsset, address quoteAsset, uint128 launchAssetAmount, uint128 quoteAssetAmount) =
            (token0, token1, amount0, amount1);
        RewardPoolData storage rs = RewardsTrackerStorage.getRewardPool(token0);

        if (rs.quoteAsset == address(0)) {
            rs = RewardsTrackerStorage.getRewardPool(token1);

            if (rs.quoteAsset == address(0)) revert RewardsDoNotExist();

            (launchAsset, quoteAsset, launchAssetAmount, quoteAssetAmount) = (token1, token0, amount1, amount0);
        }

        if (rs.totalShares == 0) revert NoSharesToIncentivize();

        if (launchAssetAmount > 0) {
            rs.addBaseRewards(launchAsset, launchAssetAmount);
            _increaseTotalPending(launchAsset, launchAssetAmount);
            launchAsset.safeTransferFrom(msg.sender, address(this), uint256(launchAssetAmount));
        }

        if (quoteAssetAmount > 0) {
            rs.addQuoteRewards(launchAsset, quoteAsset, quoteAssetAmount);
            _increaseTotalPending(quoteAsset, quoteAssetAmount);
            quoteAsset.safeTransferFrom(msg.sender, address(this), uint256(quoteAssetAmount));
        }
    }

```
---

## Proof of Concept
Using the provided test, the issue can be reproduced directly:

- A rewards pair is created for `(baseToken, quoteToken)` and `userA` stakes on the base side.  
- Sponsor intends to fund `(100 base, 200 quote)` but mistakenly calls `addRewards(baseToken, quoteToken, 200, 100)`.  
- The pool records `(200 base, 100 quote)`, so `userA` accrues 200 base + 100 quote instead of the intended 100 base + 200 quote.  

This mismatch causes direct misallocation of funds.This misalignment cannot be corrected once rewards are added, making the loss immediate and irrecoverable. 
The PoC works with the included test file.  

Paste this test in File location: 2025-08-gte-perps/test/c4-poc/PoCLaunchpad.t.sol

```solidity

import {ILaunchpad} from "contracts/launchpad/interfaces/ILaunchpad.sol";
import {IOperatorPanel} from "contracts/utils/interfaces/IOperatorPanel.sol";

import {Distributor} from "contracts/launchpad/Distributor.sol";
import {ERC20Harness} from "../harnesses/ERC20Harness.sol";
import {LaunchToken} from "contracts/launchpad/LaunchToken.sol";


    function test_submissionValidity() public {
        address userA = makeAddr("userA");
        vm.label(userA, "userA");

        // -------------------------------------------------
        // Step 1: Buy out bonding curve to trigger graduation
        // -------------------------------------------------
        // Curve has a fixed bonding supply (total base to be sold).
        // We leave 1e18 unpurchased, buy the rest, and then finish the last purchase
        // to trigger "graduation" (unlock transfers + set up pool).
        uint256 leave = 1e18;
        uint256 almostAll = curve.bondingSupply(token) - leave;
        uint256 quoteForAlmostAll = curve.quoteQuoteForBase(token, almostAll, true);

        // Fund user with enough quote to buy out the curve
        quoteToken.mint(userA, quoteForAlmostAll + 1e21);

        vm.startPrank(userA);
        quoteToken.approve(address(launchpad), type(uint256).max);

        // Buy almost all tokens from the curve
        launchpad.buy(
            ILaunchpad.BuyData({
                account: userA,
                token: token,
                recipient: userA,
                amountOutBase: almostAll,
                maxAmountInQuote: quoteForAlmostAll
            })
        );

        // Buy the last bit → this triggers graduation & unlocks transfers
        uint256 quoteForLast = curve.quoteQuoteForBase(token, leave, true);
        launchpad.buy(
            ILaunchpad.BuyData({
                account: userA,
                token: token,
                recipient: userA,
                amountOutBase: leave,
                maxAmountInQuote: quoteForLast
            })
        );
        vm.stopPrank();

        // -------------------------------------------------
        // Step 2: UserA stakes base tokens
        // -------------------------------------------------
        uint96 stakeAmount = 100 ether;
        vm.startPrank(userA);
        ERC20Harness(token).approve(distributor, type(uint256).max);
        vm.stopPrank();

        // Launchpad triggers Distributor::increaseStake on behalf of user
        vm.prank(address(launchpad));
        Distributor(distributor).increaseStake(token, userA, stakeAmount);

        // -------------------------------------------------
        // Step 3: Sponsor funds distributor with rewards
        // -------------------------------------------------
        address sponsor = makeAddr("sponsor");
        vm.label(sponsor, "sponsor");

        // Sponsor is given quote tokens directly
        quoteToken.mint(sponsor, 500 ether);

        // UserA transfers some base tokens to sponsor
        // (so sponsor can also fund base rewards)
        vm.prank(userA);
        LaunchToken(token).transfer(sponsor, 500 ether);

        // Sponsor approves distributor for both tokens
        vm.startPrank(sponsor);
        ERC20Harness(token).approve(distributor, type(uint256).max);
        quoteToken.approve(distributor, type(uint256).max);

        // -------------------------------------------------
        // BUG TRIGGER: Misaligned parameter order
        // -------------------------------------------------
        // Intended reward funding:
        //   base = 100 ether, quote = 200 ether
        //
        // Actual call (parameters swapped):
        //   base = 200 ether, quote = 100 ether
        //
        // This misalignment causes users to receive the wrong reward proportions.
        Distributor(distributor).addRewards(token, address(quoteToken), 200 ether, 100 ether);
        vm.stopPrank();

        // -------------------------------------------------
        // Step 4: Verify misaligned outcome
        // -------------------------------------------------
        // Expected (intended):   pendingBase = 100 ether, pendingQuote = 200 ether
        // Actual (buggy call):   pendingBase = 200 ether, pendingQuote = 100 ether
        //
        // The test confirms that misalignment occurred → user is overpaid in base
        // and underpaid in quote.
        (uint256 pendingBase, uint256 pendingQuote) = Distributor(distributor).getPendingRewards(token, userA);
        assertApproxEqAbs(
            pendingBase,
            200 ether,
            1e14, // tolerance: 0.0001 ether = 100,000,000,000,000 wei
            "UserA got wrong base rewards (overpaid)"
        );
        assertApproxEqAbs(
            pendingQuote,
            100 ether,
            1e14, // tolerance: 0.0001 ether = 100,000,000,000,000 wei
            "UserA got wrong quote rewards (underpaid)"
        );
    }
```


# Finding-2-Low
# TITLE---->DoS (liveness) during Launchpad's `Graduation: _swapRemaining` pulls msg.sender (operator) instead of the buyer account.

## Root:

During graduation, `Launchpad:_swapRemaining` calls `data.quote.safeTransferFrom(msg.sender, ...)`, which pulls tokens from the operator (msg.sender) when an operator relays the buy, not from the buyer account that actually funded the purchase. This causes a revert when the operator has no allowance and results in a liveness/DoS failure.

`Launchpad:_swapRemaining` uses msg.sender as the source of funds. When an operator/relayer executes `buy(...)` on behalf of `buyData.account`, `msg.sender is the operator`, who typically has no ERC20 allowance to Launchpad so `transferFrom` reverts. The intended payer is the buyer (buyData.account), but that address is never used as the transferFrom source.


## Impact:

This blocks the final graduation step (locking bonding supply and creating the Uniswap LP). As a result, protocol liveness is broken: the token cannot complete its launch, even though the buyer intended to finish it. No funds are directly stolen, but launches can be temporarily stalled.

Concrete observed failure in test: transaction reverts with `ERC20: transferFrom failed / Insufficient Allowance`. User and operator balances remain unchanged, but graduation cannot complete.

`Severity-Medium`: Temporary denial of service for operator-triggered graduation. No direct fund loss, but it can delay launches and reduce reliability of operator relaying.

`Additional Problem:` Incorrect refund to msg.sender.
In the current `Launchpad_swapRemaining` implementation, the catch block also uses `msg.sender` when refunding quote tokens if the swap fails. `data.quote.safeTransfer(msg.sender, data.quoteAmount);`


## Description:

### Bug / Problem:

- In `Launchpad:_swapRemaining`, the contract executes:`data.quote.safeTransferFrom(msg.sender, address(this), data.quoteAmount);`
- When an operator relays the call, msg.sender is the operator, not the buyer.
- The operator usually has no ERC20 allowance set, while the buyer is the one who approved the Launchpad.
- This mismatch causes the transfer to fail, creating a denial-of-service vector for graduation.

File location: 2025-08-gte-perps/contracts/launchpad/Launchpad.sol
```solidity
    function _swapRemaining(SwapRemainingData memory data) internal returns (uint256, uint256) {
        // Transfer the remaining quote from the user
>@      data.quote.safeTransferFrom(msg.sender, address(this), data.quoteAmount);

        // Prepare swap path
        address[] memory path = new address[](2);
        path[0] = data.quote;
        path[1] = data.token;

        // Approve router to spend remaining quote
        data.quote.safeApprove(address(uniV2Router), data.quoteAmount);

        try uniV2Router.swapTokensForExactTokens(
            data.baseAmount, data.quoteAmount, path, data.recipient, block.timestamp + 1
        ) {
            // Return the tokens received and quote used
            return (data.baseAmount, data.quoteAmount);
        } catch {
            // If swap fails, return the additional quote tokens to the user and remove approval
            data.quote.safeApprove(address(uniV2Router), 0);
            data.quote.safeTransfer(msg.sender, data.quoteAmount);
            return (0, 0);
        }
    }
```

### Mitigation:

Ensure the graduation swap flow explicitly charges and refunds the original buyer `(the buyData.account)` rather than relying on `msg.sender`. The swap routine must always transferFrom the buyer for the AMM quoteNeeded and, on failure, refund that same buyer. This removes the incorrect reliance on the relayer/operator and prevents operator-relayed buys from causing allowance reverts or misdirected refunds.

This ensures the transfer uses the user’s allowance and cannot be blocked by an operator’s missing approval.
Also ensure the refund path in the catch uses the buyer’s address (same payer), or the contract may incorrectly refund the operator.”


## Proof of Concept:

- User buys almost all bonding supply, leaving only 1e18 base unpurchased.
- Operator (malOp) is authorized via IOperatorPanel to act for the user.
- Operator requests more base than remains, triggering graduation `(_graduate → _createPairAndSwapRemaining → _swapRemaining)`.
- `Launchpad:_swapRemaining` calls `safeTransferFrom(msg.sender, ...)`, where `msg.sender = malOp`.
- Because malOp gave no allowance, the transfer reverts with ERC20InsufficientAllowance.

Result: Graduation fails, protocol liveness is stuck, and the launch cannot complete.

Paste this test in File location: 2025-08-gte-perps/test/c4-poc/PoCLaunchpad.t.sol

```solidity
import {ILaunchpad} from "contracts/launchpad/interfaces/ILaunchpad.sol";
import {IOperatorPanel} from "contracts/utils/interfaces/IOperatorPanel.sol";

    function test_submissionValidity() public {
        /**
         * ---------------------------------------------------------------
         * 1) Setup: User buys almost the entire bonding supply
         * ---------------------------------------------------------------
         * Leave exactly 1e18 base remaining so that the *next* buy
         * will trigger graduation (since supply will be exhausted).
         */
        uint256 leave = 1e18;
        uint256 buyAlmostAllBase = curve.bondingSupply(token) - leave;
        uint256 quoteForBuyAlmostAll = curve.quoteQuoteForBase(token, buyAlmostAllBase, true);

        // Mint quote tokens to user and perform the large buy
        quoteToken.mint(user, quoteForBuyAlmostAll);
        vm.startPrank(user);
        launchpad.buy(
            ILaunchpad.BuyData({
                account: user,
                token: token,
                recipient: user,
                amountOutBase: buyAlmostAllBase,
                maxAmountInQuote: quoteForBuyAlmostAll
            })
        );
        vm.stopPrank();

        // Sanity check: only `leave` tokens remain unbought
        assertEq(curve.baseSoldFromCurve(token) + leave, curve.bondingSupply(token));

        /**
         * ---------------------------------------------------------------
         * 2) Prepare malicious operator
         * ---------------------------------------------------------------
         * - Operator (`malOp`) is allowed by operator panel to act on behalf of user.
         * - Operator has plenty of quote tokens, but gives NO allowance to Launchpad.
         * - Operator will request *more base than remains* so `_swapRemaining` is triggered.
         */
        address malOp = makeAddr("malOp");
        address operatorAddr = makeAddr("operator");

        // Mock operator approval so malOp can act for user
        vm.mockCall(
            address(operatorAddr),
            abi.encodeWithSelector(IOperatorPanel.getOperatorRoleApprovals.selector, user, malOp),
            abi.encode(uint256(1))
        );

        // Fund operator and user with quote tokens (large balance, but no allowance set)
        uint256 operatorQuoteBalance = 1e24;
        quoteToken.mint(malOp, operatorQuoteBalance);
        quoteToken.mint(user, operatorQuoteBalance);

        // Operator will over-request base: ask for 2e18 while only 1e18 remains
        uint256 requestedByOperator = leave + 1e18;
        uint256 operatorMaxQuote = operatorQuoteBalance; // set very large, so quote is not the limiting factor

        // --- Sanity checks: confirm preconditions ---
        assertEq(quoteToken.allowance(malOp, address(launchpad)), 0, "operator must NOT have approved launchpad");
        assertGt(quoteToken.balanceOf(user), 0, "user must have quote tokens");
        assertGt(quoteToken.balanceOf(malOp), 0, "malOp must have quote tokens");

        /**
         * ---------------------------------------------------------------
         * 3) Graduation buy triggered by operator
         * ---------------------------------------------------------------
         * - Prank as `malOp` so msg.sender = operator.
         * - Graduation logic calls `_swapRemaining`, which tries:
         *     data.quote.safeTransferFrom(msg.sender, address(this), data.quoteAmount)
         * - Since msg.sender is malOp and malOp gave no allowance,
         *   transferFrom fails and the tx reverts.
         */
        vm.startPrank(malOp);
        vm.expectRevert();
        launchpad.buy(
            ILaunchpad.BuyData({
                account: user, // operator buys on behalf of user
                token: token,
                recipient: user,
                amountOutBase: requestedByOperator,
                maxAmountInQuote: operatorMaxQuote
            })
        );
        vm.stopPrank();

        /**
         * ---------------------------------------------------------------
         * 4) Post-condition: No tokens pulled from operator
         * ---------------------------------------------------------------
         */
        assertEq(quoteToken.balanceOf(malOp), operatorQuoteBalance, "malOp funds must remain untouched");
    }

```

# Finding-3-Low
# Title---->Uneconomic/Impractical Gas Consumption When Matching Many Orders at Same Price

## Description
The protocol’s order matching engine iterates through each individual maker order at a given price level.
If a taker order tries to consume multiple maker orders at once, the gas usage grows linearly with the number of maker orders at that price.
This creates scenarios where valid user trades become impractical because the gas cost can exceed the economic value of the trade.

`Bug/Problem:`

- Multiple small maker orders placed at the same price level are not aggregated.
- A taker consuming them must loop over all orders sequentially.
- Gas usage becomes very high even under normal usage (not just attacker spam).

File Location: 2025-08-gte-perps/contracts/perps/types/Book.sol

```solidity

    // Unbounded iteration over numOrders at the same price
    // If thousands of orders exist, taker order must loop through all of them
    // High gas cost, potential DoS by exceeding block gas limit

    function _getQuoteLimit(Book storage self, Limit storage limit, uint256 price, uint256 baseAmount)
        private
        view
        returns (uint256 quoteAmount, uint256 baseUsed)
    {
        uint256 numOrders = limit.numOrders;
        OrderId orderId = limit.headOrder;

        uint256 fillAmount;
        for (uint256 i; i < numOrders; ++i) {
            if (baseAmount == 0) break;
            if (orderId.unwrap() == 0) break;

            fillAmount = self.orders[orderId].amount.min(baseAmount);

            quoteAmount += fillAmount.fullMulDiv(price, 1e18);
            baseAmount -= fillAmount;
            baseUsed += fillAmount;

            orderId = self.orders[orderId].nextOrderId;
        }
    }

    function _getBaseLimit(Book storage self, Limit storage limit, uint256 price, uint256 quoteAmount)
        private
        view
        returns (uint256 baseAmount, uint256 quoteUsed)
    {
        uint256 numOrders = limit.numOrders;
        OrderId orderId = limit.headOrder;

        uint256 fillAmount;
        for (uint256 i; i < numOrders; ++i) {
            if (quoteAmount == 0) break;
            if (orderId.unwrap() == 0) break;

            fillAmount = self.orders[orderId].amount.min(quoteAmount.fullMulDiv(1e18, price));

            baseAmount += fillAmount;
            quoteUsed += fillAmount.fullMulDiv(price, 1e18);
            quoteAmount -= fillAmount.fullMulDiv(price, 1e18);

            orderId = self.orders[orderId].nextOrderId;
        }
    }
```

`Impact:`

- Normal usage DoS: Large trades (with many small orders at the same price) become infeasible, discouraging users and reducing protocol usability.
- Attacker DoS: An attacker can deliberately place thousands of tiny orders at one price to clog the book, making it prohibitively expensive for takers to fill.

`Severity:`
- Normal usage case → Medium (protocol becomes self-DOS’d, no one trades big sizes).
- Attacker case → Low (attacker spends capital to spam book, mitigatable via fees or limits).

## Mitigation
- Aggregate maker orders at the same price level into a single entry (merge orders by price).
- Impose a cap on the number of active orders per price level.
- Use batching / skip-list / heap-based structures to reduce iteration cost.

## POC
- Place N maker orders at the same price (e.g., 40 makers, 1 lot each).
- Place a large taker order consuming all makers.
- Observe that gas usage grows linearly with N (≈6M gas for 40 orders).

Paste this test in File Location: 2025-08-gte-perps/test/c4-poc/PoCPerps.t.sol

```solidity

import "../../contracts/perps/types/Structs.sol";

    function test_submissionValidity() public {
        // -------------------------------
        // Setup: basic params
        // -------------------------------
        bytes32 asset = ETH;
        uint256 price = _conformTickEth(4000e18); // price aligned to tick size
        Side side = Side.SELL; // makers will SELL
        uint256 lotSize = perpManager.getLotSize(asset);

        uint256 numMakers = 40; // number of maker orders at same price
        uint256 orderAmount = lotSize; // each maker sells 1 lot

        // -------------------------------
        // Step 1: Place multiple maker orders at same price
        // -------------------------------
        for (uint256 i = 0; i < numMakers; i++) {
            // deterministic unique maker address (same pattern as elsewhere)
            address maker = address(uint160(uint256(keccak256(abi.encodePacked("maker", i)))));

            // fund + deposit
            _mintAndApproveAndDeposit(maker, 1_000_000e18);

            // maker order args
            PlaceOrderArgs memory makerArgs = PlaceOrderArgs({
                subaccount: 0,
                asset: asset,
                side: side,
                limitPrice: price,
                amount: orderAmount,
                baseDenominated: true,
                tif: TiF.MOC, // maker-only (limit order)
                expiryTime: 0,
                clientOrderId: 0,
                reduceOnly: false
            });

            // place order as maker
            vm.startPrank(maker);
            perpManager.placeOrder(maker, makerArgs);
            vm.stopPrank();
        }

        // -------------------------------
        // Step 2: Place a large taker order to consume all makers
        // -------------------------------
        address taker = address(uint160(uint256(keccak256(abi.encodePacked("taker")))));
        _mintAndApproveAndDeposit(taker, 1_000_000e18);

        PlaceOrderArgs memory takerArgs = PlaceOrderArgs({
            subaccount: 0,
            asset: asset,
            side: Side.BUY,
            limitPrice: price,
            amount: numMakers * orderAmount, // consume all maker orders
            baseDenominated: true,
            tif: TiF.IOC, // immediate-or-cancel taker order
            expiryTime: 0,
            clientOrderId: 0,
            reduceOnly: false
        });

        // -------------------------------
        // Step 3: Measure gas used by taker order
        // -------------------------------
        vm.startPrank(taker);
        uint256 startGas = gasleft();
        perpManager.placeOrder(taker, takerArgs);
        uint256 used = startGas - gasleft();
        vm.stopPrank();

        // Log gas used (forge test logger)
        emit log_named_uint("Gas used by taker order consuming all makers", used);
    }
    /*
    Conclusion:
    - Maker setup: 40 orders × 0.001 ETH each = 0.04 ETH total liquidity at price 4000 USDC/ETH = $160.
    - Gas used: 6,049,814
    - GasPrice = 1.5 gwei → ETH cost = 0.009074721 ETH → $36.30 → ≈ 22.7% of $160 trade.
    - GasPrice = 2.0 gwei → ETH cost = 0.012099628 ETH → $48.40 → ≈ 30.2% of $160.
    - GasPrice = 30.0 gwei → ETH cost = 0.181494420 ETH → $725.98 → ≈ 453.7% of $160.

    Even with ETH = $4,000, numbers show gas is a huge chunk (22–30% at low gas prices; catastrophic at high gas).
    This demonstrates that the protocol can self-DOS under normal usage.
    */

```

# Finding-4-Low
# Title---->Inconsistent denominators in Market:getImpactPrice() cause asymmetric bid/ask weighting and skewed mark price.

## Root
A unit mistake: the bid-side fallback uses type(uint256).max as the denominator while the ask-side fallback uses 1. This makes the bid contribution effectively zero and the ask contribution extremely large for the same inputs, producing a badly biased impactPrice.

## Description

`Bug / Problem:`
Inside getImpactPrice() the code adds a fallback amount when impactNotional > quoteUsed. 
Because type(uint256).max is astronomically large, (x * 1e18) / type(uint256).max ≈ 0 for any realistic x. The ask side uses / 1, producing (x * 1e18). The resultant impactBid ≈ 0 and impactAsk ≫ 0, so impactPrice = (impactBid + impactAsk)/2 is dominated by the ask side. The operations were clearly intended to be symmetric but are not.
The two symmetric branches use different denominators:

File Location: 2025-08-gte-perps/contracts/perps/types/Market.sol

```solidity
// buggy
baseAmount += (impactNotional - quoteUsed).fullMulDiv(1e18, type(uint256).max); // bid side
...
baseAmount += (impactNotional - quoteUsed).fullMulDiv(1e18, 1);                // ask side

```

`Impact:`

- setMarkPrice(...) consumes getImpactPrice() when updating market markPrice. Mark price is a core pricing primitive used for liquidations, P&L calculations, funding and snapshots.
- In thin-liquidity scenarios (where quoteUsed is small or zero) the bug is triggered and mark price becomes arbitrarily skewed toward the ask side. That can cause:
-- False/incorrect liquidations (users liquidated when they should not be),
-- Incorrect P&L (losses/gains miscomputed),
-- Distorted funding rates and snapshots,
-- Potential exploitable windows where adversaries manipulate on-chain mark price for profit.

Likelihood: Thin orderbook states occur frequently for new/low-liquidity markets or during flash events. Because this touches core accounting, the finding is High severity.

## Mitigation
- One-line fix (recommended): make the denominators symmetric. Replace the bid-side denominator type(uint256).max with 1 (or the intended consistent scale factor used by both branches). E.g.:

```solidity
- if (impactNotional > quoteUsed) baseAmount += (impactNotional - quoteUsed).fullMulDiv(1e18, type(uint256).max);
+ if (impactNotional > quoteUsed) baseAmount += (impactNotional - quoteUsed).fullMulDiv(1e18, 1);

```

`Notes / alternative fixes:`
- Confirm intended units: if both sides should scale to base units, 1 is likely correct; if they should scale relative to a price or another factor, use that same factor on both sides. Add explicit comments describing units and why 1e18 is used in the numerator.
- Consider replacing the ad-hoc fallback with a small helper function addFallbackBaseAmount(baseAmount, deltaNotional, scaleDenom) so symmetry is enforced by code structure (reduces copy-paste bugs).
- Add defensive guards: clamp or sanity-check impactPrice relative to indexPrice (e.g., enforce |markPrice - indexPrice| <= divergenceCap * indexPrice) to limit transient manipulation or catastrophic values.

## POC

PoC test: exercise the admin entrypoint `setMarkPrice(...)` to prove the inconsistent denominators bug in `getImpactPrice()`.

What this test does (high level):
1. Compute the local "buggy" impact price using the same wrong denominators the repo uses.
2. Compute the "fixed" impact price using the symmetric denominator (for comparison).
3. Call perpManager.setMarkPrice(...) as admin to execute the on-chain logic.
4. Record and decode the emitted `MarkPriceUpdated` event and extract `p3` (impactPrice).
5. Assert that the emitted p3 equals the locally computed buggy impact price and that buggy != fixed (i.e., the bug is material).
- Keep in mind: this test is intentionally robust it doesn't assert the entire event payload (markPrice/p1/nonce) because mocks may set them differently; instead it decodes and asserts only `p3`.

Paste this test in File Location: 2025-08-gte-perps/test/c4-poc/PoCPerps.t.sol
```solidity
import "forge-std/Vm.sol"; // gives us the `Vm` type (Vm.Log) and helper functions like recordLogs/getRecordedLogs
import {FixedPointMathLib} from "@solady/utils/FixedPointMathLib.sol";

    using FixedPointMathLib for uint256;

    /**
     * We re-declare the event signature locally so we can compute its keccak256 signature and
     * understand the layout: `MarkPriceUpdated(bytes32 indexed asset, uint256 markPrice, uint256 p1, uint256 p2, uint256 p3, uint256 nonce)`
     *
     * - asset is indexed (so appears in topics[1])
     * - the rest (markPrice, p1, p2, p3, nonce) are ABI-encoded into the event `data` field
     *
     * We declare it here only for clarity; we do not emit it ourselves.
     */
    event MarkPriceUpdated(bytes32 indexed asset, uint256 markPrice, uint256 p1, uint256 p2, uint256 p3, uint256 nonce);

    /**
     * Test: call the admin entrypoint so the contract runs the internal code-path that
     * computes the impact price and emits MarkPriceUpdated(..., p3, ...).
     *
     * Structure/comments inside the function explain every step.
     */
    function test_submissionValidity() external {
        // --- Setup: choose an existing market + an index price ---
        bytes32 asset = ETH;
        uint256 indexPrice = 4000e18; // 4,000 scaled by 1e18, matches repo scaling
        uint256 impactNotional = 1e18; // chosen to trigger (impactNotional > quoteUsed) fallback branch

        // === Local replication of the repo's buggy math ===
        // These compute what the repo currently does:
        // - bid branch uses denom = type(uint256).max (effectively zero contribution)
        // - ask branch uses denom = 1 (large contribution)
        uint256 baseBid_bug = 0;
        baseBid_bug += (impactNotional - 0).fullMulDiv(1e18, type(uint256).max);
        uint256 impactBid_bug = baseBid_bug == 0 ? 0 : impactNotional.fullMulDiv(1e18, baseBid_bug);

        uint256 baseAsk_bug = 0;
        baseAsk_bug += (impactNotional - 0).fullMulDiv(1e18, 1);
        uint256 impactAsk_bug = baseAsk_bug == 0 ? 0 : impactNotional.fullMulDiv(1e18, baseAsk_bug);

        uint256 impactPrice_bug = (impactBid_bug + impactAsk_bug) / 2;

        // === Local computation of the intended/fixed symmetric math ===
        // Make both sides use denominator = 1 so contributions mirror each other.
        uint256 baseBid_fix = 0;
        baseBid_fix += (impactNotional - 0).fullMulDiv(1e18, 1);
        uint256 impactBid_fix = baseBid_fix == 0 ? 0 : impactNotional.fullMulDiv(1e18, baseBid_fix);

        uint256 baseAsk_fix = baseAsk_bug; // same as above
        uint256 impactAsk_fix = impactAsk_bug;

        uint256 impactPrice_fix = (impactBid_fix + impactAsk_fix) / 2;

        // Quick sanity checks on the local math before invoking on-chain:
        //  - ask side should be > 0 under the buggy calculation
        //  - bid side should be effectively 0 under the buggy calculation
        assertTrue(impactAsk_bug > 0, "ask contribution should be > 0 in bug scenario");
        assertEq(impactBid_bug, 0, "buggy bid contribution expected to be zero-ish");

        // --- Call the on-chain admin entrypoint and record emitted logs ---
        // Using vm.recordLogs + vm.getRecordedLogs allows us to decode events freely
        // instead of using vm.expectEmit which requires exact matching of all fields.
        vm.recordLogs();

        // Call as admin (perpManager.setMarkPrice has an onlyAdmin guard)
        vm.prank(admin);
        perpManager.setMarkPrice(asset, indexPrice);

        // Retrieve the recorded logs
        Vm.Log[] memory entries = vm.getRecordedLogs();

        // Compute the event signature hash so we can find it in topics[0]
        bytes32 sig = keccak256("MarkPriceUpdated(bytes32,uint256,uint256,uint256,uint256,uint256)");

        // Search logs for our event and decode the non-indexed data:
        // event data layout: (markPrice, p1, p2, p3, nonce)
        bool found = false;
        uint256 emitted_p3 = 0;

        for (uint256 i = 0; i < entries.length; i++) {
            // topics[0] is the event signature; topics[1] is indexed asset
            if (entries[i].topics.length > 0 && entries[i].topics[0] == sig) {
                // decode the ABI-encoded non-indexed event parameters
                (uint256 markPrice, uint256 p1, uint256 p2, uint256 p3, uint256 nonce) =
                    abi.decode(entries[i].data, (uint256, uint256, uint256, uint256, uint256));

                // keep the extracted p3 for assertion
                emitted_p3 = p3;
                found = true;
                break;
            }
        }

        // Ensure the event was emitted (sanity)
        require(found, "MarkPriceUpdated event not found in logs");

        // --- Assertions proving the bug end-to-end ---
        // 1) Emitted on-chain p3 must equal the local buggy calculation -> proves contract executed buggy math
        assertEq(emitted_p3, impactPrice_bug, "emitted p3 should match buggy computed impactPrice");

        // 2) The buggy and fixed values must differ -> proves bug.
        assertFalse(impactPrice_bug == impactPrice_fix, "buggy and fixed impact prices should differ");
    }

```

# Finding-5-Low
# Title---->Unbounded Iteration Across Multiple Price Levels in CLOBLib(_matchIncomingBid and _matchIncomingAsk) Causes Gas DoS in Matching Engine

## Root
The functions _matchIncomingBid and _matchIncomingAsk in CLOBLib.sol iterate sequentially across multiple price levels without any global bound or aggregation. When a taker order crosses many ticks, gas usage scales linearly with the number of levels traversed.

`Note:`This is distinct from the same-price-level issue (Book.sol: _getQuoteLimit / _getBaseLimit); that finding demonstrates intra-tick iteration. This PoC demonstrates inter-tick iteration across many price levels (CLOBLib: _matchIncomingBid / _matchIncomingAsk).

## Description
`Bug/Problem:`
- The matching engine scans price levels one by one while filling a large taker order.
- Each tick with at least one maker order is visited individually; no batching or global guard exists.
- With fragmented liquidity across many ticks, taker fills become gas-inefficient and may exceed block gas limits.
- Normal usage with many small orders spread across different ticks can already trigger uneconomic execution.

File Location: File Location: 2025-08-gte-perps/contracts/perps/types/CLOBLib.sol

```solidity
// _matchIncomingBid
        while (bestAsk <= incomingOrder.price && incomingOrder.amount > 0) {
            if (bestAsk == type(uint256).max) break;
            if (bestAsk > maxAsk) break;

            Limit storage limit = ds.askLimits[bestAsk];
            Order storage bestAskOrder = ds.orders[limit.headOrder];

            if (bestAskOrder.isExpired()) {
                _removeUnfillableOrder(ds, bestAskOrder);
                bestAsk = ds.getBestAsk();
                continue;
            }

// _matchIncomingAsk
        while (bestBid >= incomingOrder.price && incomingOrder.amount > 0) {
            if (bestBid == 0) break;
            if (bestBid < minBid) break;

            Limit storage limit = ds.bidLimits[bestBid];
            Order storage bestBidOrder = ds.orders[limit.headOrder];

            if (bestBidOrder.isExpired()) {
                _removeUnfillableOrder(ds, bestBidOrder);
                bestBid = ds.getBestBid();
                continue;
            }

```

`Impact:`
- Normal case: A fragmented book (many small orders across consecutive ticks) makes taker trades extremely costly. Our test with 40 orders (0.04 ETH spread across 40 ticks) consumed ~6.28M gas, where execution cost was ~24–32% of the trade value at normal gas prices, and >400% at high gas prices.
- Attack case: An attacker can deliberately distribute small orders across many ticks to amplify gas griefing. Any large taker order that tries to cross the book can be forced into prohibitively expensive execution.
- Severity: Medium (self-DoS under normal use, griefing vector for attackers). Severity increases toward High if market configuration allows extremely small lot/tick sizes or very large maxNumOrders, or if maker fee structure does not deter fragmentation.

## Mitigation
Aggregate per-tick liquidity (robust fix)
Track total liquidity at each price level (Limit.totalBase, Limit.totalQuote) so fills can skip per-order iteration when possible.
Example modification to Limit struct:
``` solidity
struct Limit {
    uint256 headOrder;
    uint256 tailOrder;
    uint256 numOrders;
    uint256 totalBase;   // new: sum of base across all orders
    uint256 totalQuote;  // optional: cached quote liquidity
}

// When adding/removing orders:
limit.totalBase += newOrder.amount;
limit.totalQuote += newOrder.amount.fullMulDiv(price, 1e18);


// When matching:
if (incomingOrder.amount >= limit.totalBase) {
    // fill entire price level in O(1)
    incomingOrder.amount -= limit.totalBase;
    baseReceived += limit.totalBase;
    quoteSent   += limit.totalQuote;
    _clearLimit(limit); // remove all orders at this tick
} else {
    // fall back to per-order iteration for partial fill
    _matchOrdersIndividually(ds, limit, incomingOrder);
}

```
`Note:` Note: Batched settlement must still preserve per-maker effects (fees, reduceOnly, processMakerFill semantics).

## Poc
- The PoC deploys 40 maker orders of 0.001 ETH each at consecutive price levels (3900 → 3939 USDC/ETH).
- A single taker buy order large enough to consume all liquidity is then placed.
- The gas usage logged is ~6.28M, showing linear growth with the number of price levels traversed.
- At realistic gas prices, this cost exceeds or significantly eats into trade value, proving the DoS potential.
- Test config: tickSize=0.001, lotSize=0.001, minLimitOrderAmountInBase=0.001, maxNumOrders=1_000_000

Paste this test in File Location: 2025-08-gte-perps/test/c4-poc/PoCPerps.t.sol

```solidity
import "../../contracts/perps/types/Structs.sol";

    function test_submissionValidity() public {
        bytes32 asset = ETH;
        uint256 baseLot = perpManager.getLotSize(asset);
        uint256 startPrice = _conformTickEth(3900e18);

        uint256 numMakers = 40; // number of price levels to fill
        uint256 orderAmount = baseLot; // 1 lot per price level

        // -------------------------------
        // Step 1: Place multiple maker orders across price levels
        // -------------------------------
        for (uint256 i = 0; i < numMakers; i++) {
            address maker = address(uint160(uint256(keccak256(abi.encodePacked("makerMulti", i)))));
            _mintAndApproveAndDeposit(maker, 1_000_000e18);

            uint256 price = startPrice + (i * perpManager.getTickSize(asset)); // increasing prices each tick

            PlaceOrderArgs memory makerArgs = PlaceOrderArgs({
                subaccount: 0,
                asset: asset,
                side: Side.SELL, // makers are selling
                limitPrice: price,
                amount: orderAmount,
                baseDenominated: true,
                tif: TiF.MOC, // maker only
                expiryTime: 0,
                clientOrderId: 0,
                reduceOnly: false
            });

            vm.startPrank(maker);
            perpManager.placeOrder(maker, makerArgs);
            vm.stopPrank();
        }

        // -------------------------------
        // Step 2: Place a taker order to consume across price levels
        // -------------------------------
        address taker = address(uint160(uint256(keccak256("takerMulti"))));
        _mintAndApproveAndDeposit(taker, 1_000_000e18);

        uint256 takerAmount = numMakers * orderAmount; // big enough to cross all levels
        uint256 takerPrice = startPrice + (numMakers * perpManager.getTickSize(asset));

        PlaceOrderArgs memory takerArgs = PlaceOrderArgs({
            subaccount: 0,
            asset: asset,
            side: Side.BUY,
            limitPrice: takerPrice, // will cross through all lower sell prices
            amount: takerAmount,
            baseDenominated: true,
            tif: TiF.IOC, // taker order
            expiryTime: 0,
            clientOrderId: 0,
            reduceOnly: false
        });

        // -------------------------------
        // Step 3: Measure gas consumption
        // -------------------------------
        vm.prank(taker);
        uint256 startGas = gasleft();
        perpManager.placeOrder(taker, takerArgs);
        uint256 used = startGas - gasleft();

        emit log_named_uint("Gas used by taker order across multiple price levels", used);

        /**
         * Conclusion (Cross-Price-Level Iteration):
         *
         * Maker setup: 40 orders × 0.001 ETH each, spread across 40 price ticks (3900 → 3939 USDC/ETH).
         *
         * Total base liquidity = 0.04 ETH, total notional ≈ 156,000 USDC ($156).
         *
         * Gas used by taker order crossing all ticks: 6,287,822.
         *
         * GasPrice = 1.5 gwei → ETH cost = 0.009431733 ETH → $37.73 → ≈ 24.2% of $156 trade.
         *
         * GasPrice = 2.0 gwei → ETH cost = 0.012575644 ETH → $50.31 → ≈ 32.3% of $156 trade.
         *
         * GasPrice = 30.0 gwei → ETH cost = 0.188634660 ETH → $754.65 → ≈ 483.9% of $156 trade.
         */
    }

```


# Finding-6-Low
# Title---->Fragile swap deadline: block.timestamp + 1 causes flaky swaps.

## Description

- During graduation the contract calls the router function. In Launchpad:_swapRemaining

```solidity
uniV2Router.swapTokensForExactTokens(data.baseAmount, data.quoteAmount, path, data.recipient, block.timestamp + 1)
``` 

- That gives the router only a one-second margin between the deadline and the current block timestamp. Deadlines are checked at execution time (when the transaction is mined), so a 1-second window is brittle: mempool delays, sequencer/miner timestamp variance or propagation delays commonly exceed 1s, which will cause legitimate swaps to revert. 
- In practice, even small timestamp skews (2–5s) already make the deadline invalid, causing swaps to revert and gas to be wasted.

`Impact:` flaky user transactions, wasted gas, and confusing debugging.

Relevant snippet:

File Location: 2025-08-gte-perps/contracts/launchpad/Launchpad.sol
File link: https://github.com/code-423n4/2025-08-gte-perps/blob/9fb17ba2de649106c6310aa351226c9bfab7f40a/contracts/launchpad/Launchpad.sol#L536-L559

```solidity
        try uniV2Router.swapTokensForExactTokens(
            data.baseAmount, data.quoteAmount, path, data.recipient, block.timestamp + 1
        ) {
            // Return the tokens received and quote used
            return (data.baseAmount, data.quoteAmount);
        } catch {
            // If swap fails, return the additional quote tokens to the user and remove approval
            data.quote.safeApprove(address(uniV2Router), 0);
            data.quote.safeTransfer(msg.sender, data.quoteAmount);
            return (0, 0);
        }
    }
```

## Mitigation/Suggestion:

- `First Option:` Increase the default deadline to a sane window, e.g. uint256 deadline = block.timestamp + 300; (5 minutes) or at least +60. Low-risk and fixes flakiness.

- `Second Option:` Allow callers to pass a deadline param through the flow (caller-controlled override), and fall back to a reasonable default.

One-line patch example:

```solidity
uint256 deadline = block.timestamp + 60; // safe default
uniV2Router.swapTokensForExactTokens(..., deadline);
```


# Finding-7-Quality/Low
# Title---->Unbounded loop in LiquidatorPanel::_settleBackstopLiquidation() can cause out-of-gas during backstop liquidation

## Root
LiquidatorPanel::_settleBackstopLiquidation() reads the transient backstop-liquidator array and credits each liquidator in a single on-chain loop (no cap or batching). If the transient array length is large, the loop’s cumulative gas can exceed block limits.

## Description
`Bug / Problem:`
During backstop liquidations, the contract iterates over all transient backstop-liquidator entries and calls CollateralManager.creditAccount(...) for each entry within the same transaction. There is no upper bound, batching, or safeguarded continuation mechanism. With sufficiently many registered liquidators / high-volume entries, the liquidation transaction may run out of gas and revert, preventing settlement from completing.

File Location: 2025-08-gte-perps/contracts/perps/modules/LiquidatorPanel.sol
File Link: https://github.com/code-423n4/2025-08-gte-perps/blob/9fb17ba2de649106c6310aa351226c9bfab7f40a/contracts/perps/modules/LiquidatorPanel.sol#L665-L699

```solidity
        uint256 totalPoints;
        uint256 totalVolume;
        for (uint256 i; i < data.length; ++i) {
            points[i] = clearingHouse.liquidatorPoints[data[i].liquidator];
            totalPoints += points[i];
            totalVolume += data[i].volume;
        }

        uint256 pointShare;
        uint256 volumeShare;
        uint256 rate;
        uint256 fee;
        for (uint256 i; i < data.length; ++i) {
            pointShare = points[i].fullMulDiv(1e18, totalPoints);
            volumeShare = data[i].volume.fullMulDiv(1e18, totalVolume);

            rate = (pointShare + volumeShare) / 2;
            fee = margin.fullMulDiv(rate, 1e18);

            StorageLib.loadCollateralManager().creditAccount(data[i].liquidator, fee);
        }
```

`Impact:`
- Quality/Low: Affected operations are liquidation finalization and distribution of backstop fees. Functional availability risk; not immediate remote RCE/steal of funds, but could disrupt liquidation flow and require manual remediation.
- If triggered, legitimate backstop liquidations may fail (revert) due to out-of-gas, leaving liquidated positions unsettled and funds/fees undisbursed. That can stall liquidation resolution and put stress on downstream systems (insurance fund, collateral manager).
- Attack surface: an attacker with the ability to register many backstop entries (or an accidental accumulation) could make some backstop liquidations impossible in a single tx causing availability issues and requiring manual/operational intervention.

## Mitigation/Recommended fixes
- `Cap per-tx processing-` impose a configurable max number of liquidators processed per settlement tx (e.g., maxBackstopPayoutsPerTx) and process remaining entries in follow-up txs.
- `Push → Pull pattern-` instead of looping and crediting all liquidators in one tx, record each liquidator’s entitlement on-chain, then let each liquidator *pull* their share in a separate claim tx. This removes the OOG risk entirely.
- `Batching / pagination-` change _settleBackstopLiquidation to process at most k entries per call and return a cursor/index so callers can continue processing remaining entries in subsequent transactions.
- `Gas-aware limits-` optionally check gasleft() and stop processing when remaining gas is below a safe threshold, returning a continuation flag.
- `On-chain queue with idempotency-` persist per-entry processed flags or shift the transient array into a queue structure that supports partial consumption safely. Ensure operations are idempotent so retries don’t double-credit.
- `Atomic accounting / checkpointing-` record partial progress before crediting to avoid double/spurious payments in case of revert. Prefer a pattern: compute distribution amounts off-chain or in a read-only pass, then apply updates in capped batches.
- `Operational monitoring & limits-` add monitoring/alerts when transient list length or total volume grows unusually; add on-chain sanity checks to reject absurd registration volumes.


#
