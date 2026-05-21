# Writing Correct Daml Contracts on Canton

Daml provides a robust foundation for building distributed applications on Canton right out of the box. The execution model ensures atomic and deterministic state transitions, abstracting away the underlying complexity of distributed consensus. Business logic and data schemas are strongly typed, and authorization is handled declaratively through \`signatory\` and \`controller\` keywords rather than manually written checks.

If you're coming from general-purpose languages or other smart contract platforms, this declarative approach drastically accelerates your development cycle. However, it also means that the design flaws you encounter tend to be architectural. Contracts compile fine and pass basic unit tests, but issues with multi-party workflow modeling, privacy boundaries, and ledger time semantics often only surface in production-like network topologies. This post covers the most common development pitfalls and the practical patterns to avoid them.

## Table of Contents

1. [Authorization: Controllers Are Not Access Control Lists](#1-authorization-controllers-are-not-access-control-lists)
2. [Signatory Consent Is Not the Same as Informed Consent](#2-signatory-consent-is-not-the-same-as-informed-consent)
3. [Observer Lists Are Permanent and Total](#3-observer-lists-are-permanent-and-total)
4. [`ensure` Blocks and Precondition Checks](#4-ensure-blocks-and-precondition-checks)
5. [Arithmetic: Multiplication Before Division](#5-arithmetic-multiplication-before-division)
6. [Consuming vs. Nonconsuming Choices](#6-consuming-vs-nonconsuming-choices)
7. [Contract Keys: Uniqueness Is Not Global](#7-contract-keys-uniqueness-is-not-global)
8. [Time Checks and Ledger Time Skew](#8-time-checks-and-ledger-time-skew)
9. [Delegation Chains](#9-delegation-chains)
10. [Stale Contract References and Upgrade Safety](#10-stale-contract-references-and-upgrade-safety)
11. [Testing What the Compiler Can't See](#11-testing-what-the-compiler-cant-see)
12. [Using CantonGuard](#using-cantonguard)
13. [About Us](#about-us)
14. [FAQ](#faq)

## 1. Authorization: Controllers Are Not Access Control Lists

The `controller` clause determines who can exercise a choice. The most common mistake is deriving the controller from a choice argument rather than from the contract's payload, which lets whoever exercises the choice pass themselves as the authorized party.

```daml
-- Risky: any party with visibility can pass themselves as requestor
choice Withdraw : ContractId Vault
  with
    requestor : Party
    amount : Decimal
  controller requestor
  do
    create this with balance = balance - amount
```

In Daml, a party needs to *see* a contract to exercise a choice on it, but visibility should never be treated as a security boundary. The custodian is a signatory and sees the contract. An operator or regulator added as an observer sees it too. Hardcode the controller to a party field on the contract itself:

```daml
choice Withdraw : ContractId Vault
  with
    amount : Decimal
  controller owner
  do
    assert (amount > 0.0 && amount <= balance)
    create this with balance = balance - amount
```

A related but less obvious variant is **authority smuggling**. Within a choice body, executing code carries the combined authorization of the contract's signatories *and* the choice controller. A template with a low-privilege controller (say, an `operator`) can still create contracts signed by a high-privilege `admin` if the choice body does so. The operator exercises a routine choice, and embedded in that choice body is a `create IOU with issuer = admin`. This is a template design problem, it can't be exploited at runtime, but it can be introduced at template authoring time, whether maliciously or carelessly. Audit every `create` and `exercise` inside a choice body against the question: "Do all authorizing parties actually intend this?"

 

## 2. Signatory Consent Is Not the Same as Informed Consent

When a party becomes a signatory on a contract, they implicitly authorize every choice on that contract, including choices controlled by other parties. This is the **signatory bait** pattern: a contract looks benign at acceptance time, but the template contains choices that let the other signatory act unilaterally using the victim's authority.

```daml
template Partnership
  with
    partyA : Party
    partyB : Party
  where
    signatory partyA, partyB

    -- partyA can issue bonds in partyB's name without further consent
    choice IssueBond : ContractId Bond
      with amount : Decimal
      controller partyA
      do
        create Bond with issuer = partyB; holder = partyA; faceValue = amount
```

Party B accepts the Partnership and is now bound to anything `partyA` does through it. The fix is to require explicit consent for each high-impact action via a propose-accept pattern, so Party B can accept or reject each individual Bond issuance.

In general, before accepting a signatory role on any contract, review *all* choices, not just the one you're about to exercise.

 

## 3. Observer Lists Are Permanent and Total

Observers can see the full contract payload. This sounds obvious, but it creates real problems in financial workflows where pricing, counterparty identity, or notional amounts are sensitive.

A supply chain contract that adds all downstream retailers as observers exposes the wholesale price to every one of them. A trade contract with a broad `allParticipants` list leaks pricing to parties who should only see a confirmation reference. In regulated environments, this can be a compliance violation, not just a design smell.

The right pattern is separation by visibility scope:

```daml
-- Wholesale terms: only manufacturer and distributor
template WholesaleContract
  with
    manufacturer : Party
    distributor   : Party
    wholesalePrice : Decimal
  where
    signatory manufacturer, distributor
    -- no observers: retailers cannot see this

