# How Web3's Weakest Link Became Its Most Expensive One

*Three case studies. $1.8 billion gone. Zero smart contract bugs exploited.*

The narrative that Web3 security is fundamentally a smart contract problem is dead. Audit culture built an entire industry around Solidity bugs, reentrancy vectors, and integer overflows, and it worked. DeFi-specific exploits dropped 89% year-over-year in Q1 2026. But losses didn't drop. They shifted.

The threat moved up the stack. Not into the contracts, but into the off-chain infrastructure surrounding them: the frontend serving layer, the developer workstations signing transactions, the RPC nodes feeding state data to bridge verifiers. The smart contracts held. Everything else didn't.

This piece examines three incidents, Radiant Capital (October 2024, $53M), Bybit (February 2025, $1.5B), and Kelp DAO (April 2026, $292M), as a technical sequence. Together, they trace the anatomy of a new class of attack that bypasses cryptographic security entirely by corrupting the information layer sitting in front of it. All three are attributed, with high confidence, to North Korea's Lazarus Group.

## Table of Contents

1. [Attack Vector 1: Malware-Mediated Hardware Wallet Bypass, Radiant Capital](#attack-vector-1-malware-mediated-hardware-wallet-bypass--radiant-capital)
2. [Attack Vector 2: Supply Chain JS Injection at CDN Layer, Bybit](#attack-vector-2-supply-chain-js-injection-at-cdn-layer--bybit)
3. [Attack Vector 3: RPC Node Poisoning with DDoS Failover Forcing, Kelp DAO](#attack-vector-3-rpc-node-poisoning-with-ddos-failover-forcing--kelp-dao)
4. [The Pattern](#the-pattern)
5. [What Doesn't Work (and Why)](#what-doesnt-work-and-why)
6. [What the Attack Surface Actually Looks Like](#what-the-attack-surface-actually-looks-like)
7. [Practical Implications](#practical-implications)
8. [Conclusion](#conclusion)
9. [About Us](#about-us)
10. [FAQ](#faq)



## Attack Vector 1: Malware-Mediated Hardware Wallet Bypass - Radiant Capital

**A 3-of-11 multisig designed for defense-in-depth became an attack surface with eleven entry points.** Radiant Capital operated this configuration to govern protocol parameter changes and contract upgrades. What looked like redundancy was actually exposure: eleven potential signers meant an attacker had eleven chances before needing to succeed on any three of them.

### Delivery: INLETDRIFT via Telegram Social Engineering

On September 11, 2024, 35 days before the exploit, a Radiant developer received a Telegram message from a threat actor impersonating a former contractor. The message requested peer review on a "Penpie Hack Analysis," a topic plausibly relevant to a DeFi security team. The ZIP file contained a decoy PDF rendering a legitimate-looking security analysis alongside `INLETDRIFT`, a macOS-targeting backdoor that established persistence via a malicious AppleScript communicating with `atokyonews[.]com`.

The developer opened the file. It looked exactly right. The malware deployed silently, no unusual permission prompts, no AV flagging, because `INLETDRIFT` was specifically crafted to avoid detection. When the file was forwarded internally for team feedback, the malware spread to additional developer machines. The attacker now had persistent access to multiple signing devices.

### Execution: MITM on the Signing Interface

Radiant's team followed textbook signing procedure: simulating every transaction in Tenderly before signing, verifying payload data at each step, using a mix of operating systems and hardware wallet models across signers, and cross-checking the Safe{Wallet} UI before each approval. None of it mattered.

`INLETDRIFT` performed a man-in-the-middle attack at the OS layer, sitting between the wallet interface and the hardware device. The Safe{Wallet} frontend rendered legitimate transaction data. What was actually sent to the hardware wallets for signature was different. The malware intercepted the signing request, replaced the payload, and forwarded the signed malicious transaction back to the interface. The result was three valid hardware wallet signatures on a transaction that upgraded Radiant's Pool Provider contract to an attacker-controlled implementation.

```
[Radiant Signer Device]
        |
        | UI renders: "Routine parameter update, destination: 0xabc..."
        |
[INLETDRIFT MITM Layer] ← payload swap happens here
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

The attack bypassed hardware wallets entirely. **The private keys were never extracted.** The signing devices performed exactly as designed. The malware compromised the data layer upstream of them.

### Losses and Attribution

$53M was drained from BSC and Arbitrum lending pools. Mandiant's post-mortem formally attributed the campaign to `UNC4736` (Citrine Sleet), a Lazarus Group subcluster, in a report published December 2024.



## Attack Vector 2: Supply Chain JS Injection at CDN Layer - Bybit

**The security model assumed the web UI accurately represented transaction intent. That assumption was the vulnerability.** Bybit used Safe{Wallet} (formerly Gnosis Safe) to manage an Ethereum cold wallet through a standard multisig flow: a proposer initiates a transaction via the `app.safe.global` web UI, the transaction is queued, co-signers review and approve through the same UI, and once threshold is met, execution proceeds.

### Delivery: Compromised Developer Machine → S3 Bucket Access

The attack began in early February 2025 when a Safe{Wallet} developer's machine was compromised, almost certainly via social engineering consistent with TraderTraitor TTPs. The attacker obtained the developer's AWS session tokens.

With those credentials, they accessed Safe{Wallet}'s AWS S3 bucket serving the Next.js frontend application for `app.safe.global`. On February 19, 2025, they modified the JavaScript bundle at `_next/static/chunks/pages/_app-52c9031bfa03da47.js`. The injected code was surgical: it did not alter Safe{Wallet}'s behavior for general users. It contained conditional logic that activated only when the transaction initiator matched Bybit's specific cold wallet address. For all other users: normal application behavior. For Bybit: a different payload was silently constructed before being sent to signers' hardware devices.

### Execution: delegatecall to Malicious Implementation Contract

When Bybit's authorized signers accessed the Safe UI on February 21, 2025, they saw a routine internal transfer. Destination address: expected. Amount: expected. Contract interaction: appeared normal. What was actually queued for signature was a `delegatecall` to an attacker-deployed contract. That contract's logic, when executed in the context of Bybit's Safe, called `upgradeTo()` on the proxy, replacing the canonical Safe implementation with attacker-controlled bytecode.

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

Once the implementation was swapped, the attacker had unconditional control over the Safe's execution logic. All assets, 401,347 ETH, were drained in a single subsequent transaction. Approximately two minutes post-execution, the attacker removed the malicious JavaScript from the S3 bucket to destroy forensic evidence. The Wayback Machine captured both the clean and modified versions.

### Why Hardware Wallets Didn't Help

The Ledger and Trezor devices in Bybit's signing workflow display the actual calldata being signed. In theory, a signer reviewing the `delegatecall` to an unknown implementation contract should have triggered rejection. In practice, the small-screen representation of complex multisig transaction data is not human-readable. An `execTransaction` call with nested ABI encoding does not surface as "you are upgrading this contract to attacker code." It surfaces as hex data that no signer is expected to manually decode.

**The malicious JavaScript exploited precisely this gap**: the UI layer showed what humans could verify, while the actual signing payload contained what could drain the wallet.

### Losses and Attribution

$1.5B, the largest single theft in cryptocurrency history. The FBI formally attributed the attack to TraderTraitor on February 26, 2025. Assets were laundered through a combination of mixers, cross-chain bridges, and OTC brokers at a pace that outran most freeze attempts.



## Attack Vector 3: RPC Node Poisoning with DDoS Failover Forcing - Kelp DAO

**This attack contained no smart contract exploit, no cryptographic break, and no stolen private keys. Every on-chain transaction was valid. The fraud was entirely in the off-chain layer.** Kelp DAO's rsETH token existed across multiple chains via a LayerZero OFT bridge. The bridge's security model relied on a Decentralized Verifier Network (DVN): when tokens are burned on the source chain, a DVN reads the event and attests to it, triggering minting on the destination chain. Kelp operated a 1-of-1 DVN configuration, one verifier, one point of failure.

### Delivery: Social Engineering → Session Key Theft → RPC Infrastructure Access

The attack chain began on March 6, 2026, when the attacker used social engineering against a LayerZero Labs developer to obtain session credentials. With those credentials, they accessed LayerZero's internal cloud infrastructure and located the RPC nodes used by the LayerZero Labs DVN to read source-chain state. The attacker identified all RPC endpoints in the DVN's configuration, a mix of internal nodes and external providers, and gained access to two internal nodes.

The compromise was not a simple misconfiguration grab. The attacker patched the running RPC processes in memory, such that LayerZero tools would receive responses as usual even as manipulated data was served to the DVN's signing requests. The patched nodes returned normal responses to health checks and general queries while serving crafted data to DVN-specific calls.

### Execution: DDoS + Forged Cross-Chain Message

The two compromised internal nodes alone were insufficient, the DVN queried additional external RPC providers that would contradict the attacker's fabricated state. So the attacker launched a DDoS attack against the external providers, knocking them offline and forcing the DVN's RPC resolution to fail over exclusively to the compromised internal nodes.

With the DVN's entire view of source-chain state now under attacker control, the attacker injected a fabricated cross-chain message: a phantom rsETH burn event that had never occurred on the source chain. The DVN validated the fake event against the poisoned RPC data. The attestation was cryptographically valid, the keys weren't stolen, the verifier logic wasn't bypassed, and the LayerZero protocol itself wasn't exploited. The inputs it received were simply false.

```
[Attacker's forge]
"116,500 rsETH burned on source chain at block X, tx: 0xfake..."

         ↓ injected into DVN's RPC data feed

[LayerZero Labs DVN]
  Queries RPC nodes for burn event at block X
  Internal node 1 (compromised): "Confirmed."
  Internal node 2 (compromised): "Confirmed."
  External provider 1 (DDoS'd): timeout
  External provider 2 (DDoS'd): timeout

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

These three attacks, separated by 18 months across two different protocol types, are technically distinct but strategically identical.

| | Radiant Capital | Bybit | Kelp DAO |
|---|---|---|---|
| **Year** | 2024 | 2025 | 2026 |
| **Loss** | $53M | $1.5B | $292M |
| **Entry Point** | Developer workstation | Third-party dev workstation | Developer session keys |
| **Mechanism** | Malware MITM on signing | JS injection at CDN layer | RPC memory patching + DDoS |
| **What was bypassed** | Hardware wallet display | Hardware wallet display | Bridge verifier consensus |
| **Smart contract bug?** | No | No | No |
| **Attribution** | Lazarus (UNC4736) | Lazarus (TraderTraitor) | Lazarus (TraderTraitor) |

The common thread is not a vulnerability class. It is an architectural assumption: that the off-chain software layer accurately represents on-chain state and transaction intent to the humans and automated systems responsible for approving actions. Across all three attacks, that assumption was false. The contracts held. The keys held. The cryptography held. What broke was the information layer sitting upstream of all of it.



## What Doesn't Work (and Why)

**Hardware wallets** are not a sufficient defense when the payload being sent to the device is constructed by compromised software. Hardware wallets verify signatures, not intent. If the UI renders "transfer 100 ETH" and the actual ABI-encoded calldata says "upgrade implementation to 0xattacker," the hardware wallet sees calldata, not the English summary. Improving on-device transaction readability is an open and unsolved problem for the wallet ecosystem.

**Multi-sig thresholds** are not a sufficient defense when malware has persistent access to multiple signing machines in the same organization. A 3-of-11 scheme with eleven collocated developers represents eleven attempts, not eleven independent airgaps. The threshold only matters if compromising any single signer is genuinely isolated from compromising the others.

**Transaction simulation** (Tenderly et al.) is not a sufficient defense when the malware intercepts the payload after simulation and before signing. Tenderly can only simulate what it receives. In the Radiant case, simulation passed because the simulation saw a legitimate transaction, the payload swap happened downstream of the simulation step.

**Standard smart contract audits** cover none of this. An audit of Radiant's contracts would not have found `INLETDRIFT`. An audit of Bybit's Safe deployment would not have found the compromised S3 bucket. An audit of Kelp's bridge contract would not have found the social engineering that compromised the LayerZero developer. Audits address the bytecode boundary. These attacks operated above it.



## What the Attack Surface Actually Looks Like

**The Web3 security perimeter is no longer the bytecode boundary.** The 2026 Sherlock quarterly report was unambiguous: infrastructure attacks, private key compromise, cloud key management failures, bridge validator compromise, represented 76% of all classified incidents by count in Q1 2026. Smart contract exploits were 89% lower than the same period in 2025. The contracts got harder. Everything around them stayed the same.

The actual attack surface now extends to developer workstations and their OpSec hygiene, third-party frontend dependencies and their CDN infrastructure, CI/CD pipelines with access to deployment credentials, RPC providers and their redundancy and compromise detection, session key management for cloud infrastructure, and the human layer, every developer with signing authority is a potential entry point for a patient, well-resourced adversary. Lazarus Group has demonstrated it is willing to invest months of preparation for a single operation. The March-to-April timeline of the Kelp DAO attack, six weeks between initial access and execution, is representative, not exceptional.



## Practical Implications

**For protocols managing multisig governance**, signing devices must be network-isolated during the signing session, not merely "hardware wallets" that remain connected to a compromised OS. Signers should verify transaction calldata independently from the primary UI, using a separate RPC endpoint and a separate device to decode what they are actually signing. Threshold-to-signer ratios matter in a specific way: 3-of-11 gives an attacker eight misses before they need to succeed; 3-of-5 with truly independent airgaps, different machines, different networks, different physical locations, is meaningfully stronger in practice.

**For bridge and cross-chain infrastructure**, DVN redundancy is not optional for protocols above a meaningful TVL threshold. Single-verifier configurations must be treated as critical vulnerabilities regardless of what a provider's default configuration or quickstart guide suggests. RPC node independence must be real independence: different providers, different geographic regions, different infrastructure operators, with active anomaly detection on response divergence. DDoS resilience for critical infrastructure RPC endpoints must be built into the threat model before deployment, not retrofitted after an incident.

**For all protocol teams**, the developer machine is now in scope. OpSec hygiene, phishing resistance, verified file sources, sandboxed execution for received files, hardware security keys for cloud access, is protocol security. Frontend integrity verification through Subresource Integrity headers, reproducible builds, and signed release artifacts is not optional when the frontend mediates billion-dollar transactions. Session key and cloud credential hygiene must be treated with the same rigor as private key management, because as the Kelp DAO attack demonstrated, they are operationally equivalent attack surfaces.



## Conclusion

The smart contracts held. That is the good news. The bad news is that the attack surface is now everything a developer touches between writing the contract and the moment assets move. Lazarus Group has spent three years systematically mapping that surface, moving from endpoint malware to supply chain injection to RPC infrastructure compromise, with a level of patience and technical sophistication that the industry's current security practices are not designed to match.

The $1.8 billion total across these three incidents is not the cost of smart contract bugs. It is the cost of treating off-chain infrastructure as someone else's security problem. As audit coverage of on-chain code improves and smart contract exploits become rarer, this category of attack will represent an increasing share of total losses, until the industry builds the same rigor around its operational security that it has built around its bytecode.



## About Us

At SC Audit Studio, we specialize in protocol security assessments. Our team of experts has worked with companies like Aave, 1inch, and many more to conduct thorough security assessments across EVM and non-EVM environments.

[Reach out to us](https://scauditstudio.com/contact) for queries and security assessments!



## Tags

["Web3", "OpSec", "Security", "Lazarus Group", "Bybit", "Radiant Capital", "Kelp DAO", "RPC Poisoning", "Private Key Compromise", "DeFi", "Supply Chain Attack", "INLETDRIFT"]



## FAQ

[
  {
    "question": "What is an off-chain attack in Web3?",
    "answer": "An off-chain attack targets the infrastructure surrounding a blockchain protocol rather than the smart contracts themselves. This includes developer workstations, frontend applications, CDN infrastructure, RPC nodes, cloud credentials, and any software that mediates between a human approver and the on-chain transaction. In the three cases examined here, the smart contracts were never exploited, the attacks operated entirely in the off-chain layer that produced, displayed, or verified transaction data."
  },
  {
    "question": "How did the Bybit hack work technically?",
    "answer": "Attackers compromised a Safe{Wallet} developer's machine via social engineering, obtained AWS session tokens, and used them to modify a JavaScript bundle in Safe{Wallet}'s AWS S3 bucket. The injected code contained conditional logic targeting only Bybit's cold wallet address. When Bybit's signers used the Safe UI to approve what appeared to be a routine transfer, the malicious JavaScript silently replaced the transaction payload with a delegatecall to an attacker-controlled contract. That contract upgraded the Safe's implementation to attacker-controlled bytecode, giving the attacker full control. 401,347 ETH was then drained in a single transaction."
  },
  {
    "question": "What is RPC poisoning and how was it used in the Kelp DAO hack?",
    "answer": "RPC (Remote Procedure Call) poisoning refers to compromising the nodes that blockchain applications query for on-chain state data, causing them to return manipulated or fabricated responses. In the Kelp DAO hack, attackers obtained access to LayerZero's internal RPC infrastructure, patched the running processes in memory to serve false data to the DVN's signing requests, and simultaneously DDoS'd external RPC providers to eliminate contradicting data sources. The DVN then attested to a fabricated rsETH burn event that never occurred, causing the bridge contract to release 116,500 rsETH to the attacker."
  },
  {
    "question": "Why didn't hardware wallets prevent these attacks?",
    "answer": "Hardware wallets verify cryptographic signatures, they confirm that you, the key holder, authorized a transaction. They cannot verify that the transaction you intended to sign is the same as the transaction that was constructed by the software layer above them. In both the Radiant Capital and Bybit attacks, the malware operated upstream of the hardware wallet, swapping the payload before it reached the device. The hardware wallet then correctly signed the malicious transaction. Improving on-device readability of complex ABI-encoded calldata is an unsolved problem."
  },
  {
    "question": "Are smart contract audits sufficient to protect against these attacks?",
    "answer": "No. Smart contract audits examine bytecode and contract logic for vulnerabilities within the on-chain execution layer. The attacks in this article operated entirely in the off-chain infrastructure: malware on developer machines, injected JavaScript in CDN-served frontends, and compromised RPC nodes. None of these attack vectors would be detected by a standard smart contract audit. Protecting against off-chain attacks requires endpoint security, supply chain integrity verification, operational security programs, and infrastructure hardening that is outside the scope of traditional audits."
  },
  {
    "question": "What is INLETDRIFT?",
    "answer": "INLETDRIFT is a macOS-targeting backdoor malware used in the Radiant Capital hack of October 2024. It was delivered inside a ZIP file sent via Telegram, disguised as a PDF security analysis document. Once deployed, it established persistence through a malicious AppleScript communicating with a command-and-control domain, and performed a man-in-the-middle attack on the signing interface, displaying legitimate transaction data while silently forwarding malicious payloads to connected hardware wallets. The malware was attributed to UNC4736 (Citrine Sleet), a Lazarus Group subcluster, by Mandiant."
  },
  {
    "question": "What is the common thread across all three Lazarus Group attacks?",
    "answer": "All three attacks exploited the gap between what a human or automated verifier sees and what is actually being signed or attested. In Radiant Capital, malware showed legitimate data on screen while sending a malicious payload to hardware wallets. In Bybit, injected JavaScript showed a routine transfer on screen while constructing a contract upgrade payload. In Kelp DAO, compromised RPC nodes showed a fabricated burn event to a bridge verifier. The cryptography held in every case. The attack surface was the information layer sitting upstream of it."
  }
]