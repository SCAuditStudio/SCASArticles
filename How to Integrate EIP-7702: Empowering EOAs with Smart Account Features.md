# How to Integrate EIP-7702: Empowering EOAs with Smart Account Features


## Why EIP-7702 Was Proposed?

Ethereum’s traditional model uses Externally Owned Accounts (EOAs) that are simple. they're tied to a private key and can only execute basic operations like signing transactions. This limits how users and developers can interact with the blockchain complex features like `batch transactions`, `gas sponsorship`, or `spending limits` require full smart contract wallets, which introduce friction and fragmentation.

**EIP-7702**, introduced as part of the **Pectra upgrade**, aims to bridge this gap. It allows EOAs to **temporarily behave like smart contract accounts for a single transaction**, enabling advanced features without requiring users to migrate to new wallets or smart contract accounts.

This proposal addresses key developer and UX pain points:

* Enables **transaction batching** (e.g., approval + swap in one go), reducing gas and complexity .
* Supports **gas sponsorship**, allowing third parties to pay fees making onboarding smoother and enabling metatransactions.
* Allows **privilege de-escalation**, such as sub-keys limited to certain permissions (like spending caps) improving usability without compromising control.
* Offers features like **social recovery**, **alternative authentication**, and **token-based gas payments** enhancing UX and flexibility.
* Aligns with the broader vision of **account abstraction** while preserving EOA identity and compatibility.

EIP-7702 delivers smart account capabilities directly in EOAs, unlocking powerful features without sacrificing simplicity or compatibility.

---

## What Is EIP-7702 & Why It Matters

EIP-7702 introduces a new transaction type **`0x04`** `setcode`that enables an EOA to include smart contract-like execution logic (via a "delegation designator") only for that transaction. After it's processed, the EOA reverts to its standard behavior

This clever enhancement preserves the EOA's address, private key, and UX while unlocking:

* **Batch operations**
* **Gas-free or sponsored interactions**
* **Custom policy logic** (like spending limits or social recovery fallback)
* **Broader compatibility with ERC-4337 and account abstraction plans**.

---

## EIP-7702 Integration Steps

### Step 1: Initialize a Foundry Project & Set EVM Version

```bash
forge init eip7702-foundry
cd eip7702-foundry
```

In your `foundry.toml`, specify an EVM version supporting the necessary features:

```toml
[default]
evm_version = "prague"
```

This ensures Foundry’s cheatcodes like `signDelegation` and `attachDelegation` are available.

---

### Step 2: Write a Delegate Contract in Solidity

Create `src/SimpleDelegate.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

contract SimpleDelegate {
    struct Call { bytes data; address to; uint256 value; }
    event Executed(address indexed to, uint256 value, bytes data);

    function execute(Call[] memory calls) external payable {
        for (uint i = 0; i < calls.length; i++) {
            (bool success, bytes memory result) =
                calls[i].to.call{value: calls[i].value}(calls[i].data);
            require(success, string(result));
            emit Executed(calls[i].to, calls[i].value, calls[i].data);
        }
    }

    receive() external payable {}
}
```

This contract enables batching of arbitrary calls—ideal for EIP-7702 delegation.


---

### Step 3: Write Foundry Tests with Delegation Cheatcodes

Create `test/SimpleDelegate.t.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "forge-std/Test.sol";
import "../src/SimpleDelegate.sol";

contract EIP7702Test is Test {
    SimpleDelegate public delegate;

    function setUp() public {
        delegate = new SimpleDelegate();
    }

    function testBatchCallWithDelegation() public {
        // Sign delegation and attach in one shot
        vm.signAndAttachDelegation(address(delegate), vm.envUint("PRIVATE_KEY"));

        // Execute batched transaction via delegate
        SimpleDelegate.Call[] memory calls = new SimpleDelegate.Call[](1);
        calls[0] = SimpleDelegate.Call({
            to: address(this),
            value: 0,
            data: abi.encodeWithSignature("dummyFunc()")
  02 delegation
        delegate.execute{value: 0}(calls);
        // Add appropriate assertions/events here
    }

    function dummyFunc() public {}
}
```

This illustrates signing and attaching the delegation in one step using Foundry’s built-in cheatcodes.

---

### Step 4: Deploy 

Run your tests with:

```bash
forge test -vv
```

Foundry will deploy the delegate contract, apply the signed delegation to your EOA, and allow execution of batch logic as if the EOA were a contract all within the test environment.

---

### Step 5: Send Delegation via CLI with `cast`

You can also use Foundry’s `cast send` command to broadcast an actual EIP-7702 transaction:

```bash
cast send 0xYourEOAAddress --auth 0xSignedAuthorization --rpc-url http://localhost:8545
```

* `--auth` accepts a hex-encoded signed authorization (chainId, delegate address, nonce, signature parts).

Pair this with deployme and a prepared authorization list to complete delegation on a real or forked network.

---

## When to Choose EIP-7702 vs ERC-4337

Use **EIP-7702 when**:

* Maintaining the same EOA address across chains matters.
* You need lightweight upgrades with minimal upfront gas.
* Backward compatibility and simpler UX are priorities. ([Alchemy][7])

Use **ERC-4337 when**:

* You require full programmability and customizability in account logic.

**Combining Both**: Use EIP-7702 for address continuity, and ERC-4337 for deep abstraction capabilities.

---

### Ecosystem & Tool Support

* **Hardhat + GitHub Example**: Check out the `woogie96/eip7702-example` repo, a full Hardhat project demonstrating batch delegation logic [Luganodes | Hassle-Free Staking](https://www.luganodes.com/blog/ethereum-pectra-eip/).
* **MetaMask Delegation Toolkit**: Offers developer flow to upgrade EOAs into smart accounts with minimal friction [ethereum.org](https://ethereum.org/te/roadmap/pectra/7702/).
* **Wallet UX Templates & dApp Interfaces**: Enhanced payment flows—for example, paying gas with USDC or other tokens are already being rolled out by wallets like MetaMask and platforms supporting ERC-5792/6900 standards [Bitpowr](https://bitpowr.com/blog/the-ethereum-pectra-upgrade-how-eip-7702-will-transform-ethereum-s-ux-with-stablecoin-gas-payments-and-more), [Metana](https://metana.io/blog/ethereum-pectra-upgrade/)

### TL;DR

1. **Deploy** delegate smart contract
2. **Sign** delegation authorization
3. **Send** `0x04` setcode transaction via your preferred tooling
4. **Use** the EOA for smart account behavior
5. **Revoke** delegation when needed

# About Us

At SC Audit Studio, we specialize in protocols security assessments. Our team of experts is dedicated to ensuring the safety and reliability of your projects. Partner with us to enhance your project's security and gain peace of mind.

[Reach out to us](https://x.com/SCAuditStudio) for queries and security assessments!
