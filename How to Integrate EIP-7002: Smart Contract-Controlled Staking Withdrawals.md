
# How to Integrate EIP-7002: Smart Contract-Controlled Staking Withdrawals

## Table of Contents

1. [Why Staking Needed an Upgrade?](#why-staking-needed-an-upgrade)
2. [The Smart Contract Era of ETH Withdrawals](#the-smart-contract-era-of-eth-withdrawals)
3. [How to Integrate EIP-7002 Step by Step](#how-to-integrate-eip-7002-step-by-step)

    * [Step 1: Project Bootstrap (Foundry)](#step-1-project-bootstrap-foundry)
    * [Step 2: Core Contracts (WithdrawalManager + Mocks)](#step-2-core-contracts-withdrawalmanager--mocks)
    * [Step 3: Foundry Tests (Simulating Beacon Withdrawals)](#step-3-foundry-tests-simulating-beacon-withdrawals)
    * [Step 4: Deployment Script](#step-4-deployment-script)
    * [Step 5: Integrations (Lido, EigenLayer, Gnosis Safe)](#step-5-integrations-lido-eigenlayer-gnosis-safe)
4. [Real-World Use Cases of EIP-7002](#real-world-use-cases-of-eip-7002)
5. [Best Practices for EIP-7002 Contracts](#best-practices-for-eip-7002-contracts)

    * [Security Considerations](#security-considerations)
    * [Gas & Timing Considerations](#gas--timing-considerations)
    * [TL;DR](#tldr)
6. [About Us](#about-us)
7. [FAQ](#faq)

## Why Staking Needed an Upgrade?

Ethereum’s staking withdrawals before **Pectra** were restrictive:

* Validators could only set their **withdrawal credential** to an Externally Owned Account (EOA).
* Funds would flow to a plain wallet address, with **no automation, routing, or programmability**.
* If a staker wanted to **restake ETH, delegate to a pool, or direct funds into DeFi**, it required manual intervention.

This rigidity slowed down **restaking adoption**, added **operational friction**, and limited innovation in validator services.

**EIP-7002**, introduced in the **Pectra upgrade**, solves this by allowing validator withdrawals to target **smart contract addresses**. This makes withdrawals **programmable**, enabling automated restaking, revenue distribution, and DAO-level staking flows.

## The Smart Contract Era of ETH Withdrawals

EIP-7002 allows a validator to set their withdrawal credentials to a **smart contract** instead of just an EOA.

When withdrawals (either partial or full) occur, ETH is sent directly into that contract, which can then execute custom logic:

* **Automated Restaking** → Instantly redeposit ETH into EigenLayer or similar protocols.
* **Pooling & Revenue Sharing** → Distribute ETH to multiple stakeholders (e.g., a validator DAO).
* **Treasury Management** → Route ETH into lending, liquidity, or hedging strategies.
* **Access Controls** → Protect funds via multisig, timelocks, or upgradeable modules.

This transforms validator rewards from **static flows** into **programmable assets**.

## How to Integrate EIP-7002 Step by Step

### Step 1: Project Bootstrap (Foundry)

```bash
forge init eip7002-withdrawal
cd eip7002-withdrawal
```

Update `foundry.toml` to lock compiler version:

```toml
[default]
solc_version = "0.8.20"
```

Create project folders:

```bash
mkdir -p src src/mocks script test
```

### Step 2: Core Contracts (WithdrawalManager + Mocks)

**WithdrawalManager.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface ILido {
    function submit(address referral) external payable returns (uint256);
}

interface IRestake {
    function restakeFor(address beneficiary) external payable returns (bool);
}

contract WithdrawalManager {
    address public owner;
    address public treasury;
    ILido public lido;
    IRestake public restake;
    bool private _locked;
    uint256 public pendingBalance;

    event Received(address indexed sender, uint256 amount);
    event Forwarded(address indexed to, uint256 amount);
    event Restaked(uint256 amount);
    event DepositedToLido(uint256 amount, uint256 shares);
    event PendingQueued(uint256 amount);
    event ProcessedPending(address indexed by, uint256 amount);

    modifier onlyOwner() {
        require(msg.sender == owner, "owner only");
        _;
    }

    modifier noReentrant() {
        require(!_locked, "reentrant");
        _locked = true;
        _;
        _locked = false;
    }

    constructor(address _treasury, address _lido, address _restake) {
        owner = msg.sender;
        treasury = _treasury;
        lido = ILido(_lido);
        restake = IRestake(_restake);
    }

    receive() external payable {
        emit Received(msg.sender, msg.value);
        _immediateStrategy(msg.value);
    }

    function _immediateStrategy(uint256 amount) internal noReentrant {
        if (amount == 0) return;
        uint256 half = amount / 2;

        (bool okTreasury, ) = treasury.call{value: half}("");
        if (okTreasury) emit Forwarded(treasury, half);
        else { pendingBalance += half; emit PendingQueued(half); }

        try restake.restakeFor{value: amount - half}(owner) returns (bool success) {
            if (success) emit Restaked(amount - half);
            else { pendingBalance += (amount - half); emit PendingQueued(amount - half); }
        } catch {
            pendingBalance += (amount - half);
            emit PendingQueued(amount - half);
        }
    }

    function processPending(uint256 amount) external onlyOwner noReentrant {
        require(amount > 0 && amount <= pendingBalance && amount <= address(this).balance, "invalid amount");
        pendingBalance -= amount;

        uint256 half = amount / 2;
        (bool okTreasury, ) = treasury.call{value: half}("");
        if (okTreasury) emit Forwarded(treasury, half);
        else pendingBalance += half;

        try restake.restakeFor{value: amount - half}(owner) returns (bool success) {
            if (success) emit Restaked(amount - half);
            else pendingBalance += (amount - half);
        } catch {
            pendingBalance += (amount - half);
        }

        emit ProcessedPending(msg.sender, amount);
    }

    function setTreasury(address t) external onlyOwner { treasury = t; }
    function setLido(address l) external onlyOwner { lido = ILido(l); }
    function setRestake(address r) external onlyOwner { restake = IRestake(r); }
    function transferOwnership(address newOwner) external onlyOwner { owner = newOwner; }

    function emergencyWithdraw(address payable to) external onlyOwner noReentrant {
        uint256 bal = address(this).balance;
        require(bal > 0, "no balance");
        (bool ok, ) = to.call{value: bal}("");
        require(ok, "withdraw failed");
    }
}
```

**Mocks**

`src/mocks/MockRestake.sol`

```solidity
pragma solidity ^0.8.20;

contract MockRestake {
    event RestakedFor(address indexed beneficiary, uint256 amount);
    receive() external payable {}
    function restakeFor(address beneficiary) external payable returns (bool) {
        emit RestakedFor(beneficiary, msg.value);
        return true;
    }
}
```

`src/mocks/MockTreasury.sol`

```solidity
pragma solidity ^0.8.20;

contract MockTreasury {
    receive() external payable {}
}
```

### Step 3: Foundry Tests (Simulating Beacon Withdrawals)

`test/WithdrawalManager.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/WithdrawalManager.sol";
import "../src/mocks/MockRestake.sol";
import "../src/mocks/MockTreasury.sol";

contract WithdrawalManagerTest is Test {
    WithdrawalManager manager;
    MockRestake restake;
    MockTreasury treasury;

    function setUp() public {
        restake = new MockRestake();
        treasury = new MockTreasury();
        manager = new WithdrawalManager(address(treasury), address(0), address(restake));
    }

    function testImmediateForwardAndRestake() public {
        uint256 sendAmt = 1 ether;
        uint256 beforeTreasury = address(treasury).balance;
        uint256 beforeRestake = address(restake).balance;

        (bool ok,) = address(manager).call{value: sendAmt}("");
        require(ok);

        assertEq(address(treasury).balance, beforeTreasury + (sendAmt / 2));
        assertEq(address(restake).balance, beforeRestake + (sendAmt / 2));
    }
}
```

Run tests:

```bash
forge test -vv
```

### Step 4: Deployment Script

`script/Deploy.s.sol`

```solidity
pragma solidity ^0.8.20;
import "forge-std/Script.sol";
import "../src/WithdrawalManager.sol";

contract Deploy is Script {
    function run() external {
        address treasury = vm.envAddress("TREASURY");
        address lido = vm.envAddress("LIDO");
        address restake = vm.envAddress("RESTAKE");

        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        new WithdrawalManager(treasury, lido, restake);
        vm.stopBroadcast();
    }
}
```

Deploy:

```bash
export RPC_URL="https://rpc.testnet.example"
export PRIVATE_KEY="0x..."
export TREASURY="0xYourTreasury"
export LIDO="0xLidoAddress"
export RESTAKE="0xRestakeAddress"

forge script script/Deploy.s.sol:Deploy --rpc-url $RPC_URL --broadcast
```

> [!WARNING]
> **Disclaimer:** The following code examples are for **illustrative purposes only** and have **not been audited**. Do **not** deploy them directly in production.


## Real-World Use Cases of EIP-7002

EIP-7002 turns passive rewards into **programmable flows**. Key patterns:

* **Auto-Restake Loops** – Rewards auto-route into EigenLayer / LST minting for **compounding yield**.
* **Validator-as-a-Service Splits** – Contracts **trustlessly split** flow between operator, delegators, treasury.
* **DAO Treasury Streaming** – ETH lands directly in **governed treasuries**: restake, diversify, or fund ops.
* **DeFi Pipelines** – Immediate conversion to **stETH / rETH**, LP provisioning, or **lending collateral**.
* **Safety Buffers** – Slice a % to **insurance / slashing reserves** creating self-healing economics.

Bottom line: **EIP-7002 = autonomous validator economies** tightly integrated with **restaking + DeFi**.

## Best Practices for EIP-7002 Contracts

### Security Considerations

* Keep logic **minimal** in withdrawal contracts.
* Use **try/catch** for all external integrations.


### Gas & Timing Considerations

* Withdrawals are **protocol-driven** you don’t pay gas directly.
* But failed executions can cause ETH to get stuck → always have a **pending queue** + manual processing fallback.

### TL;DR

1. Set validator withdrawal credentials to your `WithdrawalManager`.
2. Contract receives ETH automatically from consensus layer.
3. Forward funds into DAO treasuries, Lido, EigenLayer, or multisigs.
4. Keep contracts minimal + audited.
5. Use fallback safety (queue pattern) to avoid loss.

## About Us

At SC Audit Studio, we specialize in protocols security assessments. Our team of experts is dedicated to ensuring the safety and reliability of your projects. Partner with us to enhance your project's security and gain peace of mind.

[Reach out to us](https://x.com/SCAuditStudio) for queries and security assessments!

## FAQ

[
    {
        "question": "What is EIP-7002?",
        "answer": "EIP-7002 lets Ethereum validators set their withdrawal credentials to a smart contract instead of an EOA, making withdrawals programmable."
    },
    {
        "question": "Why does it matter?",
        "answer": "It unlocks automated restaking, revenue splitting, DAO treasury routing, and direct integration with DeFi + restaking protocols."
    },
    {
        "question": "Is using a contract withdrawal address risky?",
        "answer": "Yes. A vulnerable contract could permanently lose validator rewards. Use minimal, audited, upgrade-safe patterns with fallback controls."
    },
    {
        "question": "How does this relate to EIP-7251 (validator consolidation)?",
        "answer": "With higher validator balance caps (up to 2048 ETH), routing larger flows through contracts amplifies the need for secure automated handling."
    }
]
