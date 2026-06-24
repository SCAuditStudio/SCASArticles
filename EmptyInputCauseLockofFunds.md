# How a Zero-Amount Call Could Lock Any LP's Funds in Perpetuals Protocols

Between April 14 and May 4, 2026, we ran a pre-audit consultation on Plether Perpetuals, a forex-indexed perps protocol on Arbitrum built around a USDC-denominated liquidity pool. During the review, we found two High and seven Medium severity issues. This article covers one of the more interesting Highs: a zero-amount griefing vector in the TrancheVault that let any external address reset an LP's withdrawal cooldown indefinitely and at no cost beyond gas.

## The Protocol Context

Plether's liquidity lives in a contract called `HousePool`, which splits LP capital across senior and junior tranches using a pair of ERC-4626-compatible `TrancheVault` contracts. The junior tranche absorbs first losses in exchange for higher upside, while the senior tranche earns relatively stable yield.

To prevent flash-loan-style attacks on pool liquidity, the vault enforces a `DEPOSIT_COOLDOWN`, a time-lock between when you deposit and when you're allowed to withdraw. This is tracked per LP via a `lastDepositTime` mapping:

```solidity
mapping(address => uint256) public lastDepositTime;
```

Once `block.timestamp >= lastDepositTime[owner] + DEPOSIT_COOLDOWN`, the LP is free to withdraw. The flaw was in how `_withdraw()` reset the cooldown.

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

The `lastDepositTime` reset happens *before* `_spendAllowance`. For any positive share amount, an unauthorized caller hits the allowance check and the transaction reverts. A zero-share call behaves differently.

## Tracing the Zero-Amount Path

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

The victim's `lastDepositTime` is now `block.timestamp`, restarting the cooldown. The attacker pays only the transaction's gas cost.

## The Attack

The attack works as follows:

**Step 1:** The victim deposits into the junior or senior TrancheVault. `lastDepositTime[victim]` is set.

**Step 2:** The victim waits out `DEPOSIT_COOLDOWN`. `maxWithdraw(victim)` now returns their balance. They're ready to exit.

**Step 3:** The attacker calls:

```solidity
juniorVault.withdraw(0, attacker, victim);
// or
seniorVault.redeem(0, attacker, victim);
```

**Step 4:** `_withdraw()` runs. The cooldown check passes because it had expired. `lastDepositTime[victim]` gets set to `block.timestamp`. Zero shares are burned and zero assets are moved.

**Step 5:** `maxWithdraw(victim)` returns `0`. `maxRedeem(victim)` returns `0`. The victim is locked again for the full cooldown period.

**Step 6:** Each time the cooldown expires, the attacker repeats the call. Each cycle costs only gas and requires no shares, approval, or capital.

An LP may need to exit quickly when market conditions turn against the pool. By restarting the cooldown indefinitely, an attacker can block that exit without needing to profit from the attack directly.

## Why This Matters in Plether's Context

Plether's LP tranches are the counterparty to every trade in the system. The senior tranche in particular is designed as a relatively stable yield vehicle, the kind of position an LP might want to exit fast if trading conditions shift against them.

The cooldown protects against flash-loan abuse, but the zero-amount path bypasses authorization and lets any address update `lastDepositTime`. An attacker can repeatedly target any LP in either vault for the cost of gas.

We also reported a related finding (SC-L1): a zero-value senior withdrawal could trigger `reconcile()` and leave `HousePool`'s `seniorPrincipal` and `juniorPrincipal` materially overstated. In one PoC, the raw principal was `100,000 USDC`; the correct value was `49,960 USDC`. Several vault functions handled zero-amount inputs unsafely.

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

We also recommended moving the `lastDepositTime` write to *after* `_spendAllowance`. This ensures that an unauthorized call reverts before the cooldown is updated, regardless of the amount:

```solidity
// Before
lastDepositTime[_owner] = block.timestamp;  // ← state first
_spendAllowance(_owner, caller, shares);

// After
_spendAllowance(_owner, caller, shares);    // ← check first
lastDepositTime[_owner] = block.timestamp;
```

Together, these changes prevent the attack.

## Takeaway

Protocols often extend ERC-4626 with cooldowns, lockups, or epoch-based restrictions. Each extension must account for zero-amount calls. OpenZeppelin's max-check uses the strict inequality `amount > max`, so an amount of zero passes even when the maximum is zero. If an overridden `_withdraw()` mutates state before checking allowance, anyone may be able to reach that mutation.

We have seen the same pattern in other ERC-4626 vaults. Check zero-amount behavior and the order of state changes and authorization checks in every vault extension.

## About Us

At SC Audit Studio, we specialize in protocol security assessments. Our team of experts has worked with companies like Aave, 1inch,Li.Fi and many more to conduct thorough security assessments across EVM and non-EVM environments.

Pioneers should not care about cybersecurity, we take care of it.
[Reach out to us](https://scauditstudio.com/contact)

## FAQ
[
{
"question": "Was this a flaw in OpenZeppelin's ERC-4626 or in Plether's vault?",
"answer": "The flaw is in Plether's vault. OZ provides a base implementation that protocols can extend, and TrancheVault introduced the missing zero-amount guard and premature state mutation. OZ's strict-inequality max-check behaves as designed."
},
{
"question": "Why does _spendAllowance not block an unauthorized zero-amount call?",
"answer": "OZ's allowance check reverts only when currentAllowance < value. When value == 0, that condition is never true, regardless of the caller's approval. This follows standard ERC-20 allowance behavior."
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
"answer": "We found it during the April–May 2026 pre-audit engagement. The pre-audit focuses on design-level and high-severity issues before the formal audit begins, leaving more time in the formal audit for subtler vulnerabilities."
}
]
