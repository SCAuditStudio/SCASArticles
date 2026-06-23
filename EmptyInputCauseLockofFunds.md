# How a Zero-Amount Call Could Lock Any LP's Funds in Perpetuals Protocols

Between April 14 and May 4, 2026, we ran a pre-audit consultation on Plether Perpetuals a forex-indexed perps protocol on Arbitrum built around a USDC-denominated liquidity pool. During the review, we found two High and seven Medium severity issues. This article walks through one of the more interesting Highs: a zero-amount griefing vector in the TrancheVault that let any external address reset any LP's withdrawal cooldown, indefinitely, for free.

## The Protocol Context

Plether's liquidity lives in a contract called `HousePool`, which splits LP capital across two tranches senior and junior, using a pair of ERC-4626-compatible `TrancheVault` contracts. Junior tranche absorbs first losses in exchange for higher upside. Senior tranche earns relatively stable yield. Standard tranched LP mechanics.

To prevent flash-loan-style attacks on pool liquidity, the vault enforces a `DEPOSIT_COOLDOWN`, a time-lock between when you deposit and when you're allowed to withdraw. This is tracked per LP via a `lastDepositTime` mapping:

```solidity
mapping(address => uint256) public lastDepositTime;
```

Once `block.timestamp >= lastDepositTime[owner] + DEPOSIT_COOLDOWN`, the LP is free to withdraw. Straightforward protection. The bug had nothing to do with the cooldown logic itself, it was in how `_withdraw()` reset it.

## What We Noticed

When reviewing `TrancheVault.sol`, we followed the path that `withdraw()` and `redeem()` take through OpenZeppelin's ERC-4626:

```solidity
function withdraw(uint256 assets, address receiver, address _owner)
    public override returns (uint256) {
    POOL.reconcile();
    ...
    return super.withdraw(assets, receiver, _owner);
}

function redeem(uint256 shares, address receiver, address _owner)
    public override returns (uint256) {
    POOL.reconcile();
    ...
    return super.redeem(shares, receiver, _owner);
}
```

Both route into the vault's `_withdraw()` hook. That hook is where the cooldown check and reset live:

```solidity
function _withdraw(
    address caller,
    address receiver,
    address _owner,
    uint256 assets,
    uint256 shares
) internal override {
    if (block.timestamp < lastDepositTime[_owner] + DEPOSIT_COOLDOWN) {
        revert TrancheVault__DepositCooldown();
    }
    lastDepositTime[_owner] = block.timestamp; // ← state mutation
    if (caller != _owner) {
        _spendAllowance(_owner, caller, shares);
    }
    _burn(_owner, shares);
    ...
}
```

The ordering here is what caught our eye. The `lastDepositTime` reset happens *before* `_spendAllowance`. That on its own isn't exploitable, if `shares > 0`, an unauthorized caller would hit the allowance check and revert. But what happens when `shares == 0`?

## Pulling the Thread

OpenZeppelin's ERC4626 entry point guards withdrawals with a max-check:

```solidity
if (assets > maxAssets) {
    revert ERC4626ExceededMaxWithdraw(owner, assets, maxAssets);
}
```

When `assets == 0`, this condition is `0 > maxAssets`, which is never true, even if `maxAssets == 0` because the LP is still in cooldown. The call passes. Same logic for `redeem(0, ...)` with the shares check.

Then in `_withdraw()`, `shares == 0`. OpenZeppelin's allowance spending only reverts when the caller's allowance is less than the spend amount:

```solidity
if (currentAllowance < value) {
    revert ERC20InsufficientAllowance(spender, currentAllowance, value);
}
```

When `value == 0`, that's `currentAllowance < 0`, impossible. The call goes through with zero approval from the victim.

So the full path for an unauthorized caller with zero approval:

1. OZ ERC4626 max-check passes, `0 > maxAssets` is always false
2. `_withdraw()` passes the cooldown check, cooldown had already expired
3. `lastDepositTime[_owner] = block.timestamp`, cooldown reset, state mutated
4. `_spendAllowance(_owner, caller, 0)`, no-op, no approval needed
5. `_burn(_owner, 0)`, no-op
6. Zero assets transferred

The victim's `lastDepositTime` is now `block.timestamp`. Their cooldown just restarted. The attacker spent gas. That's the entire cost.

## The Attack

Here's the full griefing flow:

**Step 1**, Victim deposits into the junior or senior TrancheVault. `lastDepositTime[victim]` is set.

**Step 2**, Victim waits out `DEPOSIT_COOLDOWN`. `maxWithdraw(victim)` now returns their balance. They're ready to exit.

**Step 3**, Attacker calls:

```solidity
juniorVault.withdraw(0, attacker, victim);
// or
seniorVault.redeem(0, attacker, victim);
```

