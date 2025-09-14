# Vercles introduces x402-mcp: Risks of handling payments between MCP tools using stablecoins

## Table of Contents
1. [Introduction](#introduction)  
2. [Why is x402 needed?](#why-is-x402-needed)  
3. [Core Components of x402](#core-components-of-x402)  
4. [How does x402 work?](#how-does-x402-work)  
5. [Risks and Limitations of the Current x402 Implementation](#risks-and-limitations-of-the-current-x402-implementation)  
   - [Centralization Risks](#centralization-risks)  
   - [Supply Chain Attacks](#supply-chain-attacks)  
   - [AI Model Exploits](#ai-model-exploits)  
   - [Smart Contract Vulnerabilities](#smart-contract-vulnerabilities)  
   - [Compliance Risks](#compliance-risks)  
6. [About Us](#about-us)  
7. [FAQ](#faq)

## Introduction

AI agents are becoming more capable at handling complex tasks, 
but a challenge arises when they try to use paid external services. 
Today’s approach requires registering with each API, managing access keys, and maintaining separate billing accounts. 
This setup falls apart when an agent needs to independently discover and connect with new services.

Vercel intends to solve this by creating the [x402](https://www.x402.org/) protocol, which adds payments directly into the http request.
It uses the 402 "Payment required" status code, to let every endpoint request payment without account creation.
Vercel build x402 to work with MCP servers as well as their AI SDK.

## Why is x402 needed?

Developers creating agents often face the same issue: while agents can call APIs or MCPs, 
handling payments still demands manual setup and account registration, exchanging keys, and configuring billing. 
The x402 protocol solves this by embedding programmatic payments directly into the standard HTTP request flow.

Unlike proprietary payment APIs, x402 is an open web protocol. 
It isn’t bound to a single provider and can work across multiple payment networks and schemes.

## Core Components of x402

Integration with x402 is simple. On the server side, when defining API endpoints, users can add a middleware component:
```js
import { paymentMiddleware } from "x402-next";

export const middleware = paymentMiddleware(
  {
    "/protected": {
      price: 0.01,
      config: {
        description: "Access to protected content"
      }
    },
  }
);
```
On the client side, a simple wrapper for the request is enough:

```js
import { wrapFetchWithPayment } from "x402-fetch";

const fetchWithPay = wrapFetchWithPayment(fetch, client);
const response = await fetchWithPay("https://api.example.com/paid-endpoint");
```

For MCP servers, x402-mcp is a light wrapper around the existing mcp-handler package. 
The integration is similar to the above; in this article, however, we won't focus on that.
We advise you to visit the official docs at https://x402.gitbook.io/x402/core-concepts/client-server.

## How does x402 work?

The protocol works through a standard HTTP flow:

- The client requests a protected resource.
- The server replies with a 402 status code and payment instructions.
- The client includes a payment authorization in the header and retries the request.
- The server verifies the payment with an external facilitator.
- The server returns the resource, returning payment results in the header.

Most current implementations settle using exact, one-time payments in USDC on the Base blockchain, allowing fast, accountless transactions. 
However, the x402 standard itself is payment-network agnostic, supporting other currencies and schemes, including non-crypto options.
The current implementation requires tokens to be EIP-3009 compliant.

## Risks and Limitations of the Current x402 Implementation

The current implementation of x402 works most efficiently with stablecoins. However, like any payment technology, it introduces risks that should not be overlooked.

### Centralization Risks
Using USDC on Base, together with Coinbase-managed wallets, raises concerns about centralization. 
While the protocol itself does not depend on decentralized features, this setup could enable blacklisting of non-compliant companies. 
Such outcomes may be unlikely in practice, but they challenge the notion of x402 as an “open” standard. Importantly, x402 only defines the protocol and is not inherently tied to specific tokens or wallet providers.

### Supply Chain Attacks
Another risk is supply chain compromise. Recent incidents have shown that even well-maintained npm packages can be targeted. 
If libraries published by providers such as Vercel were ever compromised, it could severely impact integrations. 
Mitigation strategies include pinning package versions and implementing safeguards for fund management.

### AI Model Exploits
A different class of vulnerabilities arises from AI-driven agents using x402. 
The protocol itself does not enforce protections against overspending, and it cant determine whether a request price is “fair.” 
Malicious actors could bait AI models into exhausting funds on overpriced or useless APIs. 
To reduce this risk, integrations should establish strict spending policies and rules for online transactions.

### Smart Contract Vulnerabilities
When APIs accept payments through smart contracts, attackers may exploit callback functions to launch new types of attacks. 
This requires carefull consideration when sending payments to smart contracts, or accpeting payments using smart contracts.

### Compliance Risks
Regulatory compliance presents significant challenges. 
Accepting stablecoin payments globally exposes companies to issues such as VAT accounting, sanctions, and KYC requirements. 
Meeting these obligations may remove some of the openness and simplicity that x402 aims to provide.

## About Us

At SC Audit Studio, we specialize in protocols security assessments. 
Our team of experts has worked with companies like Aave, 1Inch and several more to conduct security assessments. 
Partner with us to enhance your project's security and gain peace of mind.

[Reach out to us](https://x.com/SCAuditStudio) for queries and security assessments!

## FAQ

[
  {
    "question": "What is the x402 protocol?",
    "answer": "x402 is an open web protocol introduced by Vercel that integrates payments directly into the HTTP request. It enables APIs and MCP tools to request payment without requiring account creation or separate billing setups."
  },
  {
    "question": "Why is x402 needed for AI agents?",
    "answer": "AI agents often face difficulties handling payments because traditional APIs require manual setup. x402 solves this problem by embedding programmatic payments into standard HTTP requests."
  },
  {
    "question": "How does x402 handle payments?",
    "answer": "The client requests a protected resource, the server responds with a 402 status and payment instructions, the client retries with payment authorization, the server verifies, and finally the resource is returned along with payment results in the header."
  },
  {
    "question": "What are the main risks of using x402?",
    "answer": "Risks include centralization from reliance on USDC and Coinbase, supply chain attacks on dependencies, AI model exploits leading to overspending, vulnerabilities in smart contract payment flows, and regulatory compliance challenges such as KYC and tax obligations."
  },
  {
    "question": "Is x402 limited to crypto payments?",
    "answer": "No. While current implementations commonly use USDC on the Base blockchain for fast, accountless transactions, the x402 standard is payment-network agnostic and can support other currencies and schemes, including non-crypto options."
  }
]
