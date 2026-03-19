---
name: elytro
description: >
  Elytro — ERC-4337 smart account wallet CLI for AI agents. On-chain 2FA, spending limits,
  OS keychain vault (macOS/Windows/Linux). Deferred OTP: commands exit with otp_pending;
  complete with `otp submit <id> <code>`. Send ETH/ERC-20, batch tx, gas sponsorship.
  Use when: managing smart accounts, sending transactions, binding email, setting limits,
  or any wallet operation on Ethereum, Optimism, Arbitrum, Base. Combine with defi/uniswap
  for swaps. Node >= 24.
version: 0.6.0
homepage: https://elytro.com
metadata:
  openclaw:
    requires:
      bins:
        - elytro
      node: ">=24.0.0"
    emoji: "🔐"
    homepage: https://github.com/Elytro-eth/skills
    os: ["macos", "windows", "linux"]
    install:
      - id: npm
        kind: npm
        package: "@elytro/cli"
        bins: ["elytro"]
        label: "Install Elytro CLI (npm)"
---

# Elytro CLI — Agent Skill

Operate the Elytro smart account wallet. Every command returns JSON to stdout. Parse `success` and `result` — never regex free-form text.

Install: `npm install -g @elytro/cli` (Node >= 24)

**Command reference**: [references/commands.md](references/commands.md) — read when needing exact syntax, options, or return shapes.

---

## Rules

1. **Update check at session start.** Run `elytro update check`; inform if `updateAvailable`. Do not auto-upgrade.
2. **Never guess on-chain data.** Query via `elytro query`.
3. **Never auto-confirm.** On `(y/N)`, STOP and present choice. Wait for approval.
4. **OTP is deferred.** Commands requiring OTP exit with `status: "otp_pending"`. Parse `result.otpPending.id` and `result.otpPending.submitCommand`. Instruct user to check email and run `elytro otp submit <id> <code>`. Do not block.
5. **Chains**: 1, 10, 42161, 8453, 11155111, 11155420. `value` in ETH. `data` hex with `0x`.
6. **Deploy before tx.** `account info` → `deployed: true`. Else `account activate`.
7. **Security required.** Never `tx send` without `hookInstalled` AND `emailVerified` (from `security status`).
8. **Create with `--email` and `--daily-limit`.** Order: create → activate → email bind → spending-limit.
9. **Always pass alias/address to `account switch`.** No interactive selector.
10. **Parse JSON from every command.** stderr = spinners/prompts; stdout = JSON.

---

## Agent Communication

**Template**: `<Status> — <Key details> — <Explorer/next step>`

Keep compact. Never dump raw JSON unless asked.

### Formats

| Operation | Format |
|-----------|--------|
| Tx sent | `✅ Sent 0.05 ETH to 0xAbc…1234. Tx: 0xdef… Explorer: <url>` |
| Simulated | `🧪 Simulation passed — gas 0.00015 ETH. No warnings.` |
| Balance | `💰 agent-primary: 0.482 ETH` |
| OTP pending | `🔐 OTP sent to u***@example.com. Run: elytro otp submit <id> <code>` |
| Error | `❌ <description> (code -32xxx). → <fix>` |

**Principles**: Lead with outcome. Surface explorer links. Translate codes to plain language. Flag security gaps immediately. Never show raw JSON unless asked.

---

## Account Lifecycle

```
create → activate → email bind + spending-limit → PROTECTED
```

| State | Verify | Safe to tx? |
|-------|--------|:-----------:|
| CREATED | `deployed: false` | No |
| DEPLOYED | `deployed: true`, `emailVerified: false` | No |
| PROTECTED | `hookInstalled`, `emailVerified`, `dailyLimitUsd` | **Yes** |

---

## First-Time Setup

```bash
elytro init
elytro account create --chain 11155420 --alias agent-primary --email u@x.com --daily-limit 100
elytro account activate agent-primary   # CHECK: hookInstalled MUST be true
elytro security email bind u@x.com      # → otp_pending; user runs otp submit <id> <code>
elytro security spending-limit 100      # → otp_pending; user runs otp submit <id> <code>
elytro security status                  # Verify: hookInstalled, emailVerified, dailyLimitUsd
```

Before `tx send`: (1) deployed, (2) hookInstalled, (3) emailVerified, (4) dailyLimitUsd set, (5) sufficient balance.

---

## Workflow Patterns

**Simulate → Send**
```bash
elytro tx simulate --tx "to:0xAddr,value:0.5"   # parse warnings
elytro tx send --tx "to:0xAddr,value:0.5"
```

**Deferred OTP** (email bind, spending-limit, tx send when limit exceeded, 2fa uninstall)
```bash
elytro security email bind u@x.com
# Parse result.otpPending.id, result.otpPending.submitCommand
# User checks email → elytro otp submit <id> <code>
```

**Batch**
```bash
elytro tx send --tx "to:0xA,value:0.01" --tx "to:0xB,value:0.02"
```

**Token swap** (with defi/uniswap): Get calldata from uniswap → `tx simulate` → `tx send`.

---

## Error Recovery

| Error | Fix |
|-------|-----|
| Wallet not initialized | `elytro init` |
| Keyring locked / Vault key not found | Check OS credential store |
| Account not deployed | `elytro account activate` |
| Insufficient balance | Fund account |
| `hookInstalled: false` | `elytro security 2fa install` |
| Chain N not supported | Use valid chain ID from error |
| Alias already taken | Choose different alias |
| Challenge does not belong to session | Same account for `otp submit`; re-run original if expired |
| Unknown OTP id | `otp list`; re-run original command |
| AA21 in error | Balance or nonce issue; check simulation |
