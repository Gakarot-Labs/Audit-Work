# Title
Unused deposit tokens are permanently locked in LPPlugin due to ignored callback return values

## Summary
`LPPlugin.deposit()` transfers user tokens to the plugin and calls `LPCallback.mint()` to add liquidity, but it ignores the returned values indicating how many tokens were actually used.
If the pool requires fewer tokens than provided(which is common due to price and tick range constraints), the unused portion remains in the `LPPlugin` contract.
Unlike the NFT migration path `onERC721Received`, the deposit path does not refund these excess tokens.
As a result, user funds can become permanently locked in the plugin contract.

## Root Cause
In `deposit()`, the contract calls `mint()` but ignores its return values:

```solidity
token0.transferFrom(msg.sender, address(this), amount0);
token1.transferFrom(msg.sender, address(this), amount1);

uint256 actual0 = token0.balanceOf(address(this)) - balance0Before;
uint256 actual1 = token1.balanceOf(address(this)) - balance1Before;

token0.approve(callback, actual0);
token1.approve(callback, actual1);

ILPCallback(callback).mint(recipient, tickLower, tickUpper, actual0, actual1);
```
Because the returned values are ignored, the contract does not know how many tokens were actually used by the pool and therefore cannot compute the unused portion.

As a result, if the pool requires fewer tokens than the user supplied(which commonly occurs due to price and tick range constraints), the unused tokens remain in the `LPPlugin` contract and are never refunded to the user.

**Evidence of Intended Behavior:**

The NFT migration path `onERC721Received` correctly captures the return values and refunds any unused tokens:
```solidity
(uint256 amount0Used, uint256 amount1Used,) =
    ILPCallback(callback).mint(from, tickLower, tickUpper, amount0, amount1);

uint256 refund0 = amount0 - amount0Used;
uint256 refund1 = amount1 - amount1Used;
```
This demonstrates that unused tokens are expected to be refunded, but the `deposit()` path fails to implement the same logic, leading to permanent token locking inside the plugin contract.

[Code Reference](https://github.com/OrganizationE-STORM/Liquid-Positions-Plugin/blob/8c50ecf3657753e4de33d95851f9b2566103b4f0/contracts/LPPlugin.sol#L506-L554)

[Code Reference](https://github.com/OrganizationE-STORM/Liquid-Positions-Plugin/blob/8c50ecf3657753e4de33d95851f9b2566103b4f0/contracts/LPPlugin.sol#L318-L342)

## Impact
Users permanently lose a portion of their deposited tokens when adding liquidity through `deposit()`. If the pool consumes fewer tokens than provided, the unused tokens remain trapped in the `LPPlugin` contract with no mechanism for recovery. 

Over time, this can lead to cumulative token loss for multiple users, effectively locking user funds inside the contract.

## POC

Paste this test in `Liquid-Positions-Plugin/test/invariants/LPPlugin.spec.ts` in `describe('#deposit', async () => {` block

```typescript
        it('should leave unused tokens stuck in plugin after deposit', async function () {
            const { plugin, pluginAddr, callback, token0, token1, signers } = await setup(1);

            await token0.connect(signers[1]).approve(pluginAddr, ethers.MaxUint256);
            await token1.connect(signers[1]).approve(pluginAddr, ethers.MaxUint256);

            const deposit0 = ethers.parseEther('1');
            const deposit1 = ethers.parseEther('1');

            const pluginToken0Before = await token0.balanceOf(pluginAddr);

            const tx = await plugin.connect(signers[1]).deposit(
                signers[1].address,
                -60,
                60,
                deposit0,
                deposit1,
                0,
                Number.MAX_SAFE_INTEGER
            );

            const receipt = await tx.wait();

            const { amount0, amount1 } = readNewAmountsFromMintEvent(receipt, callback);

            const pluginToken0After = await token0.balanceOf(pluginAddr);
            const pluginToken1After = await token1.balanceOf(pluginAddr);

            const unused0 = deposit0 - amount0;
            const unused1 = deposit1 - amount1;

            if (unused0 > 0n) {
                expect(pluginToken0After - pluginToken0Before).to.equal(unused0);
            }

            if (unused1 > 0n) {
                expect(pluginToken1After).to.equal(unused1);
            }
        });
```

# Title