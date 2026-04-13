# A developer's guide from transition of EVM to Canton

## Table of Contents

1. [The Mental Model Mismatch](#the-mental-model-mismatch)
2. [What Is Canton?](#what-is-canton)
3. [The Architecture: Why There Is No "Global EVM"](#the-architecture-why-there-is-no-global-evm)
4. [Validators, Sequencers, and Super Validators](#validators-sequencers-and-super-validators)
5. [Privacy by Architecture, Not by Workaround](#privacy-by-architecture-not-by-workaround)
6. [How Wallets and Parties Work](#how-wallets-and-parties-work)
7. [Smart Contracts and Tokens: Daml vs. Solidity](#smart-contracts-and-tokens-daml-vs-solidity)
8. [Atomic Composability Across Subnetworks](#atomic-composability-across-subnetworks)
9. [Why Institutions Are Paying Attention](#why-institutions-are-paying-attention)
10. [What This Means for EVM Developers](#what-this-means-for-evm-developers)
11. [Conclusion](#conclusion)
12. [About Us](#about-us)
13. [FAQ](#faq)



If you have spent any real time in the EVM ecosystem, you have developed a mental model for how blockchains work: there is one shared ledger, every node replays every transaction, smart contracts live at addresses anyone can call, and privacy is an afterthought solved by ZK proofs bolted on later. That mental model is exactly what gets EVM developers into trouble when they first encounter Canton. The concepts look familiar on the surface, validators, tokens, smart contracts, wallets, but each of them means something subtly and importantly different. This article is a deliberate translation guide: every key Canton concept explained by first establishing what you already know from Ethereum, and then precisely describing where Canton diverges.

## The Mental Model Mismatch

**Canton's base layer is not a blockchain you deploy contracts to and interact with directly**. When EVM developers hear "blockchain," they picture something like Ethereum mainnet, a globally shared execution environment where anyone can send a transaction, call a contract, and have that execution permanently recorded in a world-readable state trie. Canton's Global Synchronizer is nothing like this.

Think of the Global Synchronizer as more analogous to a cryptographic coordination bus than a programmable ledger. It does not store your contract's state. It does not execute your business logic. It exists to sequence messages and guarantee atomicity across a distributed set of participant nodes that each hold their own private slice of the ledger. The business logic and contract state live inside those participant nodes and inside the **subnetworks** (called Synchronization Domains) that sit above the base layer. If you try to interact with Canton the way you interact with Ethereum, looking for a global address space and a shared EVM to send calls to, you will not find it, because it deliberately does not exist.

This distinction is not a bug or a limitation. It is the core architectural decision that gives Canton its unique properties: genuine data privacy without ZK cryptography overhead, atomic composability across isolated domains, and the ability to satisfy the data residency requirements of regulated institutions.

## What Is Canton?

Canton is a distributed ledger protocol developed by Digital Asset, designed to be a **network of networks**. Rather than one monolithic chain where all participants share all state, Canton is a protocol for creating interconnected subnetworks,called Synchronization Domains, each of which is an isolated ledger governed by its own rules, its own set of participants, and potentially its own regulatory regime. These domains can be permissioned or permissionless, private or semi-public, and they can each run different applications. The Global Synchronizer is the top-level domain that provides a canonical sequencing layer and coordination point across all subdomains.

To use an analogy that EVM developers will recognize: if Ethereum mainnet is like one enormous multi-tenant shared computer, Canton is like a network of isolated computers that can pass messages between each other and, crucially, execute multi-leg transactions atomically across machines. Imagine being able to call a function on Arbitrum and a function on Optimism in a single atomic transaction where either both succeed or both fail, with no bridge, no trusted relayer, and no cross-chain messaging delay. That is what Canton's atomic composability provides, except the "chains" in Canton are its Synchronization Domains.

Digital Asset built Canton on top of **Daml** (Digital Asset Modeling Language), a functional smart contract language designed from the ground up for multi-party workflows in regulated environments. While Solidity is an imperative language that describes state transitions on a global machine, Daml is a contract-oriented language that describes rights and obligations between specific named parties. Every Daml contract explicitly enumerates who the signatories are, who the observers are, and what choices each party has. This rights-based model is not just a language design choice, it is the foundation of Canton's privacy model, which we will explore in depth below.

The Canton Network (as distinct from the Canton protocol) is the production deployment built on this architecture. It includes the Global Synchronizer operated by a consortium of super validators, a growing set of application-specific Synchronization Domains, and a native token, Canton Coin (CNT), used to pay for sequencing and governance participation. Major financial institutions including Goldman Sachs, BNP Paribas, Deutsche Börse, and others have been involved in building and running infrastructure on the network, which is itself a significant signal about where institutional blockchain interest is moving.

## Why There Is No "Global EVM"

On Ethereum, you sign a transaction, broadcast it to the mempool, and a validator orders it into a block. Every full node on the network downloads that block and re-executes your transaction independently to verify it. The result is that all state changes, every storage slot you modified, every event you emitted, are globally visible to every participant on the network, forever. The price of this global consistency and openness is that you cannot have privacy without additional cryptographic machinery.

On Canton, the flow is fundamentally different. When a participant (let's call it a bank running a Canton node) wants to execute a transaction, their node first **locally validates** the transaction using the Daml interpreter. The transaction is then submitted not as raw calldata to a global mempool, but as an encrypted **transaction view tree** to a Synchronization Domain's **sequencer**. The sequencer assigns a global order to the transaction (providing the canonical sequencing needed for consistency), but it does so on an encrypted payload. The sequencer never sees the content of the transaction, only its cryptographic commitments. A **mediator** within the domain then checks that all required parties have confirmed their approvals (also via encrypted confirmations), and only upon unanimous confirmation does the transaction finalize.

**the state of a Daml contract is never stored on a shared global ledger**. It is stored locally at each participant node that is a party to that contract. The Global Synchronizer and the Synchronization Domains only ever see and store cryptographic commitments, hashes that prove a piece of state exists and is consistent without revealing what it is. This is not ZK-proofs-on-top-of-a-public-chain. The data simply never goes to the shared layer in the first place.

In a single Canton transaction that involves multiple contracts and multiple parties, different parties see different parts of the transaction tree. A party sees only the sub-transactions that involve contracts they are a party to. Everyone else sees an encrypted blob. This is privacy by design at the protocol level, not a layer-2 workaround.

For EVM developers used to thinking in terms of `msg.sender` and public storage variables, this is a significant shift. There is no global `eth_call` you can make against a Canton contract to read its state, because the state does not live on a shared server. You read state through your own participant node, which holds the contracts you are a party to.

## Validators, Sequencers, and Super Validators

In Ethereum, the term "validator" refers to a node that participates in consensus, it proposes and attests to blocks, earns rewards, and can be slashed. The validator set is global; every validator sees every block and every transaction. Canton uses a very different set of roles, and conflating them with Ethereum validators is one of the most common sources of confusion.

**Participant Nodes** are the closest Canton equivalent to "users" in the EVM sense, but they are not validators in the consensus sense. Each institution, business, or end-user that wants to interact with Canton contracts runs (or delegates to) a participant node. The participant node holds that party's private contract state, runs the Daml interpreter locally, constructs transaction view trees, and communicates with Synchronization Domains on the party's behalf. Think of a participant node as a combination of a wallet, a local Ethereum node, and a contract execution engine, except the contracts it runs are private to its party set.

**Sequencers** are the infrastructure layer of a Synchronization Domain. Their job is to receive encrypted transaction submissions from participant nodes and assign them a canonical global order. Sequencers are analogous to the ordering layer in an L2 rollup: they determine the sequence of events without necessarily executing business logic themselves. A Synchronization Domain can have multiple sequencers for redundancy and fault tolerance.

**Mediators** sit alongside sequencers within a Synchronization Domain and handle the confirmation phase of Canton's two-phase commit protocol. When a transaction requires multiple parties to approve, the mediator coordinates collecting those approvals (in encrypted form) and either finalizing the transaction or rejecting it if approvals are not received within the timeout window. The mediator provides the safety guarantee that a Canton transaction is all-or-nothing: either every party's state change commits, or none of them do.

**Super Validators** are the most important role to understand if you are trying to grasp Canton's governance structure. The Global Synchronizer, the top-level coordination layer of the Canton Network, is operated by a consortium of super validators. These are vetted institutions (exchanges, banks, infrastructure providers) that have staked Canton Coin and been approved through the governance process to run the sequencer and mediator infrastructure for the Global Synchronizer itself. Super validators have governance rights over the Global Synchronizer's parameters, can vote on protocol upgrades, and earn sequencing fees in exchange for providing the foundational ordering service. Think of super validators as an institutional version of Ethereum's proposer-builder separation, combined with a DAO governance role, except membership is permissioned and identity-verified.

when you read that Canton has "validators," you need to ask which layer they operate at. A participant node validates transactions for its own party's contracts. A sequencer-mediator pair validates the ordering and approval process for a Synchronization Domain. A super validator operates the Global Synchronizer and has network-wide governance rights. These are distinct roles that do not cleanly map to a single Ethereum validator concept.

## Privacy by Architecture, Not by Workaround

Privacy in the EVM ecosystem is, to put it charitably, retrofitted. The canonical approach involves ZK-SNARKs or ZK-STARKs, computationally intensive cryptographic proofs that allow you to verify that a computation happened correctly without revealing the inputs. Projects like Aztec, Midnight, and various ZK-rollup privacy extensions all follow this pattern: compute off-chain, prove on-chain, keep the actual data hidden. It works, but it comes with significant tradeoffs: proving time, proof size, the complexity of writing circuits, and the constant tension between what you can prove efficiently and what your business logic actually requires.

Canton's approach is architecturally different. Privacy is not achieved by cryptographic proof of hidden computation, it is achieved by **never broadcasting the data in the first place**. Because Canton's ledger model distributes contract state to only the parties of each contract (via their participant nodes), sensitive transaction data is simply never transmitted to parties who do not need to see it. There is no global state trie that a competitor can scan. There is no event log that leaks counterparty information. The only shared artifacts are cryptographic commitments on the Synchronization Domain, hashes that prove the state transitions happened consistently but reveal nothing about the underlying business data.

This model maps well to how financial institutions actually think about data. When Goldman Sachs and BNP Paribas execute a bilateral repo transaction, they do not want Deutsche Börse or any other participant in the network to see the economic terms of that trade. In an EVM environment, even with ZK proofs, you are trusting that the circuit is correct, that the proving system is sound, and that the proof does not leak information through side channels. In Canton, the other participants literally never receive the data. Their nodes never processed it.

The sub-transaction privacy model extends this even further. Consider a Canton transaction that involves three parties: a bank, a client, and a regulator. The regulator might be an observer on the top-level contract, they see that a transaction occurred and its category, but not the detailed sub-transactions between the bank and the client. The client sees their specific leg of the transaction but not the bank's internal risk management sub-contracts that were also updated as part of the same atomic operation. This granular, per-party, per-subtransaction visibility is baked into the Daml model from the ground up. You define who sees what at the contract definition level, not by applying patches at the infrastructure level after the fact.

For developers coming from Solidity, this requires a genuine shift in how you think about authorization and visibility. In Solidity, you think about `public`, `private`, and `internal` function visibility, and about `require` statements that gate access. In Daml, you think about **signatories** (parties whose authority is required to create or archive a contract), **observers** (parties who can see the contract but cannot take actions on it), and **controllers** (parties who can exercise specific choices). A Daml contract is essentially a legally-inspired description of rights and obligations, and its privacy properties flow directly from that rights model.

## How Wallets and Parties Work

The concept of a "wallet" in the EVM world is straightforward: a private key generates a public key and an Ethereum address. The address is a first-class citizen of the EVM state trie, it can hold ETH, own ERC-20 balances stored in contract mappings, and authenticate transactions via ECDSA signatures. Wallets in the self-custodial sense (MetaMask, Ledger) are simply key management interfaces. The blockchain itself is the source of truth for balances.

Canton takes a layered approach that separates identity from infrastructure in a way that EVM developers initially find unfamiliar. The fundamental identity primitive in Canton is a **Party**. A party is a logical entity, "Acme Bank" or "Alice (end-user)", that is identified by a cryptographic key pair and a party identifier string. Parties are registered on the Global Synchronizer or on a specific Synchronization Domain, establishing their identity in the network.

However, a party does not directly talk to the network. A party operates through a **Participant Node**. The participant node is the infrastructure layer: it maintains the party's private ledger state (all the Daml contracts they are a signatory or observer of), signs transactions on behalf of the party, and communicates with Synchronization Domains. One participant node can host multiple parties (a bank running a node might host thousands of customer parties), and a single party can theoretically be hosted on multiple participant nodes for redundancy.

This distinction matters enormously for how "wallets" work in Canton. There is no equivalent of MetaMask for Canton, where a browser extension holds your private key and constructs raw transactions to submit to an RPC endpoint. Instead, Canton wallets are typically **applications integrated with a participant node's API**. The wallet application talks to the participant node (via Canton's JSON API or Ledger API), queries the participant's local ledger for your party's contract state, and submits commands that the participant node translates into Canton transactions.

The Canton Network has developed a **wallet application** for Canton Coin (CNT) that is the end-user-facing interface for managing the native token. But this wallet is not a universal address-based identity like an Ethereum wallet, it is tied to a specific participant node infrastructure. Self-custody in the traditional EVM sense, controlling the raw private keys and submitting transactions directly to the network without any intermediary node infrastructure, is not the default interaction model for Canton. The participant node is a necessary piece of infrastructure, which is part of why the network is designed around institutional operators who run participant nodes on behalf of their clients.

For EVM developers building applications, this means the user experience abstraction layer is very different. Your dApp does not ask users to connect MetaMask and read their address. Instead, it interacts with a participant node API that manages the user's party and their contracts. The authentication model is also different: instead of signing a transaction hash with your wallet's private key, you issue ledger commands authenticated at the participant node level, with the participant's key acting as the network-level authentication while your application enforces user-level authentication through its own mechanisms.

## Smart Contracts and Tokens: Daml vs. Solidity

The smartest thing to internalize here is that Daml contracts are not Ethereum smart contracts. They do not compile to bytecode that runs on an EVM. They do not live at a globally-accessible address. They do not have storage slots that the whole world can read. Understanding Daml requires stepping back from the EVM execution model entirely.

A Daml contract is a **typed record with explicit parties and a set of choices**. Consider a simple token transfer in both languages. In Solidity, you have an ERC-20 contract deployed at a single address, a `balances` mapping that stores everyone's balance in one big table, and a `transfer` function that anyone (with sufficient balance) can call. The entire state of the token system is in that one contract's storage, globally readable by anyone with an RPC node.

In Daml, a fungible token is represented as a collection of **contract instances**, each of which is a separate asset held by a specific party. There is no global balances table. Instead, "Alice holds 100 tokens" is represented as a `Token` contract where Alice is the owner, a contract that exists in Alice's participant node's ledger. To transfer, Alice **exercises a choice** on that contract, which archives the original contract and creates a new one with Bob as the owner. The "transfer" is not a call to a shared function, it is the bilateral creation and archiving of typed contracts, and it requires Alice's signature (as the current owner/signatory) to happen. Bob sees the new contract appear in his node's ledger. No global contract address was involved; no balances mapping was mutated.

This is why Canton's privacy model works without ZK proofs: Alice and Bob's transaction is a private bilateral event. The Synchronization Domain gets a cryptographic commitment proving the old contract was properly archived and the new one properly created, but it never sees the economic terms, the parties, or the amounts. Only Alice's and Bob's participant nodes processed the full transaction content.

For tokens specifically, Canton has a standard called **Canton Coin (CNT)** for the network's native token, which follows this asset-as-contract model. Application-level tokens on Canton subnetworks are built in the same way: each unit of value (or each position, in the case of a financial instrument) is a typed Daml contract with explicit owners, not an entry in a shared ERC-20 mapping. This model is surprisingly natural for financial instruments, a bond, a repo agreement, a derivative, which are all fundamentally bilateral or multilateral contracts between named parties, not entries in a global table.

**Daml's choice system** is the equivalent of Solidity's function calls, but with a crucial difference: choices in Daml have a clearly defined **controller**, the party or parties whose authorization is required to exercise that choice. A choice controller is specified in the contract template, not enforced by `require(msg.sender == owner)` checks that developers have to remember to write correctly. The authorization model is declarative and enforced by the Daml runtime, which eliminates an entire category of access control bugs that are endemic in Solidity development. If a choice requires the counterparty's approval, the Daml runtime simply will not allow it to execute without a valid confirmation from that party's node, at the protocol level.

Digital Asset has invested heavily in an **EVM compatibility layer** for Canton, allowing Solidity contracts to be deployed into a Canton-compatible execution environment. This layer, sometimes called the Canton EVM, runs an Ethereum-compatible execution environment (based on Hyperledger Besu) as a Synchronization Domain on top of Canton. This means you can write Solidity, deploy ERC-20 tokens, and interact with familiar tooling (Hardhat, Foundry, ethers.js), while Canton handles the synchronization and settlement layer beneath. The EVM layer trades some of Canton's native privacy guarantees for EVM compatibility, but it provides a migration path for EVM-native teams that is significantly lower friction than learning Daml from scratch.

## Atomic Composability Across Subnetworks

This is the property that Canton's architects are most proud of, and rightly so, because it solves a problem that the EVM ecosystem has been wrestling with since the launch of the first L2 rollup: how do you execute a multi-leg transaction atomically across isolated ledger environments?

In the EVM world, if you want to execute a trade that requires moving an asset on Ethereum mainnet and simultaneously releasing a payment on Arbitrum, you cannot do it atomically in a single transaction. You use a bridge, which requires trust assumptions on relayers and oracle infrastructure, introduces settlement delay, and creates a window during which one leg has executed but the other has not, a period of counterparty risk that financial institutions are deeply uncomfortable with. This is the fundamental problem of cross-chain atomicity, and it is currently unsolved in the EVM ecosystem without centralized intermediaries.

Canton's Synchronization Domain architecture provides atomic composability as a first-class protocol feature. Because every Synchronization Domain uses the same Canton protocol for its transaction lifecycle, the encrypted view tree submission, sequencer ordering, mediator confirmation, a Canton transaction can span multiple Synchronization Domains in a single atomic unit. This is achieved through **Canton's commitment and confirmation protocol**, which is essentially a cryptographically enforced two-phase commit across domains.

when a transaction touches contracts on multiple Synchronization Domains, the transaction view tree is partitioned by domain. Each relevant Synchronization Domain's sequencer and mediator runs its local confirmation phase. The participant nodes involved collect confirmations from all required parties across all relevant domains. Only when all domains' mediators have received unanimous confirmation does the transaction finalize. If any domain's confirmation phase fails or times out, the entire transaction rolls back across all domains. No bridges, no oracles, no relayers with trust assumptions, just the Canton protocol operating uniformly across domain boundaries.

The practical implication is enormous for financial use cases. A bank can execute a Delivery vs. Payment (DvP) settlement where a security token on one Synchronization Domain (say, a private securities network) is atomically exchanged for a cash token on another domain (say, a regulated payments network), with the entire multi-leg transaction being atomic from both the bank's and the counterparty's perspective. Neither leg settles unless both settle. This is exactly how traditional financial settlement is supposed to work, and it is a capability that the EVM ecosystem cannot currently provide without introducing centralized intermediaries.

For developers, this atomic composability means that cross-domain Canton applications can be designed with the same mental model as same-chain applications in Ethereum. You define a transaction that touches contracts on multiple domains, and the Canton runtime handles the cross-domain synchronization transparently. The application developer does not need to write bridge contracts, handle partial execution states, or design recovery mechanisms for failed cross-chain operations. The protocol guarantees all-or-nothing execution.

## Why Institutions Are Paying Attention

If you have been following the space for a while, you have seen waves of "institutional blockchain" announcements that ultimately failed to deliver production systems. Canton is attracting a different quality of attention, for specific technical reasons that become obvious once you understand the architecture.

The first reason is **data privacy as a regulatory baseline, not an optional feature**. Under GDPR in Europe, MiFID II in capital markets, and countless other regulatory frameworks, financial institutions cannot put client transaction data on a shared public ledger. This is not negotiable. The standard EVM approach of ZK proofs on public chains creates a compliance complexity that legal and compliance teams struggle to approve, because the proof system itself is a novel technology with its own trust assumptions. Canton's approach, the data simply never leaves the relevant participant nodes, maps cleanly to existing data governance frameworks. A Canton transaction between two counterparties is, from a data protection standpoint, a bilateral exchange between two systems, not a broadcast to a public ledger. Regulators understand bilateral systems.

The second reason is **atomic DvP and settlement finality**. The global securities industry processes trillions in settlement daily, with T+2 (two-day settlement) being the current standard in most markets. The driver of that delay is not processing speed, it is counterparty risk management, reconciliation, and the fact that payment and delivery happen in separate systems. Canton's atomic composability directly addresses the settlement cycle problem: if DvP can be truly atomic, the rationale for T+2 delays largely evaporates. The Canton Network's involvement with SIX Digital Exchange (SDX) in Switzerland and other major market infrastructure operators is specifically focused on this use case.

The third reason is **interoperability with existing infrastructure**. Institutions have decades of investment in existing systems, core banking systems, custodians, messaging infrastructure (SWIFT, FIX, ISO 20022). Canton's architecture, with its participant node model and API-driven integration layer, is designed to sit alongside existing systems rather than replace them. A bank can run a Canton participant node that talks to its existing custody system on one side and the Canton Network on the other, without requiring a wholesale replacement of internal infrastructure. The EVM compatibility layer extends this further: existing EVM tooling, existing Solidity developer skills, and existing DeFi protocols can be connected to the Canton Network without a full rewrite.

The fourth reason, and perhaps the most forward-looking, is **regulatory observability without transparency to competitors**. Regulators are extremely interested in the possibility of having real-time, verifiable oversight of financial transactions without requiring those transactions to be visible to market participants. Canton's observer model, where a regulatory party can be added as an observer to specific contract types, seeing transaction confirmations without seeing the economic terms, creates a architecture for regulatory access that is technically precise and auditable, rather than the current model of periodic batch reporting that is easy to game.

Canton has been present at major Ethereum ecosystem events, including side events at EthCC, precisely because they recognize that the EVM developer community represents the most sophisticated pool of blockchain developers in the world, and because their EVM compatibility layer creates a genuine on-ramp. As institutional blockchain interest shifts from permissioned chains that replicate legacy systems to networks with genuine cross-party composability and regulatory-grade privacy, Canton's technical differentiation becomes increasingly relevant to anyone building in the space.

## What This Means for EVM Developers

If you are an EVM developer evaluating Canton for the first time, here is a practical summary of the mental model shifts required.

Your contracts are not global objects. In Solidity, a deployed contract is a permanent fixture of the chain, anyone can call it, anyone can read its storage (even private variables via direct slot access), and its state is the same for every participant. In Canton, a Daml contract instance is a private object held by specific participant nodes. There is no universal address you can call; you interact with contracts through your participant node's API, which shows you only the contracts your party has access to.

Your identity is not just a key pair. In Ethereum, your identity is your address, 20 bytes derived from your public key, sufficient to send and receive funds and call contracts. In Canton, your identity is a **party** registered on a participant node. That participant node is the infrastructure layer that connects your party to the network. Self-custody and direct-to-network interaction are possible but require running your own participant node, which is a more substantial operational commitment than running a local Ethereum node.

Validators are not a monolithic role. The sequencers, mediators, participant nodes, and super validators in Canton are all distinct infrastructure roles with distinct responsibilities. When you think about trust assumptions, who can censor your transactions, who can see your data, who has governance rights, you need to understand which layer of the Canton stack you are reasoning about.

Cross-domain composability is not bridging. When Canton executes a transaction that spans multiple Synchronization Domains, it is not a bridge in the EVM sense. There are no lock-and-mint mechanics, no relayers with economic incentives to correctly relay messages, and no race conditions between chains. It is a single atomic transaction with the Canton protocol enforcing all-or-nothing execution at the protocol level.

The EVM compatibility layer gives you a genuine starting point. If the Daml learning curve feels steep, the Canton EVM layer lets you write Solidity and deploy familiar contracts while still settling on Canton infrastructure. You sacrifice some of the native privacy guarantees, because Solidity's global state model does not map to Canton's sub-transaction privacy model, but you gain access to Canton's synchronization capabilities and its institutional participant network.

## Conclusion

Canton represents one of the most technically rigorous attempts to solve a genuinely hard problem: building a blockchain infrastructure that can satisfy the confidentiality, atomicity, and regulatory requirements of institutional finance without sacrificing the programmability and composability that make blockchain useful in the first place. The architectural choices, no global EVM, sub-transaction privacy, participant node model, atomic cross-domain composability, are not arbitrary. Each one is a deliberate engineering decision that trades EVM familiarity for properties that matter deeply to the financial institutions that Canton is targeting.

For EVM developers, Canton represents both a challenge and an opportunity. The challenge is that almost everything familiar about how you interact with a blockchain is different: contract deployment, identity, wallet models, privacy, validator roles. The opportunity is that Canton's EVM compatibility layer creates a genuine on-ramp, and the technical problems Canton is solving, particularly atomic cross-domain composability and regulatory-grade privacy, are problems that the entire blockchain industry will need to address as it matures beyond retail DeFi into institutional-grade financial infrastructure.

The institutions investing in Canton are not doing so for speculative reasons. They are doing so because they have compliance, legal, and operational teams that have evaluated the technical architecture and concluded that it addresses requirements that alternative approaches do not. For EVM developers watching Canton's growing presence at Ethereum ecosystem events, the right question is not "why is this different from Ethereum" but "what problems does this architecture solve that mine does not." The answer to that question reveals exactly why institutional interest in Canton is accelerating.

## About Us

At SC Audit Studio, we specialize in protocol security assessments. Our team of experts has worked with companies like Aave, 1inch, and many more to conduct thorough security assessments across EVM and non-EVM environments.

[Reach out to us](https://scauditstudio.com/contact) for queries and security assessments!



## Tags

["Canton", "Daml", "Blockchain", "Institutional", "EVM", "Privacy", "DeFi", "Architecture"]



## FAQ

[
  {
    "question": "Is Canton an L2 on top of Ethereum?",
    "answer": "No. Canton is an independent distributed ledger protocol developed by Digital Asset. It is not built on Ethereum, does not inherit Ethereum's security, and is not an L2 rollup. Canton has an EVM-compatible layer that allows Solidity contracts to be deployed in a Canton-native execution environment, but this is a compatibility layer on top of Canton, not Canton running on top of Ethereum."
  },
  {
    "question": "What is the difference between a Synchronization Domain and the Global Synchronizer?",
    "answer": "The Global Synchronizer is the top-level Canton domain, operated by super validators, that provides canonical ordering and serves as the coordination point for the entire Canton Network. Synchronization Domains are application-specific subnetworks built on Canton's protocol, each with its own sequencers, mediators, and participant set. The Global Synchronizer is the backbone; Synchronization Domains are the application environments built on top of it."
  },
  {
    "question": "How does Canton achieve privacy without ZK proofs?",
    "answer": "Canton achieves privacy through architectural design rather than cryptographic obfuscation. Contract state is stored only at the participant nodes of the parties involved in each contract, it is never broadcast to a shared global ledger. Synchronization Domains receive only cryptographic commitments (hashes) that prove consistency without revealing content. This means competitors never receive the transaction data in the first place, removing the need for ZK proofs to hide it after the fact."
  },
  {
    "question": "Can I use my existing Solidity contracts on Canton?",
    "answer": "Yes, with caveats. Canton's EVM compatibility layer (based on Hyperledger Besu) allows Solidity contracts to run in a Canton-compatible environment. Standard EVM tooling like Hardhat, Foundry, and ethers.js works with this layer. However, Solidity contracts running on the EVM layer do not automatically get Canton's native sub-transaction privacy model, because Solidity's global state model is fundamentally different from Daml's party-scoped contract model. The EVM layer provides an accessible entry point, not a full translation of Canton's properties."
  },
  {
    "question": "What is the role of Canton Coin (CNT)?",
    "answer": "Canton Coin (CNT) is the native token of the Canton Network's Global Synchronizer. It is used to pay sequencing fees for using the Global Synchronizer's ordering services, to stake for super validator participation, and for governance voting on Global Synchronizer parameters. Application-level tokens on Canton subnetworks are separate and are represented as Daml contract instances on the relevant Synchronization Domain, not as CNT."
  },
  {
    "question": "What does 'atomic composability across subnetworks' actually mean in practice?",
    "answer": "It means that a single Canton transaction can involve contracts on multiple Synchronization Domains, and that transaction will either fully commit on all domains simultaneously or roll back on all of them. In practical financial terms, this enables Delivery vs. Payment settlement where a security on one domain is exchanged for cash on another domain, with the guarantee that neither leg settles unless both do, without bridges, relayers, or trusted intermediaries. This is impossible in the EVM ecosystem today without centralized trust assumptions."
  },
  {
    "question": "Do I need to learn Daml to build on Canton?",
    "answer": "Not necessarily to get started. The Canton EVM compatibility layer allows you to write Solidity and use familiar EVM tooling. However, to fully leverage Canton's native privacy model and build applications that correctly use sub-transaction privacy, you will eventually need to engage with Daml. Daml's learning curve is significant for EVM developers but its concepts, signatories, observers, choices, contract instances, map naturally to the logic of financial instruments and multi-party workflows."
  }
]