**Step 4**, `_withdraw()` runs. Cooldown check passes (it had expired). `lastDepositTime[victim]` gets set to `block.timestamp`. Zero shares burned, zero assets moved.

**Step 5**, `maxWithdraw(victim)` returns `0`. `maxRedeem(victim)` returns `0`. Victim is locked again for the full cooldown period.

**Step 6**, Each time the cooldown expires, the attacker repeats. Gas-only cost per cycle. No shares needed, no approval needed, no capital at risk.

In a perpetuals protocol where LPs might need to exit quickly during adverse market conditions, indefinite cooldown griefing has direct financial consequences. The attacker doesn't need to profit directly, this is a targeted denial of withdrawal.

## Why This Matters in Plether's Context

Plether's LP tranches are the counterparty to every trade in the system. The senior tranche in particular is designed as a relatively stable yield vehicle, the kind of position an LP might want to exit fast if trading conditions shift against them.

The cooldown is a reasonable protection against flash-loan abuse. But the zero-amount path bypasses the entire authorization layer and exposes `lastDepositTime` as mutable state with no cost gate. Any address can target any LP on either vault, repeatedly, for as long as they're willing to pay gas.

There's also a related finding (SC-L1) we flagged in the same audit: a zero-value senior withdraw could trigger `reconcile()` in a way that left `HousePool`'s `seniorPrincipal` and `juniorPrincipal` materially overstated, in one PoC, raw principal read `100,000 USDC` while the correct value was `49,960 USDC`. The zero-amount attack surface across the vault was a recurring theme in this codebase.

## The Fix

We recommended rejecting zero-amount inputs at the public entry points, before any state is touched:

```solidity
function withdraw(uint256 assets, address receiver, address owner)
    public override returns (uint256) {
    require(assets > 0, "ZERO_ASSETS");
    POOL.reconcile();
    ...
    return super.withdraw(assets, receiver, owner);
}

function redeem(uint256 shares, address receiver, address owner)
    public override returns (uint256) {
    require(shares > 0, "ZERO_SHARES");
    POOL.reconcile();
    ...
    return super.redeem(shares, receiver, owner);
}
```

A secondary hardening we also noted: moving the `lastDepositTime` write to *after* `_spendAllowance`, consistent with checks-effects-interactions, so that any unauthorized call reverts before state is mutated, regardless of amount:

```solidity
// Before
lastDepositTime[_owner] = block.timestamp;  // ← state first
_spendAllowance(_owner, caller, shares);

// After
_spendAllowance(_owner, caller, shares);    // ← check first
lastDepositTime[_owner] = block.timestamp;
```

Both fixes together close the vector completely.

## Takeaway

ERC-4626 is designed to be extended. Protocols that add stateful mechanisms on top, cooldowns, lockups, epoch-based restrictions, need to think carefully about what zero- amount paths do to those mechanisms. OZ's max-check is a strict inequality: `amount > max`. When `amount == 0`, it always passes, even if `max == 0`. If your `_withdraw()` hook mutates state before allowance checks, that state mutation is reachable by anyone.

The pattern appears in a few other ERC-4626 vaults we've reviewed. It's worth checking if your own vault extension has the same ordering issue before it ends up in production.

## About Us

At SC Audit Studio, we specialize in protocol security assessments. Our team of experts has worked with companies like Aave, 1inch,Li.Fi and many more to conduct thorough security assessments across EVM and non-EVM environments.

Pioneers should not care about cybersecurity, we take care of it.
[Reach out to us](https://scauditstudio.com/contact)

## FAQ
[
{
"question": "Was this a flaw in OpenZeppelin's ERC-4626 or in Plether's vault?",
"answer": "It's Plether's. OZ's implementation is a base to extend, the responsibility for guarding zero-amount entry paths falls on the protocol. OZ's strict-inequality max-check is correct behavior; the missing guard and the premature state mutation are both in TrancheVault."
},
{
"question": "Why does _spendAllowance not block an unauthorized zero-amount call?",
"answer": "OZ's allowance check only reverts when currentAllowance < value. When value == 0, that condition is never true, regardless of what approval the caller has. This is a known property of ERC-20 allowance logic, not a bug in OZ."
},
{
"question": "How much does this attack actually cost?",
"answer": "One transaction per cooldown cycle. No tokens, no shares, no approval needed. Just gas."
},
{
"question": "Does the attacker gain anything beyond griefing?",
"answer": "Not directly. No assets leave the vault and nothing goes to the attacker. The attack's value is purely in denying exit to a target LP, useful for manipulation scenarios where keeping a specific LP locked changes pool dynamics at a critical moment."
},
{
"question": "Was this found in the pre-audit or a formal audit?",
"answer": "This was found during our pre-audit engagement (April–May 2026). The pre-audit is specifically designed to catch design-level and high-severity issues before the formal audit begins, so the formal audit can focus on deeper, more subtle vulnerabilities."
}
]
