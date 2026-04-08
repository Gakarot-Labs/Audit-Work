# Title(Med)
Attacker can freeze ShareTokenUpgradeable::unregisterVault forever by sending minimal dust

## Summary

The function `ShareTokenUpgradeable::unregisterVault` only allows a vault to be removed from the registry if its ERC20 asset balance is exactly zero. However, a vault cannot prevent external addresses from directly transferring tokens to it, and the system provides no sweep, no dust-cleanup, and no tolerance threshold for small balances.

This creates a permanent denial of service condition: an attacker or even an accidental user can send as little as 1 wei of the asset token directly to the vault. Once the vault holds any non-zero balance, unregisterVault will always revert with `CannotUnregisterVaultAssetBalance()`, and the owner has no way to restore the balance back to zero.

Links to affected code
[ShareTokenUpgradeable.sol#L282-L327](https://github.com/code-423n4/2025-11-sukukfi/blob/d7e734192a571511f47962e7f228ad5ed275e7db/src/ShareTokenUpgradeable.sol#L282-L327)

## Recommended mitigation steps

Add Sweep Function:

```solidity
function sweepDust(address asset, address to) external onlyOwner {
    uint256 bal = IERC20(asset).balanceOf(address(this));
    if (bal == 0) revert NoDustToSweep();
    IERC20(asset).transfer(to, bal);

    emit DustSwept(asset, bal);
}
```

In unregisterVault replace this:
```solidity
if (IERC20(asset).balanceOf(vaultAddress) != 0) {
            revert CannotUnregisterVaultAssetBalance();
        }
```        
with this:

```solidity
uint256 bal = IERC20(asset).balanceOf(vaultAddress);
if (bal > 0) {
        sweepDust(asset, owner());
    }
```

Atomic Unregister that prevents Mempool Attacks

## POC

Add this test in File Location: test/AuditReproduction.t.sol

```solidity
    function test_UnregisterVault_CanBePermanentlyDoSedBy1WeiTransfer() public {
        address attacker = address(0xBEEF);

        // Sanity check vault is empty
        assertEq(asset.balanceOf(address(mainVault)), 0);

        vm.prank(owner);
        mainVault.setVaultActive(false); // Deactivate for the DoS

        asset.mint(attacker, 1);
        vm.prank(attacker);
        // Attacker griefs the vault with 1 wei dust
        asset.transfer(address(mainVault), 1);

        assertEq(asset.balanceOf(address(mainVault)), 1);

        // Now unregister must revert due to dust
        vm.prank(owner);
        vm.expectRevert();
        mainShareToken.unregisterVault(address(asset));
    }

```

---

# Title(Low)
Cross Role rBalance flag mismatch causes DoS in computeRBalanceFlags

## Summary

The `_computeRBalanceFlagsInternal()` function contains a flawed invariant that forces global flag consistency across all roles debtor or creditor.
When an account appears multiple times in the same batch first as a creditor with its flag set to true, and later as a debtor with its flag set to false the function incorrectly reverts with `InconsistentRAccounts`.
In a real settlement system, a user can easily be a creditor in one transfer and debtor in another, with different rBalance semantics.

The function treats the rBalance update flag as a per-account global invariant, not a per-transfer, per-role signal.

```solidity
function _computeRBalanceFlagsInternal(
....
         bool currentTransferMarksDebtor = debtorsRBalanceFlags[i];
                        bool debtorAlreadyMarked = ((rBalanceFlags >> j) & 1) == 1;

         if (currentTransferMarksDebtor != debtorAlreadyMarked) {
                            revert InconsistentRAccounts(debtor, debtorAlreadyMarked, currentTransferMarksDebtor);
...
                        }

```

This implicitly assumes:
If an account was ever flagged true once, it must always be true.

This is wrong, because:
- rBalance update logic depends on role debtor vs creditor
- Roles can differ between transfers
- Flags can legitimately differ between roles
- But the contract forces them to match
- This creates a broken invariant and causes spurious reverts.

## Links to affected code
[WERC7575ShareToken.sol#L877-L918](https://github.com/code-423n4/2025-11-sukukfi/blob/163216500a54627afc6abaac3bffdc3a830051fa/src/WERC7575ShareToken.sol#L877-L918)

## POC
The Exact Failure Scenario

Transfer 0: Alice appears as a CREDITOR

```solidity
creditors[0] = Alice
creditorsRBalanceFlags[0] = true
```
Result inside computeRBalanceFlags:
```solidity
First discovery:
AliceFlag = true   // stored in rBalanceFlags bitmap
```

Transfer 1: Alice appears as a DEBTOR:
```solidity
debtors[1] = Alice
debtorsRBalanceFlags[1] = false
```

Now the function performs its “consistency check”:
```solidity
previousFlag = true     // from first discovery as creditor
currentFlag  = false    // from this transfer as debtor
```

Mismatch happens and it reverts
InconsistentRAccounts(Alice, true, false)

```solidity
 function test_DoS_CrossRoleFlagMismatch() public {
        // Deploy WERC share + vault
        WERC7575ShareToken wShare = new WERC7575ShareToken("W", "W");
        WERC7575Vault vault = new WERC7575Vault(address(asset), wShare);

        // Register vault + KYC verify
        wShare.registerVault(address(asset), address(vault));
        wShare.setKycVerified(address(this), true);

        // Prepare a batch where:
        // - Alice first appears as creditor with flag = true
        // - Later appears as debtor with flag = false
        address alice = address(0xBEEF);

        address[] memory debtors = new address[](2);
        address[] memory creditors = new address[](2);
        uint256[] memory amounts = new uint256[](2);

        // Transfer 0 Alice creditor
        debtors[0] = address(0xBEEF);
        creditors[0] = alice;
        amounts[0] = 10;

        // Transfer 1 Alice debtor
        debtors[1] = alice;
        creditors[1] = address(0xBEEF);
        amounts[1] = 20;

        // rBalance flags arrays per-transfer flags
        bool[] memory debtorFlags = new bool[](2);
        bool[] memory creditorFlags = new bool[](2);

        // First time Alice appears as creditor with flag = true
        debtorFlags[0] = false;
        creditorFlags[0] = true;

        // Second time Alice appears as debtor with flag = false
        debtorFlags[1] = false;
        creditorFlags[1] = false;

        // Expect revert due to inconsistent flag logic
        vm.expectRevert();
        wShare.computeRBalanceFlags(debtors, creditors, debtorFlags, creditorFlags);
    }
```

# Title(Low)
Pause bypass in batchTransfers and rBatchTransfers functions allows forced Transfers while contract is Paused

## Summary

The contract’s pause system only blocks user facing flows but does not block the internal settlement mechanisms: `batchTransfers` and `rBatchTransfers`. Despite the contract comments explicitly stating three times that pausing should halt `batchTransfers` and `rBatchTransfers` adjustments, these functions are missing the `whenNotPaused` modifier. As a result, even when the contract is paused, the validator can continue to move user balances, bypassing the intended emergency stop.

```solidity
    /**
     * @dev Pause critical ShareToken operations. Only callable by owner.
>@   * Used for emergency situations to halt batch transfers and rBalance adjustments.
     */
    function pause() external onlyOwner {
        _pause();
    }

     * Requirements:
     * - All arrays must have the same length
     * - Maximum 100 transfers per batch
>@   * - Contract must not be paused
     * - Sufficient balance in debtor accounts
     */
    function batchTransfers(address[] calldata debtors, address[] calldata creditors, uint256[] calldata amounts) external onlyValidator returns (bool) {}

     * Requirements:
     * - All arrays must have the same length
     * - Maximum 100 transfers per batch
>@   * - Contract must not be paused
     * - Sufficient balance in debtor accounts
     * - rBalanceFlags must be pre-computed using computeRBalanceFlags()
     */
    function rBatchTransfers(address[] calldata debtors, address[] calldata creditors, uint256[] calldata amounts, uint256 rBalanceFlags) external onlyValidator returns (bool) {}
```

## Mitigation
Enforce the pause state on all settlement paths exactly as the documentation promises.
Add whenNotPaused to both settlement entry points:

```solidity
 function batchTransfers(address[] calldata debtors, address[] calldata creditors, uint256[] calldata amounts) external onlyValidator whenNotPaused  returns (bool) {}

    function rBatchTransfers(address[] calldata debtors, address[] calldata creditors, uint256[] calldata amounts, uint256 rBalanceFlags) external onlyValidator whenNotPaused returns (bool) {}
```

## Links to affected code
[WERC7575ShareToken.sol#L700-L734](https://github.com/code-423n4/2025-11-sukukfi/blob/163216500a54627afc6abaac3bffdc3a830051fa/src/WERC7575ShareToken.sol#L700-L734)
[WERC7575ShareToken.sol#L1119-L1202](https://github.com/code-423n4/2025-11-sukukfi/blob/163216500a54627afc6abaac3bffdc3a830051fa/src/WERC7575ShareToken.sol#L1119-L1202)

## POC

Paste this test in File Location: test/AuditReproduction.t.sol

```solidity
 function test_PauseDontBlockBatchTransfers() public {
        // Setup WERC + Vault
        WERC7575ShareToken w = new WERC7575ShareToken("Inv Share", "INV");
        WERC7575Vault vault = new WERC7575Vault(address(asset), w);
        w.registerVault(address(asset), address(vault));
        // KYC for addresses
        address alice = address(0x10);
        address bob = address(0x11);
        w.setKycVerified(alice, true);
        w.setKycVerified(bob, true);

        // Mint 100 tokens to alice via vault
        vm.prank(address(vault));
        w.mint(alice, 100e18);
        assertEq(w.balanceOf(alice), 100e18);

        // Pause contract
        w.pause();
        assertTrue(w.paused(), "should be paused");

        // Mint should revert while paused coz mint has whenNotPaused
        vm.prank(address(vault));
        vm.expectRevert();
        w.mint(bob, 1e18);

        // Prepare batch transfer
        address[] memory debtors = new address[](1);
        address[] memory creditors = new address[](1);
        uint256[] memory amounts = new uint256[](1);
        debtors[0] = alice;
        creditors[0] = bob;
        amounts[0] = 50e18;

        // Expect batchTransfers to succeed despite pause coz pause not applied to batchTransfers
        bool ok = w.batchTransfers(debtors, creditors, amounts);
        assertTrue(ok, "batchTransfers should succeed even when paused");

        // Verify balances updated
        assertEq(w.balanceOf(alice), 50e18, "alice should have 50 after batch");
        assertEq(w.balanceOf(bob), 50e18, "bob should have 50 after batch");
    }
```