-- Retail-facing summary: separate contract, separate fields
template RetailPriceList
  with
    distributor : Party
    retailers   : [Party]
    retailPrice : Decimal
  where
    signatory distributor
    observer retailers
```

Also watch out for **divulgence**: when a party exercises a choice that `fetch`es a contract they aren't a stakeholder on, the content of that contract becomes visible to them through the transaction tree, permanently, even after archival. Fetching should be a deliberate decision, not a side effect of workflow logic.

 

## 4. `ensure` Blocks and Precondition Checks

Daml's `ensure` keyword enforces invariants at contract creation time. Skipping it means contracts with invalid state, negative balances, zero amounts, rates above 100%, a borrower equal to the lender, can be created and will silently corrupt downstream logic.

```daml
-- Without ensure, any values compile and create
template Loan
  with
    borrower     : Party
    lender       : Party
    principal    : Decimal
    interestRate : Decimal
  where
    signatory lender, borrower
    ensure principal > 0.0
        && interestRate >= 0.0
        && interestRate <= 1.0
        && borrower /= lender
```

Also validate inside choice bodies. A `Withdraw` choice that doesn't assert `amount <= balance` will happily create a contract with a negative balance.

 

## 5. Arithmetic: Multiplication Before Division

Daml uses arbitrary-precision `Decimal` (10 decimal places) and doesn't silently overflow. But logic errors in calculations are still possible, and they tend to surface in exactly the choices where they hurt most, fee calculations, voting thresholds, pro-rata distributions.

The classic mistake is dividing before multiplying:

```daml
-- Wrong: integer division truncates first
-- userStake=1, poolSize=3, total=900 → (1/3)*900 = 0*900 = 0
calculateShare : Int -> Int -> Int -> Int
calculateShare total userStake poolSize =
    (userStake / poolSize) * total
```

Multiply first, then divide:

```daml
-- Correct: (1.0 * 900.0) / 3.0 = 300.0
calculateShare : Decimal -> Decimal -> Decimal -> Decimal
calculateShare total userStake poolSize =
    (userStake * total) / poolSize
```

And always guard against division by zero, it aborts the transaction at runtime, which can become a denial-of-service vector if a divisor is derived from contract state that can legitimately reach zero (an empty member list, a depleted pool, etc.):

```daml
assertMsg "No members to distribute to" (members > 0)
let perMember = totalReward / intToDecimal members
```

 

## 6. Consuming vs. Nonconsuming Choices

By default, choices in Daml are consuming, they archive the contract when exercised. A read-only query choice that forgets the `nonconsuming` keyword will destroy the contract every time it's called. This is easy to miss and hard to debug after the fact.

```daml
-- Without nonconsuming, calling LookupEntry destroys the Registry
nonconsuming choice LookupEntry : Optional Party
  with name : Text
  controller admin
  do
    return (lookup name entries)
```

Similarly, if an asset-handling choice archives a contract without creating a replacement, the asset is permanently gone. Always pair `archive` (or a consuming exercise) with an explicit `create` of the replacement:

```daml
choice SettleTrade : ContractId Asset
  controller buyer
  do
    asset <- fetch assetCid
    exercise assetCid Archive
    create asset with owner = buyer  -- explicit transfer, not implicit
```

 

## 7. Contract Keys: Uniqueness Is Not Global

Contract keys in Canton are only enforced as unique among active contracts visible to the participant processing a given transaction. Under concurrent submissions, two transactions can both observe `None` from `lookupByKey` and both succeed, producing duplicate-keyed contracts.

The username registration case is the textbook example: two users register the same handle concurrently, both `lookupByKey` calls return `None`, both commits land, and you have two accounts with the same key.

The fix is to serialize creation through a consuming generator contract, so concurrent submissions contend on the same contract and only one can win:

```daml
-- Consuming choice: at most one concurrent submission can succeed
choice Register : (ContractId RegistrationService, ContractId Username)
  with user : Party; name : Text
  controller registrar
  do
    optCid <- lookupByKey @Username (registrar, name)
    case optCid of
      Some _ -> abort "Username taken"
      None -> do
        userCid <- create Username with registrar; user; name
        svcCid  <- create this
        return (svcCid, userCid)
