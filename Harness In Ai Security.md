# AI Agent Harnesses for Smart Contract Audits: What Matters and What Lasts

Teams comparing AI smart contract audit tools are often comparing three things at once: the underlying model, the agent harness around it, and the checks used to verify its findings. A benchmark score rarely shows which layer produced the result. That makes it difficult to answer practical questions such as: Does this tool find vulnerabilities because its model is better? Is it using better prompts and tools? Or does it have an independent verification pipeline that rejects weak findings?

If you're comparing AI smart contract audit tools or trying to understand AI agent harnesses, you need to know what a harness does, what current benchmarks actually show, which components may lose value as models improve, and which controls remain necessary regardless of model capability.

Do not read this as "the harness will not matter." Harnesses matter now and will continue to matter as software. However, features that only compensate for a temporary model limitation are unlikely to remain durable differentiators. Verification, reproducibility, permissions, and accountability solve different problems and do not become optional when the model improves.

## Table of Contents

1. [TLDR](#tldr)
2. [What Is an AI Agent Harness?](#what-is-an-ai-agent-harness)
3. [What Coding Benchmarks Show About Harness Engineering](#what-coding-benchmarks-show-about-harness-engineering)
4. [What EVMbench Shows About Smart Contract Auditing](#what-evmbench-shows-about-smart-contract-auditing)
5. [Why Smart Contract Auditing Is a Different Task](#why-smart-contract-auditing-is-a-different-task)
6. [Two Types of Harness Components](#two-types-of-harness-components)
7. [What May Lose Value as Models Improve](#what-may-lose-value-as-models-improve)
8. [What Does Not Lose Its Purpose](#what-does-not-lose-its-purpose)
9. [How to Evaluate an AI Smart Contract Audit Tool](#how-to-evaluate-an-ai-smart-contract-audit-tool)
10. [Limits of the Current Evidence](#limits-of-the-current-evidence)
11. [Conclusion](#conclusion)
12. [About Us](#about-us)
13. [FAQ](#faq)

## TLDR

- An AI agent harness is the software around a model: prompts, context selection, tools, memory, retries, permissions, tracing, and validation steps.
- Harness changes can produce large gains with a fixed model. LangChain reported a Terminal-Bench 2.0 increase from 52.8% to 66.5% after changing the harness while keeping the model fixed.
- Smart contract results are less settled. The initial EVMbench release reported a best detection score of 45.6% and a best exploit score of 72.2% on a smaller exploit subset. A later re-evaluation found that scaffold choice and dataset choice materially changed results.
- Prompt checklists, rigid chunking rules, and retry strategies may become less valuable when they only patch a specific model weakness. They may still remain useful for cost, workflow, or product-control reasons.
- Fuzzing, invariant tests, static analysis, formal verification, reproducible exploits, sandboxing, and human sign-off provide independent evidence or risk control. Better models do not remove their purpose.
- Evaluate tools on held-out protocol code, reproducible findings, false-positive burden, coverage, cost, permissions, and remediation workflow—not a single leaderboard score.

## What Is an AI Agent Harness?

A model accepts input and produces output. An agent harness turns that model into a working system. In a coding or security product, the harness may decide:

- which repository files enter the model's context;
- how the system maps contracts, dependencies, roles, and call paths;
- which instructions and vulnerability checklists the model receives;
- whether the model can use a compiler, test runner, fuzzer, debugger, or blockchain node;
- how long the agent can run and when it should retry;
- whether several agents review the same code or challenge one another's findings;
- what the agent can read, modify, deploy, or execute;
- how traces, evidence, and intermediate results are stored; and
- what must be verified before a finding reaches a developer or client.

The word "harness" is sometimes used more narrowly to mean the prompt and tool loop. When I say "harness," I mean the broader product layer because permissions, evidence handling, and verification affect the security outcome even when they do not improve the model's raw reasoning.

That distinction matters when comparing products. Two tools can use the same model and behave differently because one retrieves the relevant files, runs tests, and verifies reachable code paths while the other only asks for a prose review. Conversely, two tools can have similar interfaces while using models with different vulnerability-discovery ability. A product-level result combines both layers.

## What Coding Benchmarks Show About Harness Engineering

[LangChain](https://www.langchain.com/blog/improving-deep-agents-with-harness-engineering) published a useful fixed-model experiment in February 2026. Its coding agent improved from 52.8% to 66.5% on [Terminal-Bench 2.0](https://arxiv.org/abs/2601.11868) while keeping GPT-5.2-Codex fixed. The changes were made to the system prompt, tools, and middleware. The score moved from just outside the top 30 to the top five at the time of publication.

This result shows that the model name alone is not enough to predict agent performance. The agent must be encouraged to test its work, given tools that expose useful feedback, and guided through failure modes that appear in long-running terminal tasks.

Research on automated harness search points in the same direction. The [Stanford IRIS Lab Meta-Harness paper](https://arxiv.org/abs/2603.28052) describes a system that inspects prior harness code, scores, and execution traces before proposing new harnesses. On Terminal-Bench 2.0, its discovered harness ranked first among the reported agents using the same Claude Haiku 4.5 base model. That is a narrower result than beating every agent on the benchmark, but it still demonstrates meaningful variation around a fixed model.

These experiments concern coding and terminal tasks, not complete smart contract audits. Terminal-Bench evaluates whether an agent can finish verifiable work in a command-line environment. That includes planning, tool use, editing, testing, and recovery from errors. A harness has many direct ways to improve those behaviors.

## What EVMbench Shows About Smart Contract Auditing

EVMbench was developed by researchers from [OpenAI](https://cdn.openai.com/evmbench/evmbench.pdf), [Paradigm](https://www.paradigm.xyz/2026/02/evmbench), and [OtterSec](https://osec.io/). The initial release used 120 high-severity vulnerabilities from 40 audits, mostly drawn from [Code4rena](https://code4rena.com/), and separated the evaluation into three modes:

- **Detect:** review a repository and report the known high-severity vulnerabilities.
- **Patch:** modify vulnerable code while preserving intended behavior and preventing the known exploit.
- **Exploit:** interact with contracts on a local chain and produce the transactions needed to demonstrate impact.

The initial paper reported that the best agent detected 45.6% of the vulnerabilities. The best patch score was 41.5%, and the best exploit score was 72.2%. These percentages should not be compared as if they came from identical tasks. Detection covered all 120 vulnerabilities, while the authors manually configured 45 vulnerabilities for patching and 24 for exploitation. The exploit subset was smaller and selected for reproducible local deployment.

The paper also tested hints. When agents received information about the broken mechanism, patch and exploit performance increased. This supports the view that finding the relevant mechanism inside a large repository is a major difficulty. It does not prove that discovery is the only bottleneck, or that the harness has little effect on discovery.

A subsequent [independent re-evaluation of EVMbench](https://arxiv.org/abs/2603.10795) tested 26 configurations across four model families and three scaffolds. It also introduced 22 incidents that occurred after the evaluated models' release dates to reduce the risk that benchmark cases had appeared in training data. The researchers found that rankings changed across configurations and datasets. An open-source scaffold outperformed vendor alternatives by 1.7 to 5 percentage points in some comparisons. On the post-release incident set, agents detected up to 65% of vulnerabilities, but none completed an end-to-end exploit across the 110 agent-incident pairs.

Do not read these results as evidence that harnesses are irrelevant to smart contract security. Model, scaffold, reasoning settings, task construction, and dataset all affect the result. A strong score on historical contest findings is also not proof that a tool can audit a new protocol autonomously.

## Why Smart Contract Auditing Is a Different Task

A normal coding task usually has an explicit objective: implement a function, repair a failing test, or produce an artifact that a grader can check. An audit starts without a complete list of what is wrong. The reviewer must infer intended behavior, identify valuable assets and privileged roles, build a threat model, and question assumptions that may not appear in the code.

Many important smart contract failures are cross-functional. A local calculation may be correct while the protocol-level accounting is wrong. An oracle can return the specified value while the asset is not liquid enough to support that valuation. An access check can work as written while governance can change the protected parameter too quickly. A bridge can validate a message correctly while its finality assumption is unsafe.

This makes detection sensitive to context that a benchmark may not include: design documents, deployment configuration, off-chain services, governance processes, expected asset behavior, and economic conditions. It also makes false positives expensive. A tool that reports every unusual external call may achieve broad recall but consume more engineering time than it saves.

Patch and exploit tasks begin with more structure. A failing test, known mechanism, target contract, or executable environment provides feedback. Harness engineering is especially useful in that setting because it can sequence tools, preserve state, retry failed transactions, and require the agent to test its conclusion.

## Two Types of Harness Components

For purchasing and architecture decisions, it is useful to split harness components by purpose rather than by implementation.

![Performance scaffolding and verification infrastructure in AI smart contract audits](https://raw.githubusercontent.com/SCAuditStudio/SCASArticles/main/images/ai-agent-harness-security.svg)

**Performance scaffolding** tries to help the model produce a better answer. Examples include prompts, context retrieval, repository maps, decomposition rules, agent roles, memory, retries, and voting between model outputs.

**Verification and control infrastructure** checks the answer or limits the system's authority. Examples include compilers, deterministic tests, fuzzers, invariant suites, static analysis, formal specifications, transaction simulation, sandboxes, permission boundaries, evidence logs, and human approval.

One component can serve both purposes. A test runner gives the model feedback, improving its next attempt, while also providing independent evidence that a patch does not break tested behavior. The important question is not what the component is called. It is whether its value depends on the model continuing to have a particular weakness.

## What May Lose Value as Models Improve

Some harness techniques are direct workarounds for a current limitation:

- splitting every contract into small chunks because the model cannot navigate the full repository;
- repeating a generic vulnerability checklist in every prompt because the model does not apply it reliably;
- retrying the same task several times because outputs are unstable;
- using several agents to vote on findings without adding new evidence; and
- maintaining model-specific prompt wording that becomes unnecessary after a model update.

If the underlying weakness improves, these techniques can become less important or disappear. This is the sense in which part of a harness "decays." The claim is about a component's source of value, not about an entire product becoming obsolete.

There are also reasons to retain similar mechanisms. Context selection can control token cost even when a model can read the entire repository. Decomposition can create reviewable work units. Multiple agents can represent different threat models rather than simply vote. Retries can recover from network or tool failures unrelated to model intelligence. Prompts can encode a firm's review policy or the protocol's specific assumptions.

A team should therefore ask why each layer exists. "The current model forgets this instruction" describes a temporary compensation. "We need a documented review of every privileged state transition" describes a process requirement that can survive model changes.

## What Does Not Lose Its Purpose

Independent verification does not assume that the model is weak. It assumes that high-impact conclusions need evidence.

An invariant test can reject a state transition that violates an accounting rule. A fuzzer can search input sequences the model did not propose. Static analysis can apply a deterministic rule across every relevant path. Formal verification can prove a stated property within a defined model. A transaction simulation can show the exact balance and storage changes produced by an exploit. Human review can question whether the property, environment, and economic assumptions were correct in the first place.

These methods have limitations. A test only covers what it asserts. A fuzzer depends on its input generation and state setup. Static analysis can produce false positives. A formal proof says nothing about an important property that was never specified. A simulated exploit may omit production conditions. Human reviewers can also miss issues. The point is not that one check creates certainty. Different checks fail in different ways, so they provide stronger evidence when combined.

For a deeper explanation of how generated properties should be reviewed and mechanically checked, see our article on [AI and formal verification in DeFi](https://scauditstudio.com/blog/AI-and-Formal-Verification-In-Defi).

Permissions and accountability also remain necessary. An audit agent does not need unrestricted production credentials to analyze code. Exploit generation should run in an isolated environment. Findings should retain the affected code, assumptions, reproduction steps, tool output, and reviewer decision. A more capable agent increases the importance of these controls because it can take more consequential actions when given broad access.

## How to Evaluate an AI Smart Contract Audit Tool

A useful evaluation starts with the protocol's intended use. A team seeking a quick pre-commit scan needs a different system from an audit firm investigating economic invariants across a lending protocol. Define whether the tool is expected to find common implementation bugs, review a pull request, generate tests, validate known findings, or support a full manual audit.

Then ask for evidence in these areas:

| Question | What useful evidence looks like |
|---|---|
| Was the model held constant? | A comparison that changes one major variable at a time, rather than crediting the harness for a model upgrade. |
| Is the benchmark version identified? | Dataset version, task count, model version, scaffold, reasoning setting, run count, and evaluation date. |
| Was the code held out? | Private or post-training cases that reduce memorization risk, plus a clear policy against training on the evaluation set. |
| Are false positives measured? | Precision, triage time, duplicate rate, and the number of invalid high-severity reports—not recall alone. |
| Can findings be reproduced? | A minimal test, transaction sequence, trace, or state diff that another reviewer can run. |
| Does the tool understand scope? | Evidence that it maps assets, trust assumptions, roles, external dependencies, upgrades, and off-chain components. |
| Which checks are independent? | Fuzzing, invariants, static analysis, formal methods, or simulation whose result does not depend only on another model judgment. |
| What authority does the agent have? | Sandboxed execution, least-privilege tools, secret handling, network restrictions, and approval gates. |
| What does a run cost? | Token, compute, license, setup, and human-triage cost for a representative repository. |
| How are fixes reviewed? | Regression tests, patch diff review, reruns, unresolved assumptions, and human sign-off. |

Run a pilot on code whose known issues are hidden from the vendor or internal tool team. Include at least one protocol-specific logic error, not only standard reentrancy or access-control examples. Record valid findings, missed findings, false positives, time to triage, and whether the evidence was sufficient to fix the problem.

An AI scan can also be part of audit preparation rather than a substitute for the audit. Our guide to [preparing a codebase for a smart contract audit](https://scauditstudio.com/blog/Maximizing-the-Value-of-Your-Smart-Contract-Audit) explains the documentation, scope, and testing work that should exist before external review. Teams comparing automation cost with a formal engagement can use the [smart contract audit pricing guide](https://scauditstudio.com/blog/Why-Some-Smart-Contract-Audits-Are-So-Expensive-%282025-Edition%29) for broader cost context.

## Limits of the Current Evidence

Benchmarks are necessary, but each one samples a specific task distribution. Historical audit findings may appear in model training data. Contest repositories are usually cleaner and better documented than an unfamiliar production incident. Exploit subsets may select vulnerabilities that are practical to deploy in a local environment. A model-based judge can measure whether a known finding was mentioned, but it may not measure the operational cost of invalid reports.

Leaderboard positions also change. A rank reported on publication day is not a permanent product attribute. Scores can move with the model version, scaffold, reasoning budget, tool permissions, task timeout, and number of attempts.

Treat each benchmark result as comparative and bounded: one configuration performed better than another on that benchmark under the reported conditions. Before accepting claims about autonomous auditing, look for additional evidence from held-out code, realistic environments, false-positive measurement, and human review.

## Conclusion

AI agent harnesses matter for smart contract auditing. They determine what the model sees, which tools it can use, how it responds to failures, and what evidence reaches the reviewer. Current research does not support reducing performance to either the model or the harness alone.

The durable distinction is purpose. Components that only compensate for a named model weakness may lose value when that weakness improves. Components that verify results, enforce permissions, preserve evidence, or assign accountability continue to solve a real security problem.

For teams evaluating AI audit tools, the practical standard is straightforward: test on relevant held-out code, require reproducible evidence, measure false positives as well as recall, and keep independent checks and human judgment in the release process.

## About Us

At SC Audit Studio, we specialize in protocol security assessments. Our team of experts has worked with companies like Aave, 1inch, and many more to conduct thorough security assessments across EVM and non-EVM environments.

[Reach out to us](https://scauditstudio.com/contact) for queries and security assessments!

## Tags

["AI", "Security", "Smart-Contract-Audits", "Agent-Harnesses", "Web3", "Formal-Verification"]

## FAQ

[
  {
    "question": "What is an AI agent harness for smart contract auditing?",
    "answer": "An AI agent harness is the software around the model. It selects repository context, provides instructions and tools, manages retries and memory, limits permissions, records traces, and can run verification steps such as tests, fuzzing, static analysis, or transaction simulation."
  },
  {
    "question": "Will better AI models make agent harnesses obsolete?",
    "answer": "No. Some model-specific workarounds may become unnecessary, but products still need context management, tool integration, permissions, reproducibility, and verification. The likely change is that temporary performance scaffolding becomes thinner while control and verification infrastructure remains."
  },
  {
    "question": "Can an AI tool replace a human smart contract auditor?",
    "answer": "Current benchmark evidence does not support autonomous replacement. AI tools can help with coverage, test generation, known vulnerability patterns, patching, and reproduction. Human reviewers are still needed to define protocol intent, examine economic and operational assumptions, judge ambiguous findings, and accept responsibility for the final assessment."
  },
  {
    "question": "How should a team compare AI smart contract audit tools?",
    "answer": "Use representative held-out code and measure valid findings, missed findings, false positives, triage time, reproducibility, run cost, and permission boundaries. Record the model, scaffold, reasoning settings, benchmark version, and number of attempts so results can be compared fairly."
  },
  {
    "question": "What is the difference between performance scaffolding and verification infrastructure?",
    "answer": "Performance scaffolding helps the model produce a better answer through prompts, context retrieval, decomposition, retries, or agent roles. Verification infrastructure checks the answer or limits its impact through tests, fuzzing, static analysis, formal verification, simulation, sandboxes, evidence logs, and human approval."
  }
]
