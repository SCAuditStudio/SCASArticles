# OPSEC in Web3: How the Weakest Link Became the Most Expensive One

Web3 security used to be discussed mainly as a smart contract problem. Years of audits, formal reviews, and bug hunting around Solidity bugs, reentrancy, and integer overflows reduced many of those failures. DeFi-specific exploits dropped 89% year-over-year in Q1 2026. Total losses stayed high because attackers moved toward the systems that prepare, display, and verify transactions before they reach the chain.

In the cases below, the compromised layer sat off chain: the frontend serving layer, the developer workstations used for signing, and the Remote Procedure Call (RPC) nodes that supplied state data to bridge verifiers. The contracts executed the transactions they were given. The surrounding infrastructure supplied false transaction data and false context.

This piece examines three incidents, Radiant Capital (October 2024, $53M), Bybit (February 2025, $1.5B), and Kelp DAO (April 2026, $292M), as a technical sequence. Together, they show a class of attacks in which cryptography remains intact while attackers corrupt the information layer that humans and verifiers depend on. All three are attributed, with high confidence, to North Korea's Lazarus Group.

## Table of Contents

1. [Attack Vector 1: Malware Between the Wallet Interface and Hardware Wallet, Radiant Capital](#attack-vector-1-malware-between-the-wallet-interface-and-hardware-wallet-radiant-capital)
2. [Attack Vector 2: JavaScript Injection in the Safe Frontend, Bybit](#attack-vector-2-javascript-injection-in-the-safe-frontend-bybit)
3. [Attack Vector 3: Poisoned RPC Nodes and Forced Failover, Kelp DAO](#attack-vector-3-poisoned-rpc-nodes-and-forced-failover-kelp-dao)
4. [The Pattern](#the-pattern)
5. [What These Controls Miss](#what-these-controls-miss)
6. [What the Attack Surface Looks Like](#what-the-attack-surface-looks-like)
7. [Practical Implications](#practical-implications)
8. [Conclusion](#conclusion)
9. [About Us](#about-us)
10. [FAQ](#faq)



## Attack Vector 1: Malware Between the Wallet Interface and Hardware Wallet, Radiant Capital

**Radiant's 3-of-11 multisig created eleven signer targets.** Radiant Capital used this setup to govern protocol parameter changes and contract upgrades. The configuration improved availability and prevented any one signer from acting alone. It also gave an attacker more people and machines to approach before needing to compromise any three.

### Delivery: Telegram Message Carrying INLETDRIFT

On September 11, 2024, 35 days before the exploit, a Radiant developer received a Telegram message from a threat actor impersonating a former contractor. The message requested peer review on a "Penpie Hack Analysis," a topic relevant to a DeFi security team. The ZIP file contained a decoy PDF with a security-analysis document alongside `INLETDRIFT`, a macOS-targeting backdoor that established persistence via a malicious AppleScript communicating with `atokyonews[.]com`.

The developer opened the file because it matched the team's day-to-day work. The malware installed without unusual permission prompts and avoided antivirus detection. When the file was forwarded internally for team feedback, the malware spread to additional developer machines. The attacker now had persistent access to multiple signing devices.

### Execution: Man-in-the-Middle Attack on the Signing Interface

Radiant's team used normal signing controls: simulating every transaction in Tenderly before signing, checking payload data at each step, using a mix of operating systems and hardware wallet models across signers, and reviewing the Safe{Wallet} web interface before each approval. Those controls did not catch the payload swap.

`INLETDRIFT` sat between the wallet interface and the hardware device at the operating-system layer. The Safe{Wallet} frontend showed legitimate transaction data, while the payload delivered to the hardware wallets was different. The malware intercepted the signing request, replaced the payload, and forwarded the signed malicious transaction back to the interface. The result was three valid hardware wallet signatures on a transaction that upgraded Radiant's Pool Provider contract to an attacker-controlled implementation.

```
[Radiant Signer Device]
        |
        | Screen shows: "Routine parameter update, destination: 0xabc..."
        |
[INLETDRIFT man-in-the-middle layer] ← payload swap happens here
        |
        | Hardware wallet receives: "upgradeTo(0xattacker_contract)"
        |
[Ledger / Trezor]
        |
        | Valid cryptographic signature returned
        |
[Safe{Wallet} collects 3 signatures]
        |
[Pool Provider contract upgraded → drained]
```

The attack did not require private-key theft. The signing devices did what they were built to do: sign the payload they received. The compromised layer was the software path that prepared and forwarded that payload.

### Losses and Attribution

$53M was drained from BSC and Arbitrum lending pools. Mandiant's post-mortem formally attributed the campaign to `UNC4736` (Citrine Sleet), a Lazarus Group subcluster, in a report published December 2024.



## Attack Vector 2: JavaScript Injection in the Safe Frontend, Bybit

**Bybit's workflow depended on the Safe web interface showing the same transaction that signers were approving.** Bybit used Safe{Wallet} (formerly Gnosis Safe) to manage an Ethereum cold wallet through a standard multisig flow: a proposer initiates a transaction via the `app.safe.global` web interface, the transaction is queued, co-signers review and approve through the same interface, and once the threshold is met, execution proceeds.

### Delivery: Compromised Developer Machine → S3 Bucket Access

The attack began in early February 2025 when a Safe{Wallet} developer's machine was compromised, almost certainly through social engineering consistent with TraderTraitor behavior. The attacker obtained the developer's AWS session tokens.

With those credentials, they accessed Safe{Wallet}'s AWS S3 bucket serving the Next.js frontend application for `app.safe.global`. On February 19, 2025, they modified the JavaScript bundle at `_next/static/chunks/pages/_app-52c9031bfa03da47.js`. The injected code left Safe{Wallet}'s behavior unchanged for general users and activated only when the transaction initiator matched Bybit's cold wallet address. For Bybit, the frontend silently constructed a different payload before signers approved it.

### Execution: Hidden Safe Upgrade

When Bybit's authorized signers accessed the Safe web interface on February 21, 2025, they saw a routine internal transfer. The destination address and amount looked expected, and the contract interaction appeared normal. The transaction queued for signature was a `delegatecall`, a contract call that runs another contract's code while using the Safe's own storage. In this case, the attacker-deployed contract called `upgradeTo()` on the proxy and replaced the Safe implementation with attacker-controlled bytecode.

```solidity
// What signers thought they were approving:
// transfer(recipient, 401347 ETH)

// What their Ledgers actually signed:
safe.execTransaction(
    to: malicious_contract,
    value: 0,
    data: abi.encodeWithSelector(
        UPGRADE_TO_AND_CALL_SELECTOR,
        attacker_implementation,
        drain_calldata
    ),
    operation: DelegateCall  // ← executes in Safe's storage context
)
```

Once the implementation was swapped, the attacker controlled the Safe's execution logic. All assets, 401,347 ETH, were drained in a single subsequent transaction. About two minutes after execution, the attacker removed the malicious JavaScript from the S3 bucket to reduce forensic evidence. The Wayback Machine captured both the clean and modified versions.

### Why Hardware Wallets Didn't Help

The Ledger and Trezor devices in Bybit's signing workflow display encoded transaction data, meaning the low-level instructions sent to a smart contract. In theory, a signer reviewing a `delegatecall` to an unknown implementation contract should reject it. In practice, complex multisig transaction data is difficult to read on a small hardware-wallet screen. An `execTransaction` call with nested contract-call encoding does not plainly say, "you are upgrading this contract to attacker code." It appears as hex data that no signer is expected to manually decode.

The malicious JavaScript used this gap: the web interface showed what humans could verify, while the signing payload contained what could drain the wallet.

### Losses and Attribution

$1.5B, the largest single theft in cryptocurrency history. The FBI formally attributed the attack to TraderTraitor on February 26, 2025. Assets were laundered through a combination of mixers, cross-chain bridges, and OTC brokers at a pace that outran most freeze attempts.



## Attack Vector 3: Poisoned RPC Nodes and Forced Failover, Kelp DAO

**Kelp's loss came from a bridge verifier accepting false source-chain data.** Kelp DAO's rsETH token existed across multiple chains through a LayerZero Omnichain Fungible Token (OFT) bridge. The bridge's security model relied on a Decentralized Verifier Network (DVN): when tokens are burned on the source chain, a DVN reads the event and attests to it, triggering minting on the destination chain. Kelp operated a 1-of-1 DVN configuration, meaning one verifier's signature was enough.

### Delivery: Social Engineering → Session Key Theft → RPC Infrastructure Access

The attack chain began on March 6, 2026, when the attacker used social engineering against a LayerZero Labs developer to obtain session credentials. With those credentials, they accessed LayerZero's internal cloud infrastructure and located the RPC nodes used by the LayerZero Labs DVN to read source-chain state. RPC nodes are the servers that blockchain applications query for chain data. The attacker identified all RPC endpoints in the DVN's configuration, a mix of internal nodes and external providers, and gained access to two internal nodes.

Access to the nodes still left the attacker with a data-consistency problem. They patched the running RPC processes in memory so LayerZero tools would receive normal responses even as manipulated data was served to the DVN's signing requests. The patched nodes returned normal responses to health checks and general queries while serving crafted data to DVN-specific calls.

### Execution: Forced Failover and a Forged Cross-Chain Message

The two compromised internal nodes did not give the attacker a complete view of the DVN's data sources because the DVN also queried external RPC providers that would contradict the fabricated state. The attacker launched a distributed denial-of-service (DDoS) attack against the external providers, knocking them offline and forcing the DVN's RPC resolution to fail over to the compromised internal nodes.

With the DVN's view of source-chain state under attacker control, the attacker injected a fabricated cross-chain message: a phantom rsETH burn event that had never occurred on the source chain. The DVN validated the fake event against the poisoned RPC data. Because the DVN signed what its data sources showed it, the resulting attestation was valid even though the underlying event was fake. The keys were intact, the verifier logic followed its configured path, and the LayerZero protocol accepted a false input.

```
[Attacker's forged event]
"116,500 rsETH burned on source chain at block X, tx: 0xfake..."

         ↓ injected into DVN's RPC data feed

[LayerZero Labs DVN]
  Queries RPC nodes for burn event at block X
  Internal node 1 (compromised): "Confirmed."
  Internal node 2 (compromised): "Confirmed."
  External provider 1 (under DDoS): timeout
  External provider 2 (under DDoS): timeout

  Consensus reached. Attestation signed.

         ↓
[Ethereum bridge contract]
  Valid attestation received
  Mints 116,500 rsETH to attacker address
```

Post-execution, the malware installed on the compromised RPC nodes deleted itself and wiped local logs.

### The Configuration Fault Line

LayerZero and Kelp DAO disputed responsibility publicly. LayerZero initially blamed Kelp's 1-of-1 DVN choice; Kelp pointed to LayerZero's quickstart guide and default GitHub configuration, both of which demonstrated single-DVN setup as standard. An independent review found that roughly 40% of protocols on LayerZero were using the same configuration at the time of the exploit.

A multi-DVN setup would have required consensus across independent verifiers reading from independent infrastructure. Poisoning one DVN's RPC feed would have been detected as an outlier. The attack required the single point of failure to exist. **LayerZero has since eliminated support for 1-of-1 verifier configurations.**

### Losses and Attribution

$292M (116,500 rsETH). Attributed to Lazarus Group / TraderTraitor in a post-mortem prepared with Mandiant and CrowdStrike.



## The Pattern

These three attacks happened across different protocol types, but they relied on the same dependency: trusted off-chain data and interfaces.

| | Radiant Capital | Bybit | Kelp DAO |
|---|---|---|---|
| **Year** | 2024 | 2025 | 2026 |
| **Loss** | $53M | $1.5B | $292M |
| **Entry Point** | Developer workstation | Third-party dev workstation | Developer session keys |
| **Mechanism** | Malware swapping signing payloads | JavaScript injection in Safe frontend | Patched RPC nodes + DDoS failover |
| **What was bypassed** | Hardware wallet display | Hardware wallet display | Bridge verifier consensus |
| **Smart contract bug?** | No | No | No |
| **Attribution** | Lazarus (UNC4736) | Lazarus (TraderTraitor) | Lazarus (TraderTraitor) |

The shared weak point was an architectural assumption: off-chain software would accurately represent on-chain state and transaction intent to the humans and automated systems approving actions. Across all three attacks, that assumption failed. The contracts, keys, and cryptography behaved as designed. The information layer upstream of them did not.



## What These Controls Miss

**Hardware wallets** protect private keys. They still leave signers dependent on upstream software to build the contract call. If the web interface renders "transfer 100 ETH" and the encoded transaction data says "upgrade implementation to 0xattacker," the hardware wallet sees low-level transaction data rather than the English summary. Improving on-device transaction readability remains an open problem for the wallet ecosystem.

**Multi-sig thresholds** help only when signers are operationally independent. A 3-of-11 scheme with eleven collocated developers gives an attacker eleven machines to target. The threshold only matters if compromising one signer does not make the others easier to compromise.

**Transaction simulation** (Tenderly et al.) checks the payload it receives. In the Radiant case, simulation passed because Tenderly saw a legitimate transaction. The payload swap happened after simulation and before signing.

**Standard smart contract audits** stay inside bytecode and protocol logic. An audit of Radiant's contracts would have missed `INLETDRIFT`. An audit of Bybit's Safe deployment would have missed the compromised S3 bucket. An audit of Kelp's bridge contract would have missed the social engineering that compromised the LayerZero developer. These attacks operated in the systems that prepared, delivered, or verified inputs before the contracts executed them.



## What the Attack Surface Looks Like

**For many DeFi teams, the security perimeter now includes Web2 infrastructure.** Sherlock's 2026 quarterly report found that infrastructure attacks, private key compromise, cloud key management failures, and bridge validator compromise represented 76% of all classified incidents by count in Q1 2026. Smart contract exploits were 89% lower than the same period in 2025.

Private-key management has always been a core operational risk, so off-chain compromise is not a new category by itself. The shift comes from how much protocol control now runs through Web2 systems. Many products have moved toward "decentralized-centralized finance" (DeCeFI): smart contracts settle the final state, while centralized infrastructure prepares transactions, serves frontends, manages cloud credentials, and supplies verifier data. That structure reintroduces the trust assumptions smart contracts were supposed to reduce. A compromised workstation, cloud session, frontend build pipeline, or RPC feed can steer on-chain execution toward a false instruction.

The practical attack surface now includes developer workstations and operational security (OpSec) habits, third-party frontend dependencies and their content delivery network infrastructure, CI/CD pipelines with access to deployment credentials, RPC providers and their compromise detection, session key management for cloud infrastructure, and every person with signing authority. Lazarus Group has demonstrated that it will invest months of preparation for a single operation. The March-to-April timeline of the Kelp DAO attack, six weeks between initial access and execution, fits that pattern.



## Practical Implications

**For protocols managing multisig governance**, signing devices should be network-isolated during the signing session instead of remaining connected to the same operating system that prepares the transaction. Signers should verify encoded transaction data independently from the primary web interface, using a separate RPC endpoint and a separate device to decode what they are signing. Threshold-to-signer ratios matter in a specific way: 3-of-11 gives an attacker eight misses before they need to succeed; 3-of-5 with independent machines, networks, and physical locations is stronger in practice.

**For bridge and cross-chain infrastructure**, protocols holding significant value need DVN redundancy. Single-verifier configurations should be treated as critical vulnerabilities regardless of what a provider's default configuration or quickstart guide suggests. RPC node independence should mean different providers, different geographic regions, different infrastructure operators, and active anomaly detection on response divergence. Denial-of-service resilience for critical RPC endpoints should be part of the threat model before deployment.

**For all protocol teams**, developer machines belong in the threat model. Phishing resistance, verified file sources, sandboxed execution for received files, and hardware security keys for cloud access are part of protocol security. Frontend integrity checks through Subresource Integrity headers, reproducible builds, and signed release artifacts matter when the frontend mediates high-value transactions. Session key and cloud credential hygiene need the same rigor as private-key management because, as the Kelp DAO attack demonstrated, they can become equivalent attack surfaces.



## Conclusion

These incidents show that audited bytecode can still settle fraudulent outcomes when the systems around it feed it false instructions. The attack surface now includes everything a developer, signer, cloud system, frontend, or verifier touches between writing the contract and moving assets. Lazarus Group has spent years mapping that surface, moving from endpoint malware to supply chain injection to RPC infrastructure compromise.

The $1.8 billion total across these three incidents came from treating off-chain infrastructure as a secondary security concern. As audit coverage of on-chain code improves and smart contract exploits become rarer, this category of attack will represent a larger share of total losses until the industry applies the same rigor to operational security that it has applied to bytecode.



## About Us

At SC Audit Studio, we specialize in protocol security assessments. Our team of experts has worked with companies like Aave, 1inch, and many more to conduct thorough security assessments across EVM and non-EVM environments.

[Reach out to us](https://scauditstudio.com/contact) for queries and security assessments!



## Tags

["Web3", "OpSec", "Security", "Lazarus Group", "Bybit", "Radiant Capital", "Kelp DAO", "RPC Poisoning", "Private Key Compromise", "DeFi", "Supply Chain Attack", "INLETDRIFT"]



## FAQ

[
  {
    "question": "What is an off-chain attack in Web3?",
    "answer": "An off-chain attack targets the infrastructure surrounding a blockchain protocol rather than the smart contracts themselves. This includes developer workstations, frontend applications, content delivery network infrastructure, RPC nodes, cloud credentials, and any software that mediates between a human approver and the on-chain transaction. In the three cases examined here, attackers compromised the off-chain layer that produced, displayed, or verified transaction data."
  },
  {
    "question": "How did the Bybit hack work technically?",
    "answer": "Attackers compromised a Safe{Wallet} developer's machine through social engineering, obtained AWS session tokens, and used them to modify a JavaScript bundle in Safe{Wallet}'s AWS S3 bucket. The injected code targeted only Bybit's cold wallet address. When Bybit's signers used the Safe web interface to approve what appeared to be a routine transfer, the malicious JavaScript silently replaced the transaction payload with a delegatecall to an attacker-controlled contract. That contract upgraded the Safe's implementation to attacker-controlled bytecode, giving the attacker control. 401,347 ETH was then drained in a single transaction."
  },
  {
    "question": "What is RPC poisoning and how was it used in the Kelp DAO hack?",
    "answer": "RPC (Remote Procedure Call) poisoning refers to compromising the nodes that blockchain applications query for on-chain state data, causing them to return manipulated or fabricated responses. In the Kelp DAO hack, attackers obtained access to LayerZero's internal RPC infrastructure, patched the running processes in memory to serve false data to the Decentralized Verifier Network's signing requests, and launched distributed denial-of-service attacks against external RPC providers to eliminate contradicting data sources. The verifier then attested to a fabricated rsETH burn event that never occurred, causing the bridge contract to release 116,500 rsETH to the attacker."
  },
  {
    "question": "Why didn't hardware wallets prevent these attacks?",
    "answer": "Hardware wallets verify cryptographic signatures: they confirm that you, the key holder, authorized a transaction. They cannot verify that the transaction you intended to sign is the same as the transaction constructed by the software layer above them. In both the Radiant Capital and Bybit attacks, malware operated upstream of the hardware wallet and swapped the payload before it reached the device. The hardware wallet then correctly signed the malicious transaction. Improving on-device readability of complex encoded contract calls remains an unsolved problem."
  },
  {
    "question": "Are smart contract audits sufficient to protect against these attacks?",
    "answer": "No. Smart contract audits examine bytecode and contract logic for vulnerabilities within the on-chain execution layer. The attacks in this article operated in off-chain infrastructure: malware on developer machines, injected JavaScript in frontends served through a content delivery network, and compromised RPC nodes. A standard smart contract audit would miss these attack vectors. Protecting against off-chain attacks requires endpoint security, supply chain integrity verification, operational security programs, and infrastructure hardening that is outside the scope of traditional audits."
  },
  {
    "question": "What is INLETDRIFT?",
    "answer": "INLETDRIFT is a macOS-targeting backdoor malware used in the Radiant Capital hack of October 2024. It was delivered inside a ZIP file sent via Telegram, disguised as a PDF security analysis document. Once deployed, it established persistence through a malicious AppleScript communicating with a command-and-control domain, and performed a man-in-the-middle attack on the signing interface, displaying legitimate transaction data while silently forwarding malicious payloads to connected hardware wallets. The malware was attributed to UNC4736 (Citrine Sleet), a Lazarus Group subcluster, by Mandiant."
  },
  {
    "question": "What is the common thread across all three Lazarus Group attacks?",
    "answer": "All three attacks exploited the gap between what a human or automated verifier sees and what is actually being signed or attested. In Radiant Capital, malware showed legitimate data on screen while sending a malicious payload to hardware wallets. In Bybit, injected JavaScript showed a routine transfer on screen while constructing a contract upgrade payload. In Kelp DAO, compromised RPC nodes showed a fabricated burn event to a bridge verifier. The contracts and cryptography behaved as designed, while the upstream information layer was compromised."
  }
]
