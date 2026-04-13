# Launching a Token on Canton Network: A Beginner's Guide

A walkthrough covering how to think about token contracts in Daml, how to structure your files, write your first token, test it locally, and upload it to Canton, no blockchain experience required.

## Table of Contents

1. [What Are We Actually Building?](#what-are-we-actually-building)
2. [How to Think About Daml Tokens](#how-to-think-about-daml-tokens)
3. [How to Split Your Files Correctly](#how-to-split-your-files-correctly)
4. [Step 1: Install the Tools](#step-1-install-the-tools)
5. [Step 2: Create Your Project](#step-2-create-your-project)
6. [Step 3: Write the Token Contract](#step-3-write-the-token-contract)
7. [Step 4: Write the Mint Request Contract](#step-4-write-the-mint-request-contract)
8. [Step 5: Write the Test Script](#step-5-write-the-test-script)
9. [Step 6: Build and Test Locally](#step-6-build-and-test-locally)
10. [Step 7: Upload to Canton](#step-7-upload-to-canton)
11. [Common Mistakes](#common-mistakes)
12. [Summary](#summary)
13. [About Us](#about-us)
14. [Tags](#tags)
15. [FAQ](#faq)

## What Are We Actually Building?

A **token** on Canton is just a Daml smart contract that says: *"this party owns this amount of this asset."* That's it.

Unlike Ethereum where a token is a global ledger entry visible to everyone, a Canton token is a **private contract** between specific parties. Only the people named in the contract can see it. The Canton Network enforces this automatically, you don't have to write any privacy logic yourself.

By the end of this guide you will have:

- A `Token` contract (the actual token holding)
- A `MintRequest` contract (how new tokens get created safely)
- A test script that proves it all works
- Your compiled contract uploaded to a local Canton node



## How to Think About Daml Tokens

Before writing a single line of code, it helps to understand the core mental model.

**Everything is a contract.** A token is not a number in a database. It is a contract on the ledger that says: issuer X created this, owner Y holds it, and the amount is Z. When Alice sends tokens to Bob, the old contract is *archived* (deleted) and a new one is *created* with Bob as the owner. There is no "update", only archive and create.

**Someone has to be in charge.** In Daml, every contract has a `signatory`, the party whose authority backs the contract's existence. For tokens, the `issuer` is the signatory. This means only the issuer can create token contracts. Owners can *exercise choices* on their tokens (transfer, burn, split) but they cannot forge new ones from thin air.

**The issuer signs, the owner acts.** Think of it like a banknote. The central bank (issuer) prints it and guarantees its value. You (owner) can spend it, give it away, or destroy it. But you cannot print more.

```
Issuer                      Owner
  |                           |
  | -- creates Token -------> |  (issuer is signatory)
  |                           |
  |                           | -- Transfer --> New Owner
  |                           | -- Burn     --> (destroyed)
  |                           | -- Split    --> Two tokens
```

Keep this mental model in mind as you read the code below and it will all make sense.



## How to Split Your Files Correctly

This trips up beginners more than anything else. Here is the rule:

**One module per file. The file name must match the module name.**

If your module is called `Token`, the file must be called `Token.daml`. If your module is called `Token.Mint`, the file must be at `daml/Token/Mint.daml`. Daml treats dots as directory separators.

For a simple token project, the recommended structure is:

```
my-token/
├── daml.yaml               ← project config (SDK version, dependencies)
├── daml/
│   ├── Token.daml          ← the Token template
│   ├── MintRequest.daml    ← the MintRequest template  
│   └── Token/
│       └── Test.daml       ← test scripts (module: Token.Test)
```

**Why split at all?** Because test scripts import `Daml.Script`, which is a dev dependency. Keeping test code in a separate file means your production `.dar` file stays clean and your contract logic stays readable. As your project grows, separating concerns (token logic, minting logic, tests) makes maintenance far easier.

For this beginner guide, we will keep everything in two files: `Token.daml` for the contracts, and `Token/Test.daml` for the tests.



## Step 1: Install the Tools

You need Java 17+ and the Digital Asset Package Manager (`dpm`).

```bash
# Install Java
sudo apt update && sudo apt install openjdk-21-jdk -y

# Install dpm
curl https://get.digitalasset.com/install/install.sh | sh

# Add dpm to your PATH
echo 'export PATH="/home/$USER/.dpm/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Verify both work
java -version
dpm --version
```



## Step 2: Create Your Project

```bash
dpm new my-token
cd my-token
```

This creates a `daml.yaml` config file and a `daml/` folder. Open `daml.yaml` and make sure it looks like this:

```yaml
sdk-version: 3.4.0        # run `dpm version` and use your installed version
name: my-token
version: 1.0.0
source: daml
dependencies:
  - daml-prim
  - daml-stdlib
  - daml-script
```

Now create the folder for the test file:

```bash
mkdir -p daml/Token
```

Your project is ready.



## Step 3: Write the Token Contract

Create `daml/Token.daml` with the following content:

```haskell
module Token where

-- A token on Canton. Represents ownership of an amount of some asset.
template Token
  with
    issuer : Party    -- the authority backing this token
    owner  : Party    -- who currently holds it
    amount : Decimal  -- how much
    symbol : Text     -- e.g. "MYT" (My Token)
  where
    signatory issuer  -- only the issuer authorizes token creation
    observer  owner   -- the owner can see their token

    -- Transfer to a new owner. Only the current owner can do this.
    choice Transfer : ContractId Token
      with newOwner : Party
      controller owner
      do create this with owner = newOwner

    -- Destroy the token entirely.
    choice Burn : ()
      controller owner
      do return ()

    -- Split into two tokens. Useful for partial payments.
    choice Split : (ContractId Token, ContractId Token)
      with splitAmount : Decimal
      controller owner
      do
        assert (splitAmount > 0.0 && splitAmount < amount)
        t1 <- create this with amount = splitAmount
        t2 <- create this with amount = amount - splitAmount
        return (t1, t2)
```

**Key decisions explained:**

`signatory issuer`, the issuer is the sole authority. Without this, anyone could create tokens and claim they're backed by your issuer. With it, only a transaction that carries the issuer's signature can create a `Token` contract.

`observer owner`, the owner does not co-sign creation (which would block transfers, since new owners haven't agreed to anything yet). They are an observer: they can see the contract and exercise choices on it, but they don't sign it into existence.

`controller owner` on all choices, the owner is in full control of what happens to their token. The issuer has no power to burn or transfer an owner's tokens without their consent.



## Step 4: Write the Mint Request Contract

Create `daml/MintRequest.daml`:

```haskell
module MintRequest where

import Token (Token(..))

-- A request from a user to the issuer to mint tokens for them.
-- This two-step pattern means minting always requires the issuer's approval.
template MintRequest
  with
    issuer : Party
    owner  : Party
    amount : Decimal
    symbol : Text
  where
    signatory owner   -- the requester backs this request
    observer  issuer  -- the issuer can see and act on it

    -- Issuer approves: the token gets created.
    choice Approve : ContractId Token
      controller issuer
      do
        create Token with
          issuer = issuer
          owner  = owner
          amount = amount
          symbol = symbol

    -- Requester changes their mind.
    choice Cancel : ()
      controller owner
      do return ()
```

**Why a separate `MintRequest`?** Why not just let the issuer create tokens directly?

In a real application the issuer is a backend service, not a human sitting at a keyboard. The `MintRequest` pattern lets the *user* initiate the process (which they can do from a frontend) while ensuring the *issuer* always approves before any tokens exist. This is the standard **Propose and Accept** pattern in Daml, and it maps cleanly to real-world authorization workflows.



## Step 5: Write the Test Script

Create `daml/Token/Test.daml`:

```haskell
module Token.Test where

import Daml.Script
import Token
import MintRequest

tokenTest : Script ()
tokenTest = script do

  -- Set up three parties
  issuer <- allocateParty "Issuer"
  alice  <- allocateParty "Alice"
  bob    <- allocateParty "Bob"

  -- Alice asks the issuer to mint her 1000 tokens
  mintReqId <- submit alice do
    createCmd MintRequest with
      issuer = issuer
      owner  = alice
      amount = 1000.0
      symbol = "MYT"

  -- Issuer approves, tokens now exist
  tokenId <- submit issuer do
    exerciseCmd mintReqId Approve

  -- Alice splits off 300 tokens
  (small, _large) <- submit alice do
    exerciseCmd tokenId Split with splitAmount = 300.0

  -- Alice sends the 300-token slice to Bob
  bobTokenId <- submit alice do
    exerciseCmd small Transfer with newOwner = bob

  -- Bob burns his tokens
  submit bob do
    exerciseCmd bobTokenId Burn

  -- Nobody can forge tokens by pretending to be the issuer
  submitMustFail alice do
    createCmd Token with
      issuer = issuer   -- alice submits but can't carry issuer's signature
      owner  = alice
      amount = 9999.0
      symbol = "FAKE"

  return ()
```

The test covers the full lifecycle: mint → split → transfer → burn, plus a security check that forgery fails.



## Step 6: Build and Test Locally

Compile the project:

```bash
dpm build
```

A successful build prints something like:

```
Created .daml/dist/my-token-1.0.0.dar
```

Run the tests:

```bash
dpm test
```

You should see:

```
daml/Token/Test.daml:tokenTest: ok
```

You can also open Daml Studio to see a visual ledger view of the test:

```bash
dpm studio
```

In VS Code, click the **"Script Results"** link that appears above `tokenTest` in the editor. It shows you a table of every contract created, who can see it, and which ones are archived, a live visualization of your token's lifecycle.



## Step 7: Upload to Canton

Start a local Canton sandbox (simulates a real Canton node on your machine):

```bash
dpm sandbox
```

In a second terminal, upload your compiled `.dar` file:

```bash
curl -X POST http://localhost:7575/v2/packages \
  -H "Content-Type: application/octet-stream" \
  --data-binary @.daml/dist/my-token-1.0.0.dar
```

A `200 OK` response means your token contract is now live on the local Canton node. Any application with access to this node can now create `MintRequest` contracts, have the issuer approve them, and start transferring tokens between parties.

For deployment to **DevNet, TestNet, or MainNet**, follow the full validator node setup (spinning up a Canton participant node, getting sponsored, and uploading your `.dar` via the Ledger API). The upload command is the same, only the endpoint URL changes.



## Common Mistakes

**Putting `signatory issuer, owner` on the Token.** This requires the new owner's signature whenever a token is created, including during transfers. Since the new owner isn't part of the transaction yet, the transfer fails. Use `signatory issuer` and `observer owner`.

**Module name doesn't match file name.** If your file is `daml/Token.daml` but the first line says `module Tokens where`, Daml will refuse to compile. They must match exactly.

**Forgetting to import.** If `Token/Test.daml` uses `Token` and `MintRequest` templates, it must `import Token` and `import MintRequest` at the top. Daml does not auto-import anything.

**Reusing the same `.dar` across networks.** Each Canton network (DevNet, TestNet, MainNet) is fully isolated. Upload your `.dar` separately to each one. The same file works everywhere, the endpoint URL is the only thing that changes.

**Using `=` instead of `<-` for ledger actions.** In scripts, `<-` means "run this action and give me the result." `=` is for pure values. `tokenId <- submit ...` is correct. `tokenId = submit ...` will not compile.



## Summary

A Canton token is a private Daml contract with an `issuer` (the authority) and an `owner` (the holder). The issuer signs tokens into existence; the owner controls what happens to them via `Transfer`, `Burn`, and `Split` choices.

Split your code into focused files: one for the token template, one for the mint request template, one for tests. Keep test code separate from production code so your `.dar` stays clean.

The workflow is: write → `dpm build` → `dpm test` → `dpm sandbox` → upload `.dar`. Once your `.dar` is on a Canton node, your token is live.

**Daml Documentation:** [docs.digitalasset.com/build/3.4](https://docs.digitalasset.com/build/3.4)

**Canton Token Standard (CIP-0056):** [github.com/hyperledger-labs/splice/tree/main/token-standard](https://github.com/hyperledger-labs/splice/tree/main/token-standard)

**Validator Setup (next step):** [docs.dev.sync.global](https://docs.dev.sync.global)


## About Us

At SC Audit Studio, we specialize in protocol security assessments. Our team of experts has worked with companies like Aave, 1inch, and many more to conduct thorough security assessments across EVM and non-EVM environments.

[Reach out to us](https://scauditstudio.com/contact) for queries and security assessments!



## Tags

["Canton", "Daml", "Token", "Smart Contracts", "Beginner Guide", "Development"]



## FAQ

[
  {
    "question": "Do I need Canton Coin to launch a token?",
    "answer": "Not to develop and test locally. You only need CC when submitting transactions to a live Canton node (DevNet, TestNet, MainNet), where CC pays for network traffic fees."
  },
  {
    "question": "Can anyone see my token balances?",
    "answer": "No. Canton's privacy model ensures only the parties named in a contract (`signatory`, `observer`, `controller`) can see it. Nobody else on the network has visibility into your token holdings."
  },
  {
    "question": "What is the difference between `signatory` and `observer`?",
    "answer": "A signatory's authority is required to create or archive the contract, they are legally and cryptographically responsible for it. An observer can see the contract and exercise choices on it, but their signature is not needed to bring it into existence."
  },
  {
    "question": "Can I add more choices later, like a merge?",
    "answer": "Yes. Adding a `Merge` choice that archives two token contracts and creates one combined contract is straightforward. Canton also supports smart contract upgrades, so you can evolve your token logic over time without losing existing contract state."
  },
  {
    "question": "What is a `.dar` file?",
    "answer": "A Daml Archive. It is a compiled, packaged version of your Daml code, like a `.jar` for Java. It contains your contract logic plus all dependencies, and it is the artifact you upload to a Canton node to make your contracts available on the ledger."
  }
]