```

Also handle the case where `lookupByKey` returns `None` for visibility reasons rather than absence. If the looking-up party isn't an observer on the target contract, they simply can't see it. `None` doesn't mean the contract doesn't exist, it means *you* can't see it. Build explicit handling for both branches and document the visibility assumptions.

 

## 8. Time Checks and Ledger Time Skew

`getTime` returns the ledger effective time, which is chosen by the submitter within a configurable tolerance window. A motivated submitter can pick any time within that window, which means they can influence which side of a time boundary they land on.

For typical deadline checks this is usually fine. It becomes a problem when two parties have conflicting interests at a boundary, a borrower claiming just before expiry, an options holder exercising at the last second, an escrow timing out, and the submitter can cherry-pick ledger time within the tolerance to be on the side that benefits them.

The practical mitigations are: configure the domain's `ledger-time-record-time-tolerance` as tightly as your use case allows, and for high-stakes time-sensitive operations, avoid placing the threshold at an exact boundary where one side or the other benefits a party with submission control.

 

## 9. Delegation Chains

Daml's authority model allows choice bodies to create new contracts that carry a signatory's authority. Unconstrained delegation lets the original delegate pass that authority to anyone, indefinitely:

```daml
-- delegate can sub-delegate to any party, recursively
choice SubDelegate : ContractId DelegatedRight
  with newDelegate : Party
  controller delegate
  do
    create DelegatedRight with owner; delegate = newDelegate
```

Add a depth counter and enforce it in the choice:

```daml
template DelegatedRight
  with
    owner          : Party
    delegate       : Party
    remainingDepth : Int
  where
    signatory owner
    observer delegate
    ensure remainingDepth >= 0

    choice SubDelegate : ContractId DelegatedRight
      with newDelegate : Party
      controller delegate
      do
        assertMsg "Delegation depth exceeded" (remainingDepth > 0)
        create DelegatedRight with
          owner; delegate = newDelegate
          remainingDepth = remainingDepth - 1
```

 

## 10. Stale Contract References and Upgrade Safety

Storing a `ContractId` in a long-lived contract is risky, by the time the reference is used, the target may have been archived. Prefer contract keys for cross-contract references; they survive the archive-and-recreate cycle that upgrades or workflow resets produce.

For upgrade workflows specifically: the upgrade choice body inherits both signatories' authority and can rewrite fields unilaterally if only one controller is required. Always use a propose-accept pattern for upgrades so both parties consent to the new terms:

```daml
-- Issuer proposes; holder must explicitly accept before the old contract is archived
choice ProposeUpgrade : ContractId UpgradeProposal
  controller issuer
  do
    create UpgradeProposal with issuer; holder; amount; oldCid = self
```

 

## 11. Testing What the Compiler Can't See

Daml Script tests tend to cover happy paths well. The failure modes above mostly surface in scenarios that feel awkward to write: wrong-party submissions, boundary timestamps, partial ledger visibility. A few things worth adding explicitly:

- Exercise every choice as the wrong party and confirm rejection.
- Test at exactly the deadline boundary, not just before and after.
- Test `lookupByKey` from a party that isn't an observer.
- After every asset operation, assert conservation: inputs and outputs sum to the same value.
- Separate test DARs from production DARs, Digital Asset's SDLC guidelines explicitly flag this as a common deployment mistake.

 

## Using CantonGuard

After you've thought through the above and written your tests, [CantonGuard](https://github.com/SCAuditStudio/CantonGuard) is worth running as a final pass before shipping workflow changes. It's a Claude/Codex skill from SC Audit Studio that reviews Daml contracts specifically for authorization, privacy, and contract-key issues.

Installation is one step:

```
Install this skill: https://github.com/SCAuditStudio/CantonGuard
```

Point it at the files you're actively changing:

```
Use $canton-guard-by-scas on src/MyWorkflow.daml
```

It's most useful on targeted modules rather than whole repos. Running it more than once tends to surface different issues across passes, so it's worth doing on anything you ship to production.

It won't replace reading your own code, and it doesn't catch everything covered in this post. What it does well is flag issues you normalize after staring at a contract for hours, a controller assignment that looks fine in isolation but is wrong given the signatory set, an observer list that's wider than it needs to be, a key lookup with missing error handling. As a second pair of eyes on the specific categories it covers, it earns its place in a Canton development workflow.

## About Us

At SC Audit Studio, we specialize in protocols security assessments.
Our team of experts has worked with companies like Aave, 1Inch and several more to conduct security assessments.
Partner with us to enhance your project's security and gain peace of mind.

[Reach out to us](https://scauditstudio.com/contact) for queries and security assessments!

## Tags
["Daml", "Canton", "Smart Contracts", "Security", "Auditing", "Canton Development"]

## FAQ
[
  {
    "question": "Is Daml safer than Solidity?",
    "answer": "Daml eliminates entire classes of vulnerabilities like reentrancy and integer overflow by design. However, logic errors and misconfigured authorizations can still occur, which require careful review."
  },
  {
    "question": "What is signatory bait in Daml?",
    "answer": "Signatory bait is a pattern where a contract looks benign at the time a party accepts being a signatory, but the template contains choices that let another party act unilaterally using the first party's authority."
  },
  {
    "question": "Are observers permanently attached to a Daml contract?",
    "answer": "Yes, once a party is an observer on a contract, they can see its full payload. This visibility remains indefinitely, even after the contract is archived."
  },
  {
    "question": "Can I use CantonGuard for automated security checks?",
    "answer": "Yes, CantonGuard is an automated tool built by SC Audit Studio that reviews Daml contracts for common authorization, privacy, and contract key issues before deployment."
  }
]

 
