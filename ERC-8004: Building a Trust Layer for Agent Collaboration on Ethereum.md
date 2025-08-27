# ERC-8004: Building a Trust Layer for Agent Collaboration on Ethereum

## Table of Contents

1. [Introduction](#introduction)
2. [Why ERC-8004 Exists](#why-erc-8004-exists)
3. [Core Components of ERC-8004](#core-components-of-erc-8004)
   - [Identity Registry](#1-identity-registry)
   - [Reputation Registry](#2-reputation-registry)
   - [Validation Registry](#3-validation-registry)
4. [Community Insights & Enhancements](#community-insights--enhancements)
5. [Real-World in Action: ChaosChain's Demo](#real-world-in-action-chaoschainos-demo)
6. [Developer Quickstart: Building with ERC-8004](#developer-quickstart-building-with-erc-8004)
   - [Register Agent Identity](#1-register-agent-identity)
   - [Enable Reputation Feedback](#2-enable-reputation-feedback)
   - [Register Validation Flow](#3-register-validation-flow)
   - [Run Off-Chain Indexers](#4-run-off-chain-indexers)
   - [Build Trust-Aware Applications](#5-build-trust-aware-applications)
7. [Conclusion](#conclusion)
8. [About Us](#about-us)

---

## Introduction

In a decentralized world where autonomous agents (AI, bots, dApps) must collaborate across organizational boundaries, trust becomes the missing link. **ERC-8004**, titled **Trustless Agents**, introduces a minimal but robust on-chain infrastructure that provides agent identity, reputation, and validation while preserving flexibility and modularity.

Created by Marco De Rossi, Davide Crapis, and Jordan Ellis (August 13, 2025), this proposal extends the Agent-to-Agent (A2A) Protocol with trust primitives that are interoperable, lightweight, and future-proof. [Ethereum Improvement Proposals](https://eips.ethereum.org/EIPS/eip-8004)



## Why ERC-8004 Exists

A2A already enables skill advertising, task orchestration, and messaging among agents, but only in trusted silos. ERC-8004 addresses the **cross-organizational trust gap** by:

* Preserving minimalist on-chain logic for scalability
* Offering **pluggable trust models** (reputation, stake, TEE attestations)
* Leveraging standards like CAIP-10 and RFC-8615 for interoperability and discovery 

[Ethereum Improvement Proposals](https://eips.ethereum.org/EIPS/eip-8004)

The result: a modular, composable trust layer for any Ethereum-compatible environment.

## Core Components of ERC-8004

ERC-8004 implements a three-registry architecture that separates concerns while enabling powerful combinations:

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Identity      │    │   Reputation    │    │   Validation    │
│   Registry      │◄──►│   Registry      │◄──►│   Registry      │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Agent Cards    │    │  Feedback Data  │    │ Validation Data │
│  (Off-chain)    │    │  (Off-chain)    │    │  (Off-chain)    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```


### 1. **Identity Registry**

Agents register their on-chain identity (AgentID), domain (AgentDomain), and address (AgentAddress). Off-chain AgentCards, discoverable via well-known URIs, link back to this identity registry. 

### 2. **Reputation Registry**

Clients pre-authorize feedback by emitting an `AuthFeedback` event. Feedback itself is stored off-chain but tied back to on-chain events and optionally enriched with proof-of-payment hooks suggested by the Ethereum Magicians discussion. 

### 3. **Validation Registry**

Server agents commit output via `DataHash`, which validators then assess. The validation response (0--100 score) is submitted on-chain. This supports both stake-driven and cryptographic validation models.

[Ethereum Improvement Proposals](https://eips.ethereum.org/EIPS/eip-8004)

## Community Insights & Enhancements

The Ethereum Magicians thread sparked vibrant discussion around keeping the protocol lean while enabling composability and optional extensibility:

* **On-chain composability**: Some wanted registry responses readable by smart contracts for enforcement logic (e.g., slashing).
* **Aggregate reputation scoring**: Others proposed optional on-chain functions to compute agent reputation scores from multiple attestations. 
* **Payment separation**: Contributors emphasized keeping payments out of ERC-8004 while enabling feedback to reference payment proofs.

[Fellowship of Ethereum Magicians](https://ethereum-magicians.org/t/erc-8004-trustless-agents/25098)

## Real-World in Action: ChaosChain’s Demo

ChaosChain built a full working demo using ERC-8004:

* **Agents**: Alice (Server), Bob (Validator), Charlie (Client)
* **Workflow**:

  * Agents registered via Identity Registry
  * Alice submits market analysis and triggers validation
  * Bob validates and submits a score on-chain
  * Alice authorizes Charlie to leave feedback
  * A full on-chain audit trail emerges
* **Architecture**: On-chain identity, validation, and reputation registries; off-chain AI logic, orchestration, and data packaging; content-addressed hashes for immutable linkage

[Mirror](https://mirror.xyz/0xBCCE64A97F240784765CF8d6A99f02C903a95043/DD2-ITkpLFlj4U7MBKDmWIqCbZXzhU8tTngZl755c-o?referrerAddress=0xBCCE64A97F240784765CF8d6A99f02C903a95043)


## Developer Quickstart: Building with ERC-8004

Use this quick guide to get started integrating ERC-8004 registries into your application workflows.

### **1. Register Agent Identity**

```solidity
identityRegistry.registerAgent(
  "myagent.example.com",  // AgentDomain hosting AgentCard
  agentAddress            // Your Ethereum-compatible agent address
);
```

Then publish an AgentCard at `https://myagent.example.com/.well-known/agent-card.json`:

```json
{
  "registrations": [
    {
      "agentId": 123,
      "agentAddress": "eip155:1:0x...",
      "signature": "0x..."
    }
  ],
  "trustModels": ["feedback","inference-validation","tee-attestation"]
}
```

### **2. Enable Reputation Feedback**

```solidity
reputationRegistry.acceptFeedback(clientAgentID, serverAgentID);
```

Then publish feedback off-chain (referenced by the event). Suggested schema:

```json
{
  "FeedbackAuthID": "eip155:1:0xabc",
  "Rating": 90,
  "ProofOfPayment": { /* optional proof */ }
}
```

### **3. Register Validation Flow**

```solidity
validationRegistry.validationRequest(validatorAgentID, serverAgentID, dataHash);
```

Validator responds:

```solidity
validationRegistry.validationResponse(dataHash, responseScore);  // 0--100
```

### **4. Run Off-Chain Indexers**

Deploy a subgraph or crawler to:

* Read AgentCards and trust model metadata
* Process `AuthFeedback` and `ValidationResponse` events
* Compute reputation or trigger workflow logic

### **5. Build Trust-Aware Applications**

With trust data available, your applications can:

* Discover agents
* Filter by validation or reputation thresholds
* Automate task workflows with offline oracles
* Display transparent, audit-ready trust histories

### Conclusion

ERC-8004 is a significant step towards establishing a robust trust framework for decentralized AI agents. By integrating identity, reputation, and validation mechanisms, it empowers agents to interact with confidence and accountability.

## About Us

At SC Audit Studio, we specialize in protocols security assessments. Our team of experts is dedicated to ensuring the safety and reliability of your projects. Partner with us to enhance your project's security and gain peace of mind.

[Reach out to us](https://x.com/SCAuditStudio) for queries and security assessments!
