# AI and Formal Verification in DeFi: Practical Uses and Limits

If you searched for "AI formal verification smart contracts", "formal verification in DeFi", or "can AI replace a smart contract audit", this article is for you. It explains a narrow use case: using AI to draft tests, invariants, and formal specifications, then checking those drafts with reviewers and provers.

Formal verification proves that code satisfies a stated property. AI can draft candidate properties, summarize code paths, search for similar historical bugs, and generate tests. The pairing works when protocol behavior is written down clearly enough for a reviewer and a prover to check.

## TLDR

- Formal verification gives evidence about specific properties. A proof can pass even when an important property is missing.
- AI is best used for first drafts: invariants, test cases, audit notes, and specification templates.
- DeFi protocols need properties around accounting, permissions, collateral, liquidations, oracle assumptions, upgrades, and emergency controls.
- AI-generated specifications need human review and prover checks before they become part of the assurance case.
- AI plus formal verification can reduce audit prep work. Threat modeling, manual review, economic analysis, and operational security still matter.
- This work fits best before a formal audit, when the scope is frozen and the team can still fix design-level issues.

![AI-assisted formal verification process](https://raw.githubusercontent.com/SCAuditStudio/SCASArticles/main/images/ai-formal-verification-workflow.svg)

## Table of Contents

1. [Why This Matters for DeFi](#why-this-matters-for-defi)
2. [What Formal Verification Actually Proves](#what-formal-verification-actually-proves)
3. [Where AI Fits](#where-ai-fits)
4. [The Specification Bottleneck](#the-specification-bottleneck)
5. [Research Examples Worth Knowing](#research-examples-worth-knowing)
6. [How This Fits Into Audit Prep](#how-this-fits-into-audit-prep)
7. [Case Studies: Compound and Uniswap](#case-studies-compound-and-uniswap)
8. [What AI and Formal Verification Still Miss](#what-ai-and-formal-verification-still-miss)
9. [Practical Checklist for DeFi Teams](#practical-checklist-for-defi-teams)
10. [Conclusion](#conclusion)
11. [About Us](#about-us)
12. [FAQ](#faq)

## Why This Matters for DeFi

DeFi protocols are difficult to secure because they combine software correctness with financial correctness. A normal application bug might produce a bad user experience or require a rollback. A DeFi bug can move funds, create bad debt, break collateral accounting, or let an attacker extract value in a single transaction.

The scale is large enough that small logic errors matter. At the time of writing in July 2026, the [DefiLlama hacks database](https://defillama.com/hacks) records more than $16 billion in total value hacked across crypto systems, with more than $7.8 billion categorized as DeFi. The live [DefiLlama TVL dashboard](https://defillama.com/) still shows tens of billions of dollars locked in DeFi protocols. These figures move as new incidents and market prices change, but the order of magnitude is clear: small implementation mistakes can sit next to very large balances.

Risk determines depth. A simple token with standard access control has a different risk profile from a lending market, perpetuals exchange, bridge, vault, or restaking system. As a protocol depends more on composable state across assets, accounts, chains, and price feeds, it becomes more valuable to write key invariants explicitly and check them mechanically.

Audits, tests, fuzzing, monitoring, and formal verification answer different questions:

- Tests ask whether known examples behave as expected.
- Fuzzing asks whether many generated inputs can break an assertion.
- Static analysis asks whether known code patterns appear.
- Formal verification asks whether all executions in a modeled scope satisfy a stated property.
- Manual review asks whether the model, assumptions, and business logic make sense.

AI can support several of these steps. In DeFi, a confident wrong answer is still wrong.

## What Formal Verification Actually Proves

Formal verification is a method for proving that software satisfies a mathematical specification. In smart contracts, that specification is usually expressed as rules, invariants, preconditions, postconditions, or temporal properties.

Examples of practical DeFi properties include:

- A user cannot withdraw more shares than they own.
- Total shares are consistent with accounted assets.
- A liquidation cannot make a healthy account insolvent.
- Only authorized roles can change risk parameters.
- An oracle update must be fresh enough for the function that uses it.
- A paused market cannot accept new deposits, but withdrawals remain available unless an explicitly documented emergency state applies.
- Upgrades cannot corrupt storage layout or bypass governance delay.

Tools such as the [Certora Prover](https://docs.certora.com/en/latest/docs/prover/index.html) can check whether contract code satisfies properties written in a formal specification language. The result covers more ground than a unit test because the prover reasons over a broad space of possible executions rather than a few concrete examples.

The boundary matters: the prover checks the specification. Unstated intent sits outside the proof. If the specification says "only the owner can call this function", the prover can check that access rule. If the real problem is that the owner has too much power, or that governance can change a parameter too quickly, the proof will miss it unless that risk is encoded.

Passing rules give evidence about those claims only. A good formal verification effort starts by deciding which claims are worth proving.

## Where AI Fits

Writing good specifications is slow. AI can read a codebase quickly, compare code against known patterns, draft candidate invariants, suggest missing tests, and explain relationships between functions. This helps during audit preparation, especially when a protocol has several modules and no single engineer has all assumptions in their head.

The strongest AI use cases are constrained:

- Drafting candidate invariants from code and documentation.
- Translating natural language requirements into rough formal properties.
- Searching prior audits for similar accounting or access-control patterns.
- Generating fuzz tests that target state transitions beyond happy paths.
- Summarizing privileged roles, external dependencies, and trust assumptions.
- Suggesting counterexamples when a property fails.

Asking an AI system whether a protocol is safe is too broad to produce reliable output. A model may produce plausible prose while missing the economic mechanism that matters. DeFi bugs often live in the interaction between correct-looking components: a rounding rule, a stale price, a delayed governance update, a fee-on-transfer token, or a liquidation incentive that changes behavior under stress.

Generated properties are inputs to a security process. Reviewers and verification tools decide which ones deserve trust.

## The Specification Bottleneck

The main cost of formal verification is writing the right properties.

A DeFi team has to decide what must remain true across all valid states. For a lending protocol, that might include collateral valuation, debt accounting, interest accrual, liquidation eligibility, reserve accounting, and market pause behavior. For an AMM, it might include pool accounting, fee growth, liquidity position ownership, hook behavior, and swap invariants. For a vault, it might include share price behavior, withdrawal queues, strategy accounting, and curator permissions.

This work is slow because important properties are often hidden in cross-function behavior. A function called `withdraw` may be locally correct and still unsafe if it interacts with delayed accounting. An oracle check may be present and still insufficient if the feed can be stale during a market move. A governance delay may exist and still be too short for users to exit after a parameter change.

AI can reduce blank-page work by creating a first draft of properties. That draft should then be filtered:

1. Remove vacuous properties that are true but unimportant.
2. Remove properties that only restate implementation details.
3. Add economic properties that the code cannot infer by itself.
4. Add assumptions about trusted dependencies.
5. Run the properties through tests, fuzzing, mutation testing, and formal verification where appropriate.

The useful output is a short list of properties that would catch real failures before funds are at risk.

## Research Examples Worth Knowing

Several research projects show where AI-assisted formal methods are becoming practical. They point toward a specific pattern: constrained generation followed by mechanical checking.

[FLAMES](https://arxiv.org/abs/2510.21401) is a research system that fine-tunes language models to synthesize Solidity `require` guards. The authors trained on invariants extracted from 514,506 verified contracts and reported a 96.7% compilability rate for generated guards. Since uncompilable output is immediately unusable, this metric matters. The broader point is that domain-specific training performs better than generic code generation for smart contract safety patterns.

[InvCon+](https://arxiv.org/abs/2401.00650) takes a different route. It infers likely invariants from observed behavior and then uses static verification to reject candidates that do not hold. That distinction matters. A property that appears true in transaction traces is only a guess. A property that survives a verifier has stronger evidence behind it, assuming the model and scope are appropriate.

[PropertyGPT](https://www.ndss-symposium.org/ndss-paper/propertygpt-llm-driven-formal-verification-of-smart-contracts-through-retrieval-augmented-property-generation/) uses retrieval-augmented generation to adapt human-written properties from prior verified contracts and audit reports. This addresses a common audit problem: a new protocol may look unique, but parts of it often resemble earlier systems. Reusing the shape of prior properties can help a reviewer avoid starting from scratch.

The shared pattern matters. These systems generate candidates and then use compilers, static analyzers, provers, or human review to reject weak outputs. That check is the security-relevant part.

## How This Fits Into Audit Prep

For a DeFi team, AI-assisted formal verification fits best before the formal audit starts. This is the same stage where teams should freeze scope, document assumptions, clean up tests, and decide which properties matter. For broader preparation steps, see our guide on [how to prepare for a smart contract audit](https://scauditstudio.com/blog/Maximizing-the-Value-of-Your-Smart-Contract-Audit).

A practical sequence looks like this:

1. Write down protocol intent in plain language.
2. List assets, roles, markets, oracles, dependencies, upgrade paths, and emergency controls.
3. Ask AI tools to draft candidate invariants and tests from the code and documentation.
4. Have engineers and auditors remove weak, duplicate, or implementation-specific properties.
5. Convert the selected properties into tests, fuzzing assertions, or formal specifications.
6. Run the verification tools and inspect counterexamples.
7. Fix either the code, the property, or the documented assumption.
8. Keep the final property set in the repository so regressions can be checked later.

This also helps with audit cost. A formal audit is slower when the reviewer has to infer basic intent from code alone. If the team already has a property list, a roles table, and a set of invariant tests, reviewers can spend more time on deeper questions. For cost context, see our [smart contract audit pricing guide](https://scauditstudio.com/blog/Why-Some-Smart-Contract-Audits-Are-So-Expensive-2025-Edition).

The first property list can be rough. It needs to be explicit enough that reviewers can challenge it. Security work improves when assumptions are visible.

## Case Studies: Compound and Uniswap

The [Compound V3 Comet formal verification writeup](https://medium.com/certora/detecting-corner-cases-in-compound-v3-with-formal-specifications-b7abf137fb15) is a practical example because the bug was not a simple reentrancy issue or missing access modifier. The issue involved collateral flags and a bit-vector representation of supplied assets. The teams wrote correctness rules for the protocol and the prover found a corner case before deployment.

A carefully chosen property can expose a state combination that ordinary tests are unlikely to cover. Bit-level mistakes, accounting edge cases, and multi-asset state transitions are exactly the kind of areas where formal methods can help.

The [Uniswap V4 periphery formal verification contest repository](https://github.com/Certora/uniswap-v4-periphery-cantina-fv) shows another direction: rewarding researchers for writing and verifying high-coverage properties. This changes the incentive from only finding a bug to also improving the specification surface. That matters because strong properties can keep catching regressions after the contest ends.

AI can assist both patterns. For a Compound-like system, it can suggest properties around collateral membership, liquidation eligibility, and accounting consistency. For a contest-like setting, it can help participants draft broad mechanical properties quickly, while humans focus on economic intent and adversarial scenarios.

## What AI and Formal Verification Still Miss

The biggest risk is false confidence. A passing proof can be weak if the property is weak. A generated invariant can be irrelevant. A convincing AI explanation can be wrong. These failure modes are not theoretical; they are normal software risks made more expensive by financial context.

Several categories remain difficult:

- Economic attacks that depend on market behavior, liquidity, MEV, or incentives.
- Oracle manipulation where the code accepts a price that is technically valid but economically unsafe.
- Governance attacks where the contract follows the rules but the rules allow harmful parameter changes.
- Cross-chain assumptions where finality, message ordering, or bridge accounting is modeled incorrectly.
- Operational security failures involving compromised signers, frontends, CI systems, cloud keys, or deployment machines.

That last category matters because major Web3 losses also come from compromised devices, transaction simulation gaps, private-key exposure, and malicious interfaces. We cover that side of the problem in our article on [Web3 OpSec incidents and controls](https://scauditstudio.com/blog/Web3OpSecHacks).

Self-healing smart contracts are easy to overstate. Automatic pausing after an invariant break can help. Automatic patches to financial logic need a much higher bar, especially when the patch can create new risk or block withdrawals.

For now, controlled automation is easier to defend: alerts, invariant monitors, rate limits, circuit breakers, and emergency actions with clearly documented authority. Fully autonomous remediation belongs in the high-risk infrastructure category.

## Practical Checklist for DeFi Teams

Use AI-assisted formal verification when at least some of these conditions are true:

- The protocol manages significant TVL or will be integrated by other protocols.
- There are multiple asset types, collateral types, or market states.
- Liquidation, share pricing, fee accrual, or oracle logic affects solvency.
- Privileged roles can change parameters, pause markets, upgrade contracts, or move funds.
- The team can explain intended behavior in writing.
- The repository has stable scope before the audit starts.

Before writing formal specifications, prepare these materials:

- A plain-language protocol overview.
- A list of critical invariants.
- A roles and permissions table.
- Oracle assumptions, including staleness and fallback behavior.
- Asset assumptions, including non-standard token behavior.
- Upgrade and pause assumptions.
- Known limitations and accepted risks.
- Existing unit, integration, fuzz, and invariant tests.

When using AI, keep these review rules:

- Review generated properties before adding them to the audit package.
- Prefer properties about protocol behavior over properties that merely mirror implementation details.
- Check whether each property would catch a real failure.
- Look for missing liveness properties as well as safety properties.
- Run mutation tests or bug-injection tests when possible to see whether properties catch seeded faults.
- Keep counterexamples and rejected properties as audit context.

The output should be a tighter audit scope, clearer assumptions, and a better test and specification suite. A long list of generic properties is usually a sign that the review pass was too shallow.

## Conclusion

AI and formal verification fit together when their jobs stay separate. AI drafts candidate properties, finds similar patterns, and reduces repetitive analysis. Formal verification proves selected properties within a defined model. Human reviewers decide which properties matter and which assumptions are unsafe.

For high-risk DeFi systems, this can mean fewer obvious issues entering the audit, better coverage of critical state transitions, and better regression protection after deployment.

The standard should remain simple: generated output needs review, testing, and, where appropriate, mechanical verification.

# About Us

At SC Audit Studio, we specialize in protocols security assessments.
Our team of experts has worked with companies like Aave, 1Inch and several more to conduct security assessments.
Partner with us to enhance your project's security and gain peace of mind.

[Reach out to us](https://scauditstudio.com/contact) for queries and security assessments!

## Tags

["AI","Security","Formal-Verification","DeFi","Smart-Contracts"]

## FAQ

[
  {
    "question": "Can AI replace formal verification for smart contracts?",
    "answer": "No. AI can suggest candidate properties, tests, and explanations. Formal verification is still needed when the goal is to prove that code satisfies a stated property."
  },
  {
    "question": "Can formal verification prove that a DeFi protocol is completely safe?",
    "answer": "Formal verification proves specific properties under specific assumptions. If an important property is missing, or if an external dependency is modeled incorrectly, a proof can pass while the protocol still has risk."
  },
  {
    "question": "Where is AI most useful in formal verification?",
    "answer": "AI is useful for drafting invariants, finding similar patterns from prior audits, generating test ideas, summarizing dependencies, and helping interpret counterexamples. The generated output still needs human review and mechanical checking."
  },
  {
    "question": "Which DeFi protocols benefit most from AI-assisted formal verification?",
    "answer": "The biggest benefit is usually in protocols with complex accounting, liquidations, vault shares, multiple collateral assets, oracle dependencies, cross-chain messaging, or upgradeable governance. Simple contracts may not justify the overhead."
  },
  {
    "question": "When should a team start writing formal properties?",
    "answer": "Start before the formal audit, once the main architecture and scope are stable. Early properties help clarify intent, improve tests, and give auditors better context."
  }
]
