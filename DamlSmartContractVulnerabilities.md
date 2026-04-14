# Common Vulnerabilities in Daml Smart Contracts

A deep-dive into Daml-specific security pitfalls drawn directly from the language specification, official documentation, and real patterns found in open-source Daml codebases, with vulnerable code, exploits, and fixes.

## Table of Contents

1. [Why Daml Vulnerabilities Are Different](#why-daml-vulnerabilities-are-different)
2. [Vulnerability 1: The Unilateral Archival Attack (Weak Signatory)](#vulnerability-1-the-unilateral-archival-attack-weak-signatory)
3. [Vulnerability 2: Divulgence Privacy Leak](#vulnerability-2-divulgence-privacy-leak)
4. [Vulnerability 3: Non-Consuming Choice Abuse](#vulnerability-3-non-consuming-choice-abuse)
5. [Vulnerability 4: Missing Precondition Guards](#vulnerability-4-missing-precondition-guards)
6. [Vulnerability 5: Flexible Controller Privilege Escalation](#vulnerability-5-flexible-controller-privilege-escalation)
7. [Vulnerability 6: Contract Key Maintainer Misconfiguration](#vulnerability-6-contract-key-maintainer-misconfiguration)
8. [Vulnerability 7: Authority Laundering via Intermediate Contracts](#vulnerability-7-authority-laundering-via-intermediate-contracts)
9. [Vulnerability 8: fetchByKey Authorization Bypass](#vulnerability-8-fetchbykey-authorization-bypass)
10. [Daml vs. Solidity: How the Threat Model Differs](#daml-vs-solidity-how-the-threat-model-differs)
11. [Security Checklist](#security-checklist)
12. [Summary](#summary)
13. [About Us](#about-us)
14. [FAQ](#faq)

## Why Daml Vulnerabilities Are Different

Daml eliminates entire categories of vulnerabilities that plague Solidity, there are no reentrancy attacks, no integer overflows, no unprotected self-destruct calls. The Daml runtime enforces authorization rules automatically, and the type system prevents most arithmetic and type errors at compile time.

But Daml introduces its own class of vulnerabilities. Because Daml is built around a **multi-party authorization model**, the attack surface shifts from "can I drain this contract?" to subtler questions: *Who can archive this contract? Who can see this data? Who can exercise this choice? Can someone act on a contract they shouldn't know about?*

These vulnerabilities are harder to spot precisely because Daml code looks clean and declarative. A single word, `signatory` vs `observer`, `consuming` vs `nonconsuming`, changes the entire security model of a contract.

This article analyzes each vulnerability using examples directly grounded in the Daml language specification, official tutorials, and open-source Daml codebases.



## Vulnerability 1: The Unilateral Archival Attack (Weak Signatory)

### What It Is

The `SimpleIou` contract has one major problem: the contract is only signed by the issuer. The signatories are the parties with the power to create and archive contracts. If Alice gave Bob a SimpleIou for $100 in exchange for some goods, she could just archive it after receiving the goods.

This is the most fundamental Daml security mistake. When only one party is a `signatory`, that party can unilaterally archive the contract at any time, destroying the other party's rights without their consent.

### Vulnerable Code

```haskell
template SimpleIou
  with
    issuer : Party
    owner  : Party
    amount : Decimal
  where
    signatory issuer   -- ❌ only issuer signs
    observer  owner    -- owner has NO protection
```

### The Exploit

```haskell
exploit = script do
  alice <- allocateParty "Alice"
  bob   <- allocateParty "Bob"

  -- Alice creates an IOU for Bob worth $100
  iouId <- submit alice do
    createCmd SimpleIou with issuer = alice; owner = bob; amount = 100.0

  -- Bob delivers goods (off-ledger)...

  -- Alice simply archives the IOU, stealing Bob's $100
  submit alice do
    archiveCmd iouId   -- ✅ succeeds! Alice is the sole signatory
```

Bob has no recourse. The archive is legal because Alice, as sole signatory, can bring any transaction that references this contract.

### The Fix

For a party to have any guarantees that only those transformations specified in the choices are actually followed, they either need to be a signatory themselves, or trust one of the signatories to not agree to transactions that archive and re-create contracts in unexpected ways.

Add the owner as a co-signatory. Now archiving requires *both* parties' consent:

```haskell
template Iou
  with
    issuer : Party
    owner  : Party
    amount : Decimal
  where
    signatory issuer, owner   -- ✅ both must authorize
```

### Trade-off

Adding the owner as a signatory means their signature is required to *create* the contract too, so issuance and transfer workflows need an explicit accept/delegate pattern to remain usable.

## Vulnerability 2: Divulgence Privacy Leak

### What It Is

Showing contracts to non-stakeholders through ledger projections is called divulgence. Divulgence needs to be considered when designing Daml models for privacy-sensitive applications.

In Daml, when a choice is exercised, every party who is an *informee* of any ancestor action in the transaction tree sees all the sub-actions, even contracts they are not a stakeholder on. This is not a bug in Daml; it is a deliberate design decision to enable delegation. But it becomes a vulnerability when developers do not realize they are leaking contract data to parties who should not have it.

### Vulnerable Code

```haskell
-- A trade settlement that reveals Alice's private IOU to Charlie
template TradeSettlement
  with
    alice   : Party
    bob     : Party
    charlie : Party   -- settlement agent
  where
    signatory alice, bob
    observer charlie

    -- Charlie triggers settlement
    choice Settle : ()
      controller charlie
      do
        -- Alice's private IOU is fetched here
        -- Charlie now SEES Alice's IOU details via divulgence
        iou <- fetch alicesPrivateIouId
        archive alicesPrivateIouId
        create Settlement with ...
```

### The Exploit

Charlie is not a stakeholder on Alice's IOU. But because the `Settle` choice, of which Charlie is a controller, has a `fetch alicesPrivateIouId` as a sub-action, Alice's IOU is **divulged to Charlie**. This kind of disclosure is often called "divulgence" and needs to be considered when designing Daml models for privacy-sensitive applications.

In financial applications this can mean: a settlement agent learns the full terms of a trade they are only supposed to facilitate, not observe.

### The Fix

Use Explicit Contract Disclosure (available in Canton 3.x+) instead of fetch-based divulgence. The disclosing party provides a signed proof of the contract at submission time, and the authorization restrictions of the Daml model still apply: the submitted command still needs to be well-authorized. The contract details are shared only for the duration of the transaction and are not permanently visible to the disclosee.

Alternatively, restructure the model so the settlement agent is a genuine observer on the IOU (making the disclosure explicit and permanent), or redesign the workflow to avoid fetching private contracts in mixed-party transactions.

## Vulnerability 3: Non-Consuming Choice Abuse

### What It Is

A nonconsuming choice does NOT archive the contract it is on when exercised. This means the choice can be exercised more than once on the same contract.

This is a feature, but if a developer forgets to mark a choice as `consuming` when it should be, the same choice can be exercised repeatedly on the same contract, potentially minting unlimited assets, issuing duplicate claims, or triggering the same action multiple times.

### Vulnerable Code

```haskell
template Voucher
  with
    issuer : Party
    holder : Party
    value  : Decimal
  where
    signatory issuer
    observer  holder

    -- ❌ Missing `consuming` keyword, this voucher can be redeemed infinitely
    nonconsuming choice Redeem : ContractId Payment
      controller holder
      do
        create Payment with
          payer  = issuer
          payee  = holder
          amount = value
```

### The Exploit

```haskell
exploit = script do
  issuer <- allocateParty "Issuer"
  holder <- allocateParty "Holder"

  voucherId <- submit issuer do
    createCmd Voucher with issuer; holder; value = 100.0

  -- Redeem the same voucher 5 times, 5 payments for 1 voucher
  submit holder do exerciseCmd voucherId Redeem
  submit holder do exerciseCmd voucherId Redeem
  submit holder do exerciseCmd voucherId Redeem
  submit holder do exerciseCmd voucherId Redeem
  submit holder do exerciseCmd voucherId Redeem
  -- ✅ All succeed. Holder just printed $500 from a $100 voucher.
```

### The Fix

```haskell
    -- ✅ Consuming choice: archives the voucher on first use
    choice Redeem : ContractId Payment
      controller holder
      do
        create Payment with
          payer  = issuer
          payee  = holder
          amount = value
```

The default for choices in Daml is consuming. The danger arises specifically when developers add `nonconsuming` for legitimate read-only choices and then, through copy-paste or inattention, apply it to choices that should only execute once.

## Vulnerability 4: Missing Precondition Guards

### What It Is

Daml provides `ensure` as a contract-level precondition and `assert` / `assertMsg` inside choices for runtime checks. When these are missing, contracts can be created or choices exercised with economically nonsensical or exploit-enabling values.

### Vulnerable Code

```haskell
template Token
  with
    issuer : Party
    owner  : Party
    amount : Decimal
    symbol : Text
  where
    signatory issuer
    observer  owner
    -- ❌ No `ensure`, nothing stops amount = 0.0 or amount = -1000.0

    choice Split : (ContractId Token, ContractId Token)
      with splitAmount : Decimal
      controller owner
      do
        -- ❌ No assertion, splitAmount could be 0, negative, or > amount
        t1 <- create this with amount = splitAmount
        t2 <- create this with amount = amount - splitAmount
        return (t1, t2)
```

### The Exploits

```haskell
-- Create a zero-value token (semantic nonsense)
submit issuer do
  createCmd Token with issuer; owner = alice; amount = 0.0; symbol = "MYT"

-- Split with a negative amount, creates a token with MORE than the original
submit alice do
  exerciseCmd tokenId Split with splitAmount = -100.0
  -- t1 gets -100.0 tokens, t2 gets amount - (-100.0) = amount + 100.0
  -- Net result: Alice has more tokens than she started with
```

### The Fix

```haskell
template Token
  with
    issuer : Party
    owner  : Party
    amount : Decimal
    symbol : Text
  where
    signatory issuer
    observer  owner
    ensure amount > 0.0   -- ✅ contract-level guard; all instances must satisfy this

    choice Split : (ContractId Token, ContractId Token)
      with splitAmount : Decimal
      controller owner
      do
        assertMsg "Split amount must be positive" (splitAmount > 0.0)
        assertMsg "Split amount must be less than total" (splitAmount < amount)
        t1 <- create this with amount = splitAmount
        t2 <- create this with amount = amount - splitAmount
        return (t1, t2)
```

The `ensure` clause is checked on every contract creation and re-creation. Combined with `assertMsg` in each choice, this builds defense in depth.

## Vulnerability 5: Flexible Controller Privilege Escalation

### What It Is

Daml supports **flexible controllers**, choices where the controller is not fixed at contract creation time but is passed as an argument at exercise time. This is a powerful feature that enables delegation patterns, but it becomes a critical vulnerability when the contract does not validate that the provided controller is actually authorized to act.

### Vulnerable Code

```haskell
template AssetTransfer
  with
    asset  : ContractId Token
    sender : Party
    receiver : Party
  where
    signatory sender

    -- ❌ The controller is a choice argument, anyone can pass any party here
    choice Accept : ContractId Token
      with approver : Party    -- flexible controller
      controller approver
      do
        exercise asset Transfer with newOwner = receiver
```

### The Exploit

The contract was intended to let only the `receiver` accept. But because `approver` is a flexible controller taken from the choice arguments, any party who can see this contract (via divulgence or as an observer) can attempt to pass themselves as `approver` and exercise the choice, potentially redirecting the asset to an unintended receiver.

### The Fix

As good practice, each Daml application workflow should have business logic preconditions that safeguard against misuse. The Offer_Accept choice has a flexible controller (buyer) that is provided as an argument.

When using flexible controllers, always add an `assert` that ties the runtime controller to the statically known party:

```haskell
    choice Accept : ContractId Token
      with approver : Party
      controller approver
      do
        assertMsg "Only the designated receiver may accept" (approver == receiver)
        exercise asset Transfer with newOwner = receiver
```

Better still, avoid flexible controllers entirely for security-critical choices. Use a fixed `controller receiver` instead:

```haskell
    choice Accept : ContractId Token
      controller receiver   -- ✅ fixed at contract creation time
      do
        exercise asset Transfer with newOwner = receiver
```

## Vulnerability 6: Contract Key Maintainer Misconfiguration

### What It Is

Contract keys give you a way to look up a contract by a stable identifier rather than its contract ID. But the maintainers matter since they affect authorization, much like signatories and observers. Maintainers of keys prevent double allocation or incorrect lookups.

The most common mistake is using a key that does not include a party as the maintainer, or using a party who is not a signatory as the maintainer. Both cause the contract to fail at runtime or, worse, silently break key uniqueness guarantees.

### Vulnerable Code

```haskell
template UserAccount
  with
    bank  : Party
    owner : Party
    accountNumber : Text
    balance : Decimal
  where
    signatory bank
    observer  owner

    -- ❌ Key uses accountNumber alone, no party as maintainer
    key accountNumber : Text
    maintainer bank
    -- ❌ `bank` is not derivable from the key `accountNumber` alone
    -- The maintainer expression must be a function of the key, not the template fields
```

### The Problem

Since the key is part of the contract, the maintainers must be signatories of the contract. However, maintainers are computed from the key instead of the template arguments. If the maintainer is referenced from template fields rather than computed from the key itself, the key uniqueness guarantee cannot be enforced across the ledger, because different participants compute different maintainer sets.

### The Fix

```haskell
    -- ✅ Key includes the maintaining party so maintainer can be derived from key
    key (bank, accountNumber) : (Party, Text)
    maintainer key._1          -- bank is first element of the key tuple
```

Now `bank` is computable from the key itself. The runtime can guarantee uniqueness: only one `UserAccount` with a given `(bank, accountNumber)` pair can exist at any time.

## Vulnerability 7: Authority Laundering via Intermediate Contracts

### What It Is

Authorization is never inherited from earlier execution contexts. Daml enforces this at the language level. However, developers can unintentionally construct chains of contract exercises that accumulate authority from multiple signatories, granting a downstream action more authority than was intended.

### Vulnerable Pattern

```haskell
template Delegation
  with
    delegator : Party
    delegatee : Party
  where
    signatory delegator
    observer  delegatee

    -- delegatee can create ANY contract that requires delegator's authority
    nonconsuming choice ExecuteAs : ContractId SomeContract
      with args : SomeContractArgs
      controller delegatee
      do
        create SomeContract with signatory = delegator; ...
        -- ❌ delegatee now has unlimited ability to act as delegator
```

This pattern, often written with good intentions for admin delegation, grants the `delegatee` the ability to create *any* contract that requires `delegator`'s signature. If `SomeContract` is a financial instrument, this is a complete authority handover.

### The Fix

Scope delegation tightly. Never create a general-purpose `ExecuteAs` choice. Instead, create specific, narrowly scoped choices for exactly the operations the delegatee is allowed to perform:

```haskell
    -- ✅ Delegatee can ONLY create tokens, nothing else
    nonconsuming choice IssueToken : ContractId Token
      with owner : Party; amount : Decimal; symbol : Text
      controller delegatee
      do
        create Token with
          issuer = delegator
          owner  = owner
          amount = amount
          symbol = symbol
```

## Vulnerability 8: fetchByKey Authorization Bypass

### What It Is

Daml provides two ways to look up a contract by key: `fetchByKey` and `lookupByKey`. They have subtly different authorization semantics that developers frequently confuse, leading to silent authorization failures that are hard to diagnose.

`fetchByKey` needs to be authorized by at least one stakeholder. If it fails, it doesn't guarantee that a contract with that key doesn't exist, just that the submitting party doesn't know about it, or there are issues with authorization.

### Vulnerable Code

```haskell
template Settlement
  with
    settler  : Party
    account  : (Party, Text)   -- bank, accountNumber
  where
    signatory settler

    choice Settle : ()
      controller settler
      do
        -- ❌ settler is not a stakeholder on UserAccount, this silently fails
        -- or returns wrong data via divulgence
        (_, acct) <- fetchByKey @UserAccount account
        -- settler proceeds assuming acct is the current valid state
        -- but acct may be stale, archived, or from a divulged context
        ...
```

### The Problem

`fetchByKey` requires the submitting party to be a **stakeholder** on the contract (signatory or observer). If they are not, the behavior is undefined, the call may fail at validation time even if it appeared to succeed at interpretation time. Limiting key usage to stakeholders also means that keys cannot be used to access a divulged contract, there can be cases where `fetch` succeeds and `fetchByKey` does not.

### The Fix

Ensure the fetching party is explicitly listed as an observer on the contract being fetched by key. This is not optional when key-based lookups are part of your authorization flow:

```haskell
template UserAccount
  with
    bank    : Party
    owner   : Party
    settler : Party   -- ✅ add the settler as an explicit observer
    ...
  where
    signatory bank
    observer  owner, settler   -- ✅ now settler can use fetchByKey
```

Alternatively, use `lookupByKey` and handle the `None` case explicitly, making it clear in the code that the absence of a result does not mean the contract does not exist.

## Daml vs. Solidity: How the Threat Model Differs

Understanding how Daml's vulnerability landscape differs from Solidity's helps prioritize what to look for in an audit.

| Threat Class | Solidity | Daml |
|---|---|---|
| Reentrancy | Critical (DAO hack) | Eliminated by design, no callbacks |
| Integer overflow | Critical (SafeMath needed) | Eliminated, Daml's `Decimal` and `Int` are safe by default |
| Unilateral archival | N/A | **Critical** (Vulnerability 1) |
| Privacy leak | N/A (all public) | **High**, divulgence (Vulnerability 2) |
| Double-spend (nonconsuming) | Handled by EVM | **Critical** if `nonconsuming` misused (Vulnerability 3) |
| Access control | Common (Parity, Bancor) | Managed by `signatory`/`controller`, but flexible controllers create risk (Vulnerability 5) |
| Oracle manipulation | Common | Less relevant, Daml is ledger-agnostic, no native oracle |
| Front-running | Common | Mitigated by Canton's privacy model |
| Upgrade risks | High (proxy patterns) | Managed, Daml has explicit upgrade framework |

Daml's runtime eliminates the most catastrophic low-level bugs. What remains is a set of **authorization logic errors**, mistakes in who can do what, to which contracts, and who can see the results. These require semantic understanding of the contract's business logic, not just code scanning.



## Security Checklist

Use this checklist when auditing a Daml contract:

**Signatories**
- [ ] Does every party that needs to be protected against unilateral archival appear as a `signatory`?
- [ ] Does every multi-party contract use the Propose-Accept pattern to gather all signatures?
- [ ] Are there any `signatory` lists that include parties not derivable from the template arguments?

**Choices**
- [ ] Is every one-time-use choice marked as `consuming`?
- [ ] Are `nonconsuming` choices genuinely intended to be repeatable?
- [ ] Are flexible controllers validated with `assertMsg` to match the intended party?
- [ ] Are all numeric inputs to choices bounds-checked with `assert`?

**Privacy**
- [ ] Does any `exercise` or `fetch` inside a choice expose contracts to unintended parties via divulgence?
- [ ] Are all `observer` lists minimal, only parties who genuinely need to see the contract?
- [ ] Has Explicit Contract Disclosure been considered for cross-party data sharing?

**Contract Keys**
- [ ] Is the maintainer party computable from the key expression (not from template fields)?
- [ ] Is the maintainer a `signatory` on the contract?
- [ ] Are all `fetchByKey` callers explicit stakeholders (signatory or observer)?

**Tests**
- [ ] Does each `submitMustFail` test target a specific, named invariant?
- [ ] Is each security property tested with the correct submitting party and arguments?
- [ ] Are there tests for boundary values (0, negative numbers, exactly-equal amounts)?



## Summary

Daml's design eliminates whole categories of smart contract vulnerabilities that have cost the broader blockchain ecosystem hundreds of millions of dollars. There are no reentrancy bugs, no integer overflows, and no unprotected destructors. The ledger model itself enforces authorization rules that Solidity developers have to implement manually.

What Daml does not eliminate is logic errors in how authorization is structured. The eight vulnerabilities in this article share a common root: a developer made a decision about `signatory`, `observer`, `controller`, or `consuming` without fully thinking through who that decision empowers, what they can do, and who will be able to see the results.

The most important mindset shift for Daml security is this: *do not think about "can this contract be hacked?" Think about "who can archive this, who can see this, and who can act on this, and should they be able to?"* Those three questions, applied to every template and every choice, will surface most of the vulnerabilities before they reach production.

**Daml Security Architecture:** [docs.daml.com/canton/architecture/security.html](https://docs.daml.com/canton/architecture/security.html)

**Daml Authorization Patterns:** [docs.daml.com/daml/patterns/authorization.html](https://docs.daml.com/daml/patterns/authorization.html)

**Common Errors Reference:** [docs.daml.com/canton/reference/error_codes.html](https://docs.daml.com/canton/reference/error_codes.html)

**Daml Privacy Model:** [docs.daml.com/concepts/ledger-model/ledger-privacy.html](https://docs.daml.com/concepts/ledger-model/ledger-privacy.html)



## About Us

At SC Audit Studio, we specialize in protocol security assessments.
Our team of experts has worked with companies like Aave, 1Inch and several more to conduct security assessments.
Partner with us to enhance your project's security and gain peace of mind.

[Reach out to us](https://scauditstudio.com/contact) for queries and security assessments!

## Tags
["Daml", "Smart Contracts", "Security", "Canton", "Blockchain", "Authorization", "Audit", "Vulnerabilities"]

## FAQ

[
  {
    "question": "Does Daml prevent reentrancy attacks?",
    "answer": "Yes. Daml's execution model does not allow external contract callbacks during execution, and the ledger model enforces that each transaction is committed atomically. The reentrancy pattern that drained the DAO in 2016 cannot occur in Daml."
  },
  {
    "question": "Can a Daml contract be hacked after deployment?",
    "answer": "The contract code itself cannot be modified post-deployment, and the Daml runtime enforces authorization rules on every transaction. However, logic vulnerabilities in the contract, such as the ones described in this article, can be exploited by any party who has access to the ledger. This is why design-time review is critical."
  },
  {
    "question": "What is the most common Daml vulnerability in practice?",
    "answer": "Based on the official documentation and community patterns, the most frequently encountered issue is incorrect signatory design, especially weak signatory modeling that allows unilateral archival. The second most common is missing precondition guards on choice arguments."
  },
  {
    "question": "How does Daml's privacy model affect security?",
    "answer": "Canton's privacy-by-default architecture means most parties do not see most contracts. This significantly reduces the attack surface compared to fully public blockchains. However, divulgence, the mechanism by which contracts are temporarily shown to non-stakeholders, must be carefully managed to prevent unintended data exposure."
  },
  {
    "question": "Should I use nonconsuming choices?",
    "answer": "Yes, but only when a choice genuinely needs to be exercisable multiple times on the same contract, for example, a read-only query choice or a notification mechanism. Any choice that transfers assets, creates payments, or redeems claims must be consuming."
  },
  {
    "question": "Is there a static analysis tool for Daml security?",
    "answer": "Daml Studio provides inline diagnostics for type errors and authorization failures. There is no equivalent of Slither or Mythril for Daml yet, which makes manual review and comprehensive submitMustFail test suites especially important for security assurance."
  }
]
