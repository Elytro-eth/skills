---
name: elytro
description: >
  Elytro — security-first ERC-4337 smart account wallet CLI for AI agents.
  On-chain 2FA, configurable spending limits, and macOS Keychain-backed vault.
  Send ETH, ERC-20 tokens, and batch transactions via UserOperations on Ethereum,
  Optimism, and Arbitrum. Account abstraction wallet with gas sponsorship,
  counterfactual deployment, social recovery, and guardian management.
  For token swaps, combine this skill with `defi/uniswap` for planning and `defi/elytro` for execution.
  for programmatic consumption by agents and MCP integrations.
homepage: https://elytro.com
metadata:
  openclaw:
    requires:
      bins:
        - elytro
      node: ">=24.0.0"
    emoji: "🔐"
    homepage: https://github.com/Elytro-eth/skills
    install:
      - id: npm
        kind: npm
        package: "@elytro/cli"
        bins: ["elytro"]
        label: "Install Elytro CLI (npm)"
---

# Elytro CLI — Agent Skill

This skill teaches an autonomous agent how to operate the Elytro CLI end-to-end.
Every command returns structured JSON so agents can plan multi-step workflows,
parse output deterministically, and handle errors without human intervention.

- Repo: https://github.com/Elytro-eth/Elytro
- Required install command: `npm install -g @elytro/cli` (requires Node >= 24)

---

## RULES — Read Before Every Action

1. **Check for updates at session start.** Run `elytro update check` and inform the user if `updateAvailable` is `true`. Do not auto-upgrade — ask first.
2. **Never guess on-chain data.** Always run the appropriate `elytro query` command to fetch balances, transaction status, token holdings, etc. Do NOT infer or assume values.
3. **Never auto-confirm interactive prompts.** When any command outputs `(y/N)` or similar, STOP and present the user with a confirmation choice. The agent must wait for explicit user approval before proceeding. Never pipe `y` into stdin.
4. **Chain IDs accept hex or decimal.** Common values: Ethereum mainnet = `1`, Sepolia = `11155111` (`0xaa36a7`), OP Sepolia = `11155420`, Base = `8453`, Arbitrum = `42161`. Both formats work in `--chain`.
5. **`value` in tx specs is always in ETH**, not wei. Example: `value:0.001` means 0.001 ETH.
6. **`data` field must be valid hex.** Use `0x` for plain ETH transfers (no calldata). For contract calls, provide the full hex-encoded calldata with `0x` prefix and even length.
7. **Smart accounts must be deployed before sending transactions.** Check `elytro account info <account>` → `"deployed": true` before sending. If not deployed, run `elytro account activate <account>` first.
8. **Use alias or address interchangeably** wherever `<account>` appears. Both are accepted by all commands.
9. **Always parse structured JSON output.** Every command returns `{ "success": true/false, ... }`. Parse stdout as JSON and check the `success` field. Never regex-match free-form text.
10. **Security is non-negotiable.** Never send transactions from an account without `hookInstalled: true` and `emailVerified: true`. See the Security-First section below.

---

## Installation

**Prerequisites**: Node.js >= 24.0.0

### From npm (recommended)

```bash
npm install -g @elytro/cli
```

Or with other package managers:

```bash
pnpm add -g @elytro/cli
yarn global add @elytro/cli
bun add -g @elytro/cli
```

After installation, the `elytro` binary is available globally:

```bash
elytro --version
elytro --help
```

---

## Security-First: Non-Negotiable Setup Before Any Transaction

**DO NOT send any transaction until the account has a SecurityHook installed with
a verified email. An account without on-chain 2FA is an unprotected hot wallet —
any compromise of the vault key means total, irreversible loss of funds.**

The required sequence before any `tx send` is:

```
1. elytro init
2. elytro account create --chain <id> --email <email> --daily-limit <usd>
3. elytro security email bind <email>   ← user must provide OTP from inbox
4. elytro account activate <alias>      ← deploys wallet + installs SecurityHook atomically
```

**Step 3 is interactive** — it requires a human to check their email and provide
the 6-digit OTP code. The agent MUST pause and wait for this input. Do not skip it.

Only after `activate` succeeds with `"hookInstalled": true` in the result is the
account safe to use. If `hookInstalled` is `false`, stop and alert the user —
the SecurityHook failed to install and the account is unprotected.

**Why this matters**: The SecurityHook enforces on-chain 2FA (email OTP for
high-value transactions) and daily spending limits. Without it, a single leaked
private key can drain the entire account. With it, an attacker also needs access
to the user's email — a fundamentally different threat model.

---

## Output Format (Universal)

**Every command** outputs structured JSON to **stdout**:

```json
{ "success": true, "result": { ... } }
{ "success": false, "error": { "code": -32000, "message": "...", "data": { ... } } }
```

Error codes follow JSON-RPC conventions:

| Code     | Meaning                          |
| -------- | -------------------------------- |
| `-32000` | Internal / generic error         |
| `-32001` | Resource not found               |
| `-32002` | Account not ready / not deployed |
| `-32003` | Sponsorship failed               |
| `-32004` | UserOp build failed              |
| `-32005` | Send failed                      |
| `-32006` | Execution reverted on-chain      |
| `-32007` | Hook authorization failed        |
| `-32010` | Email not bound                  |
| `-32011` | Safety delay not elapsed         |
| `-32012` | OTP verification failed          |
| `-32602` | Invalid parameters               |

**Parsing rule**: Always `JSON.parse(stdout)` and check `result.success`.
Spinners and interactive prompts go to **stderr** — never mix into stdout.

---

## Secret Management

### Vault Key (macOS Keychain — zero configuration)

The vault key is a 256-bit random key that decrypts the local keyring vault.

On macOS, `elytro init` generates the vault key and stores it in the **system
Keychain** automatically. Every subsequent CLI invocation loads it from Keychain
with no env vars, no SecretRef, and no user interaction. This is the only
officially supported platform.

**The user never sees, copies, or configures the vault key.** It is fully managed
by the OS.

#### Security Properties

- **Domain separation**: The encrypted vault (`keyring.json`) lives on disk; the
  decryption key lives in Keychain. Copying `~/.elytro/` to another machine is
  useless without the Keychain entry.
- **OS-level protection**: Keychain is encrypted with the user's login password and
  locked when the user is logged out or the machine is powered off.
- **Zero-fill**: The raw key buffer is zeroed in memory after the keyring is unlocked.

### Non-macOS Fallback (not recommended)

> **Warning**: Running on Windows, Linux or in containers weakens the security model.
> The vault key must be injected as an environment variable (`ELYTRO_VAULT_SECRET`),
> which is exposed to `/proc/PID/environ`, inherited by child processes before
> consume-once scrubbing, and lacks hardware-backed protection. Use at your own risk.
> **Users must be fully briefed on these limitations.**
> On-chain SecurityHook (2FA + spending limits) is strongly recommended as
> compensating control.

If you must run on Non-macOS:

1. `elytro init` returns `vaultSecret` in the JSON result.
2. The agent captures this value and stores it in SecretRef.
3. On subsequent runs, OpenClaw injects it as an env var. The CLI reads it once
   and immediately deletes it from `process.env` (consume-once).

```jsonc
// openclaw.json — only needed on non-macOS
{
  "skills": {
    "entries": {
      "elytro-cli": {
        "env": {
          "ELYTRO_VAULT_SECRET": {
            "source": "env",
            "provider": "default",
            "id": "ELYTRO_VAULT_SECRET",
          },
        },
      },
    },
  },
}
```

### Optional API Keys

These improve reliability but are not required for basic operation:

| Variable             | Purpose                          | Injected Via                                     |
| -------------------- | -------------------------------- | ------------------------------------------------ |
| `ELYTRO_ALCHEMY_KEY` | Alchemy RPC (higher rate limits) | `elytro config set alchemy-key <KEY>` or env var |
| `ELYTRO_PIMLICO_KEY` | Pimlico bundler/paymaster        | `elytro config set pimlico-key <KEY>` or env var |

These can be persisted once via `config set` and don't need re-injection.

---

## First-Time Setup

An agent running against a fresh `~/.elytro/` must execute this sequence exactly once.
**Every step is mandatory. Do not skip or reorder.**

```bash
# Step 1: Initialize wallet
# Vault key is auto-stored in macOS Keychain. No secrets to manage.
elytro init

# Step 2: Create a smart account WITH security enabled
# --email and --daily-limit are NOT optional in practice.
# Omitting them creates an unprotected account — DO NOT DO THIS.
elytro account create --chain 11155111 --alias agent-primary \
  --email user@example.com --daily-limit 100

# Step 3: Bind and verify email (INTERACTIVE — requires human OTP input)
# The user must check their inbox and provide the 6-digit code.
# DO NOT proceed until this succeeds.
elytro security email bind user@example.com

# Step 4: Activate (deploy + install SecurityHook atomically)
# Check result: "hookInstalled" MUST be true. If false, STOP.
elytro account activate agent-primary

# Step 5: (Optional) Set API keys for better reliability
elytro config set alchemy-key "$ELYTRO_ALCHEMY_KEY"
elytro config set pimlico-key "$ELYTRO_PIMLICO_KEY"
```

### Pre-Transaction Safety Checklist

Before executing any `tx send`, the agent MUST verify:

1. `account info` shows `"deployed": true`
2. `account info` shows `"securityStatus": { "hookInstalled": true }`
3. `security status` shows `"hookInstalled": true` and `"profile.emailVerified": true`
4. Account has sufficient balance (sponsor covers gas only, NOT value)

If any check fails, **do not send the transaction**. Alert the user and guide them
through the missing step.

---

## Command Reference

### `elytro init`

```bash
elytro init
```

Generates a 256-bit vault key and an EOA signing key.

**Success result:**

```json
{
  "status": "initialized",
  "dataDir": "~/.elytro",
  "secretProvider": "KeychainProvider",
  "nextStep": "Run `elytro account create --chain <chainId>` ..."
}
```

On non-macOS, result also includes `vaultSecret` (base64) — save it immediately.

If already initialized: `{ "status": "already_initialized", "dataDir": "..." }`.

---

### Account Commands

#### `elytro account create`

```bash
elytro account create -c <chainId> [-a <name>] [-e <email>] [-l <usd>]
```

- `-c, --chain` (required): Numeric chain ID (hex or decimal).
- `-a, --alias` (optional): Human-readable name. Auto-generated if omitted (e.g. `swift-panda`).
- `-e, --email` (**required in practice**): Email for 2FA OTP. Stored as security intent.
- `-l, --daily-limit` (**required in practice**): Daily spending limit in USD.

**Supported chains**: 1 (Ethereum), 10 (Optimism), 42161 (Arbitrum), 11155111 (Sepolia), 11155420 (OP Sepolia).

> **Rule: ALWAYS pass `--email` and `--daily-limit`.** The flags are technically
> optional to allow edge-case testing, but in any real workflow, creating an account
> without them produces an unprotected wallet. If the user has not provided an email
> or spending limit, the agent should ask for them before proceeding.

**Success result:**

```json
{
  "alias": "swift-panda",
  "address": "0x...",
  "chain": "Sepolia",
  "chainId": 11155111,
  "deployed": false,
  "security": {
    "email": "user@example.com",
    "emailBindingStarted": false,
    "dailyLimitUsd": 100,
    "hookPending": true
  }
}
```

When `--email`/`--daily-limit` are provided, a **security intent** is stored locally.
The SecurityHook is installed atomically during `activate`. Intent is temporary
and deleted after activation completes — it exists only to bridge create → activate.

**If `"security": null` in the result**, the account has no security configuration.
The agent should warn the user and suggest re-creating with `--email` and `--daily-limit`.

#### `elytro account activate`

```bash
elytro account activate [alias|address] [--no-sponsor]
```

Deploys the smart contract wallet on-chain. If a security intent exists (from
`--email`/`--daily-limit` at create), batches SecurityHook installation into the
same deploy UserOp. Atomic — if hook install fails, the entire deployment reverts.

**Success result:**

```json
{
  "alias": "swift-panda",
  "address": "0x...",
  "chain": "Sepolia",
  "chainId": 11155111,
  "transactionHash": "0x...",
  "gasCost": "0.0001 ETH",
  "sponsored": true,
  "hookInstalled": true,
  "explorer": "https://sepolia-optimism.etherscan.io/tx/0x..."
}
```

**Critical: Check `hookInstalled` in the result.**

- `true` → SecurityHook is active. Safe to proceed with transactions.
- `false` → Hook installation failed. **DO NOT send transactions.** The account
  is deployed but unprotected. Run `elytro security 2fa install` manually, or
  alert the user.

If already deployed: `{ "status": "already_deployed" }`.

**Important**: Fund the CREATE2 address _before_ activation if sponsorship might fail.

#### `elytro account list`

```bash
elytro account list [alias|address] [-c <chainId>]
```

**Success result:**

```json
{
  "accounts": [
    {
      "active": true,
      "alias": "swift-panda",
      "address": "0x...",
      "chain": "Sepolia",
      "chainId": 11155111,
      "deployed": true,
      "recovery": false
    }
  ],
  "total": 1
}
```

#### `elytro account info`

```bash
elytro account info [alias|address]
```

Fetches live on-chain data (balance, deployment status). Requires RPC access.

**Success result:**

```json
{
  "alias": "swift-panda",
  "address": "0x...",
  "chain": "Sepolia",
  "chainId": 11155111,
  "deployed": true,
  "balance": "0.05 ETH",
  "recovery": false,
  "securityStatus": { "hookInstalled": true },
  "explorer": "https://..."
}
```

If security is pending (pre-activate): includes `securityIntent` instead of `securityStatus`.

**Use this command to verify security status before transactions.** If `securityStatus`
is absent or `hookInstalled` is false, the account is unprotected.

#### `elytro account switch`

```bash
elytro account switch <alias|address>
```

**Always pass alias or address** — without arguments it shows an interactive
selector (not suitable for agents).

**Success result:**

```json
{
  "alias": "cold-storage",
  "address": "0x...",
  "chain": "Sepolia",
  "chainId": 11155111,
  "deployed": true,
  "balance": "0.1 ETH"
}
```

After switching, SDK and wallet client re-initialize to the new account's chain.

---

### Transaction Commands

All transaction commands use the unified `--tx` flag:

```
--tx "to:0xAddress,value:0.1,data:0xAbcDef"
```

**Rules**:

- `to` is always required (valid Ethereum address).
- At least one of `value` or `data` must be present.
- `value` is in ETH (e.g. `"0.001"`), not wei.
- `data` is hex-encoded calldata (`0x` prefix, even length). Use `data:0x` for plain ETH transfers.
- Multiple `--tx` flags → batch (`executeBatch`). Order preserved.

> **Pre-flight requirement**: Before any `tx send`, verify the account has
> `securityStatus.hookInstalled === true` via `account info` or `security status`.
> Sending from an unprotected account risks total fund loss if the key is compromised.

#### `elytro tx send`

```bash
elytro tx send [account] --tx <spec> [--no-sponsor] [--no-hook] [--userop <json>]
```

```bash
# ETH transfer
elytro tx send --tx "to:0xRecipient,value:0.001"

# Contract call
elytro tx send --tx "to:0xContract,data:0xa9059cbb..."

# Batch (multiple --tx flags, order preserved)
elytro tx send --tx "to:0xA,value:0.1" --tx "to:0xB,data:0xab"

# From specific account
elytro tx send my-alias --tx "to:0xAddr,value:0.01"

# Skip sponsorship / Skip 2FA hook
elytro tx send --tx "to:0xAddr,value:0.01" --no-sponsor
elytro tx send --tx "to:0xAddr,value:0.01" --no-hook

# Send pre-built UserOp (skips build step)
elytro tx send --userop '{"sender":"0x...",...}'
```

**Pipeline**: resolve account → balance pre-check → build UserOp → fee data →
estimate gas (fakeBalance) → sponsor → confirm → sign → send → wait for receipt.

**Success result:**

```json
{
  "status": "confirmed",
  "account": "swift-panda",
  "address": "0x...",
  "transactionHash": "0x...",
  "block": "12345",
  "gasCost": "0.0001 ETH",
  "sponsored": true,
  "explorer": "https://..."
}
```

Cancelled by user: `{ "status": "cancelled" }`.

**Critical**: Sponsor covers gas only, **not** the transaction value. If sending
0.1 ETH, the account must hold >= 0.1 ETH regardless of sponsorship.

**Exit codes**: 0 = success, 1 = error or execution reverted.

#### `elytro tx build`

```bash
elytro tx build [account] --tx "to:0xAddr,value:0.1" [--no-sponsor]
```

Same pipeline as `send` but stops before signing. Returns the full unsigned UserOp.

**Success result:**

```json
{
  "userOp": { "sender": "0x...", "callData": "0x...", "...": "..." },
  "account": "swift-panda",
  "address": "0x...",
  "chain": "Sepolia",
  "chainId": 11155111,
  "txType": "ETH Transfer",
  "sponsored": true
}
```

#### `elytro tx simulate`

```bash
elytro tx simulate [account] --tx "to:0xAddr,value:0.1" [--no-sponsor]
```

Dry-run with gas breakdown, balance check, and warnings.

**Success result:**

```json
{
  "txType": "ETH Transfer",
  "account": "swift-panda",
  "address": "0x...",
  "chain": "Sepolia",
  "chainId": 11155111,
  "transactions": [{ "to": "0x...", "value": "0.1" }],
  "gas": {
    "callGasLimit": "21000",
    "verificationGasLimit": "100000",
    "preVerificationGas": "50000",
    "maxFeePerGas": "1000000000",
    "maxPriorityFeePerGas": "100000000",
    "maxCost": "0.00017 ETH"
  },
  "sponsored": true,
  "balance": "0.5 ETH",
  "warnings": ["Insufficient balance for value: need 1.0, have 0.5 ETH"]
}
```

The `warnings` array is only present when issues are detected.

---

### Query Commands

All query commands are read-only and do not require confirmation.

#### `elytro query balance`

```bash
elytro query balance [alias|address]                 # ETH balance
elytro query balance [alias|address] --token 0xAddr  # ERC-20 balance
```

**Success result (ETH):**

```json
{
  "account": "swift-panda",
  "address": "0x...",
  "chain": "Sepolia",
  "symbol": "ETH",
  "balance": "0.05"
}
```

**Success result (ERC-20):**

```json
{
  "account": "swift-panda",
  "address": "0x...",
  "chain": "Sepolia",
  "token": "0x...",
  "symbol": "USDC",
  "decimals": 6,
  "balance": "1000.0"
}
```

#### `elytro query tokens`

```bash
elytro query tokens [alias|address]
```

Lists all ERC-20 tokens with non-zero balances. Uses Alchemy — requires Alchemy key.

**Success result:**

```json
{
  "account": "swift-panda",
  "address": "0x...",
  "chain": "Sepolia",
  "tokens": [
    { "address": "0x...", "symbol": "USDC", "decimals": 6, "balance": "1000.0" }
  ],
  "total": 1
}
```

#### `elytro query tx`

```bash
elytro query tx <hash>
```

**Success result:**

```json
{
  "hash": "0x...",
  "status": "success",
  "block": "12345",
  "from": "0x...",
  "to": "0x...",
  "gasUsed": "21000",
  "chain": "Sepolia"
}
```

#### `elytro query chain`

```bash
elytro query chain
```

**Success result:**

```json
{
  "chainId": 11155111,
  "name": "Sepolia",
  "nativeCurrency": "ETH",
  "rpcEndpoint": "https://opt-sepolia.g.alchemy.com/v2/***",
  "bundler": "https://api.pimlico.io/v2/.../rpc?apikey=***",
  "blockExplorer": "https://sepolia-optimism.etherscan.io",
  "blockNumber": "12345678",
  "gasPrice": "1000000 wei (0.000021 ETH per basic tx)"
}
```

API keys are always masked in output.

#### `elytro query address`

```bash
elytro query address <0xAddress>
```

**Success result:**

```json
{
  "address": "0x...",
  "chain": "Sepolia",
  "type": "contract",
  "balance": "1.5 ETH",
  "codeSize": "4096 bytes"
}
```

`codeSize` is only present for contract addresses.

---

### Security Commands (2FA)

All security commands require the account to be **deployed**. All output structured JSON.

> **Reminder**: These commands are for managing security _after_ the account is
> deployed. For new accounts, the recommended path is `--email` + `--daily-limit`
> at `account create` time, which installs the hook atomically during `activate`.
> Use these commands to manage an existing hook or to install one retroactively.

#### `elytro security status`

```bash
elytro security status
```

**Use this to verify the account is protected before sending transactions.**

**Success result:**

```json
{
  "account": "swift-panda",
  "address": "0x...",
  "chain": "Sepolia",
  "chainId": 11155111,
  "hookInstalled": true,
  "hookAddress": "0x...",
  "capabilities": {
    "preUserOpValidation": true,
    "preIsValidSignature": true
  },
  "profile": {
    "email": "u***@example.com",
    "emailVerified": true,
    "dailyLimitUsd": "100.00"
  }
}
```

**What to check:**

- `hookInstalled` must be `true`
- `profile.emailVerified` must be `true`
- If either is `false`, the account is not fully protected

#### `elytro security 2fa install`

```bash
elytro security 2fa install [--capability <1|2|3>]
```

Capability flags: 1=SIGNATURE_ONLY, 2=USER_OP_ONLY, 3=BOTH (default: 3).

Only needed if `activate` was run without a security intent (i.e., the account was
created without `--email`/`--daily-limit`). For new accounts, prefer the atomic
create → activate flow instead.

**Success result:**

```json
{
  "status": "installed",
  "account": "swift-panda",
  "address": "0x...",
  "hookAddress": "0x...",
  "capability": "BOTH (UserOp + Signature)",
  "safetyDelay": 172800
}
```

If already installed: `{ "status": "already_installed" }`.

**After installing the hook, you MUST also bind an email** (`security email bind`)
for the 2FA to be functional. The hook alone without a verified email provides
no OTP protection.

#### `elytro security 2fa uninstall`

```bash
elytro security 2fa uninstall                   # Normal (requires hook signature)
elytro security 2fa uninstall --force            # Start force-uninstall countdown
elytro security 2fa uninstall --force --execute  # Execute after safety delay
```

**Normal uninstall result:** `{ "status": "uninstalled" }`
**Force start result:** `{ "status": "force_uninstall_started", "safetyDelay": 172800 }`
**Force execute result:** `{ "status": "force_uninstalled" }`
**Already started:** `{ "status": "already_initiated", "canExecute": false, "availableAfter": "..." }`

#### `elytro security email bind`

```bash
elytro security email bind <email>
```

**This is the critical step that makes 2FA functional.** Requires interactive OTP
input — the user must check their email and provide the 6-digit code. The agent
MUST wait for this input and cannot skip it.

**Success result:**

```json
{
  "status": "email_bound",
  "email": "u***@example.com",
  "emailVerified": true
}
```

#### `elytro security email change`

```bash
elytro security email change <email>
```

**Success result:** `{ "status": "email_changed", "email": "..." }`

#### `elytro security spending-limit`

```bash
elytro security spending-limit          # View current
elytro security spending-limit 100      # Set to $100/day (requires OTP)
```

**View result:** `{ "dailyLimitUsd": "100.00", "email": "u***@example.com" }`
**Set result:** `{ "status": "daily_limit_set", "dailyLimitUsd": "100.00" }`

---

### Configuration Commands

#### `elytro config show`

```bash
elytro config show
```

**Success result:**

```json
{
  "rpcProvider": "Alchemy (user-configured)",
  "bundlerProvider": "Pimlico (user-configured)",
  "alchemyKey": "abc1***xyz9",
  "currentChain": "Sepolia",
  "chainId": 11155111,
  "rpcEndpoint": "https://opt-sepolia.g.alchemy.com/v2/***",
  "bundler": "https://api.pimlico.io/v2/.../rpc?apikey=***"
}
```

#### `elytro config set`

```bash
elytro config set alchemy-key <KEY>
elytro config set pimlico-key <KEY>
```

**Success result:** `{ "key": "alchemy-key", "status": "saved", "rpcEndpoint": "...", "bundler": "..." }`

#### `elytro config remove`

```bash
elytro config remove alchemy-key
```

**Success result:** `{ "key": "alchemy-key", "status": "removed" }`

---

## Token Swaps (Uniswap Planner + Elytro Execution)

Elytro relies on Uniswap AI for routing logic, but swaps are executed through the
Elytro CLI. Combine two skills:

- `defi/uniswap` — collects intent and returns calldata/UserOps plus a price/route summary.
- `defi/elytro` — simulates and sends the planner artifact from your Elytro account.

### Workflow

1. **Load `defi/uniswap`** — give it the Elytro account alias/address, chain, tokens, and amount. Ask for either `{chainId, target, calldata, valueEth}` or `{userOperation}` output plus a human-readable summary.
2. **Validate the response** — confirm router/contract addresses are legitimate, the chain ID matches the user request, and slippage/deadlines align with requirements.
3. **Preview for the user** — render the summary (amount in/out, route, gas sponsorship) and ask for explicit approval before proceeding.
4. **Execute through `defi/elytro`** — on approval, run `elytro tx simulate` first, then `elytro tx send` using the planner's calldata/UserOp.
5. **Report results** — share bundler hash / transaction hash plus refreshed balances (`elytro query balance`) back to the user.

### Rules for Swaps

- **Never guess swap outputs.** Always surface the exact minimums, routes, and slippage from `defi/uniswap`.
- **Verify contract addresses** before simulating or sending.
- **Document sponsorship** — note whether `elytro tx simulate` reports `"sponsored": true/false` so the user knows who pays gas.
- **Stay in one flow.** Do not open external swap links; all execution happens in Elytro once the planner provides calldata/UserOps.

### Example

```bash
# Planner output (from defi/uniswap)
CALLDATA=0x3593564c...
ROUTER=0xE592427A0AEce92De3Edee1F18E0157C05861564
CHAIN=42161
VALUE_ETH=0

# Simulate then execute (defi/elytro instructions)
elytro tx simulate demo-arb \
  --tx "to:$ROUTER,value:$VALUE_ETH,data:$CALLDATA"
elytro tx send demo-arb \
  --tx "to:$ROUTER,value:$VALUE_ETH,data:$CALLDATA"
```

---

## Security Intent / Status Model

Security configuration follows a two-phase model:

1. **SecurityIntent** (temporary) — Created at `account create` time when `--email`
   and/or `--daily-limit` are provided. Stored per account + per chain. Exists only
   to bridge the gap between create and activate.

2. **SecurityStatus** (persistent) — Written by `activate` after SecurityHook is
   successfully installed on-chain. Intent is deleted at this point.

**Rule**: Intent exists = not yet executed. Intent absent = consumed or never configured.
This prevents double-installation and makes state inspection trivial.

---

## Agent Workflow Patterns

### Pattern 1: Fresh Setup + Send ETH (Security-First)

```bash
# Initialize
elytro init

# Create with security — ALWAYS include --email and --daily-limit
elytro account create --chain 11155111 --alias agent-primary \
  --email user@example.com --daily-limit 100

# Bind email — INTERACTIVE, requires human OTP input
elytro security email bind user@example.com
# → agent pauses, user enters 6-digit OTP from inbox

# Fund address via faucet or external transfer (before activate)

# Activate — deploys + installs SecurityHook atomically
RESULT=$(elytro account activate agent-primary)
# → VERIFY: result.hookInstalled === true. If false, STOP.

# Now safe to transact
elytro tx send --tx "to:0xRecipient,value:0.001"
```

### Pattern 2: Pre-Transaction Safety Check

```bash
# Before every transaction session, verify security is active
STATUS=$(elytro security status)
# → Check: hookInstalled === true AND profile.emailVerified === true
# → If not, guide user through security email bind

BALANCE=$(elytro query balance agent-primary)
# → parse JSON: result.balance
# → verify sufficient funds (sponsor covers gas, NOT value)

elytro tx send --tx "to:0xRecipient,value:0.001"
```

### Pattern 3: Batch Multiple Operations

```bash
elytro tx send \
  --tx "to:0xAlice,value:0.01" \
  --tx "to:0xBob,value:0.02" \
  --tx "to:0xContract,data:0xa9059cbb..."
```

All packed into one UserOp (`executeBatch`), executed atomically.

### Pattern 4: Simulate → Send

```bash
SIM=$(elytro tx simulate --tx "to:0xAddr,value:0.5")
# → parse result.warnings array (empty = safe to proceed)
# → check result.gas.maxCost, result.sponsored
elytro tx send --tx "to:0xAddr,value:0.5"
```

### Pattern 5: Multi-Account Management

```bash
elytro account create --chain 11155111 --alias hot-wallet \
  --email user@example.com --daily-limit 50
elytro account create --chain 11155111 --alias cold-storage \
  --email user@example.com --daily-limit 500
# → bind email + activate BOTH accounts before transacting

elytro account switch hot-wallet
elytro tx send --tx "to:0xAddr,value:0.01"
elytro account switch cold-storage
elytro query balance
```

### Pattern 6: Token Swap via Uniswap

```bash
# 1. Query current holdings
elytro query tokens my-wallet

# 2. Invoke defi/uniswap planner (external skill)
#    → returns { chainId, target, calldata, valueEth } + summary

# 3. Simulate the swap through Elytro
elytro tx simulate my-wallet --tx "to:$ROUTER,value:$VALUE_ETH,data:$CALLDATA"
# → check warnings array, verify sponsored status

# 4. Get user approval, then execute
elytro tx send my-wallet --tx "to:$ROUTER,value:$VALUE_ETH,data:$CALLDATA"

# 5. Verify updated balances
elytro query balance my-wallet
elytro query tokens my-wallet
```

---

## Agent Implementation Checklist

When implementing Elytro wallet interactions, ensure:

- [ ] **Version check**: Run `elytro update check` at session start. Notify user if an update is available.
- [ ] **Real-time data fetching**: Always run `elytro query balance`, `elytro account info`, etc. before making decisions. Never use stale or assumed values.
- [ ] **Security verification**: Before any `tx send`, confirm `hookInstalled: true` and `emailVerified: true`.
- [ ] **JSON parsing**: All command output is structured JSON. Parse `stdout` and check `success` field.
- [ ] **Confirmation handling**: When commands pause for `(y/N)`, present the user with a clear summary and wait for explicit approval.
- [ ] **Error handling**: If a command returns `success: false`, parse the error code and message. Use the Error Recovery table below.
- [ ] **Chain awareness**: Account operations use the account's chain. Switching accounts may change the active chain. Always verify chain context.
- [ ] **Balance pre-check**: Before `tx send`, verify the account has enough ETH for the value being sent (sponsor only covers gas).
- [ ] **Multi-step flows**: For send and swap, follow the full pipeline: query → simulate → confirm → send → verify.

---

## Invariants the Agent Must Respect

1. **Always `init` before anything else.** Every command needs the vault key.
2. **Always `account create` with `--email` and `--daily-limit`.** No exceptions in production workflows.
3. **Always `security email bind` after create.** The hook needs a verified email to enforce 2FA.
4. **Always verify `hookInstalled: true` after activate.** If false, the account is unprotected.
5. **Never `tx send` without confirmed security.** Check `security status` if in doubt.
6. **Always `activate` before `tx send`.** Transactions require a deployed contract.
7. **Fund before `activate` if sponsorship might fail.** The CREATE2 address is deterministic.
8. **Sponsor covers gas, not value.** If sending ETH, the account needs that ETH.
9. **Chain is per-account.** Switching accounts may change the active chain.
10. **Always pass alias/address to `account switch`.** The interactive selector is not agent-compatible.
11. **Parse JSON from every command.** Success/failure is in the `success` field.
12. **API keys are never exposed.** All error messages and URLs are sanitized automatically.
13. **Never guess on-chain data.** Always query before acting.
14. **`value` is always ETH, not wei.** The CLI handles conversion internally.

---

## Error Recovery

| Error Pattern                | Cause                                   | Recovery                                |
| ---------------------------- | --------------------------------------- | --------------------------------------- |
| "Wallet not initialized"     | No `~/.elytro/keyring.json`             | Run `elytro init`                       |
| "Keyring is locked"          | Missing vault key (Keychain or env var) | Check Keychain entry or SecretRef       |
| "Vault key not found"        | Provider available but key missing      | Re-run `elytro init` or check SecretRef |
| "Wallet unlock failed"       | Wrong vault key                         | Verify the correct key in SecretRef     |
| "No accounts found"          | No account created yet                  | Run `elytro account create`             |
| "Account not deployed"       | Trying to send tx before activation     | Run `elytro account activate`           |
| "Insufficient balance"       | Value exceeds account balance           | Fund the account first                  |
| JSON-RPC error code `-32001` | Resource not found on-chain             | Check hash/address, try different chain |
| "AA21" in error              | UserOp simulation failed                | Usually a balance or nonce issue        |

---

## Update / Upgrade

**Agents should check the installed version at the start of every session** and
inform the user when a newer version is available.

### `elytro --version` (v0.4.0+)

```bash
elytro --version
```

Prints the currently installed version and exits.

### `elytro update check` (v0.4.0+)

```bash
elytro update check
```

Read-only version check. No side effects — safe to run unconditionally.

**Success result (update available):**

```json
{
  "currentVersion": "0.4.0",
  "latestVersion": "0.5.0",
  "updateAvailable": true,
  "upgradeCommand": "npm install -g @elytro/cli@0.5.0"
}
```

**Success result (up to date):**

```json
{
  "currentVersion": "0.5.0",
  "latestVersion": "0.5.0",
  "updateAvailable": false
}
```

### `elytro update` / `elytro upgrade` (v0.4.0+)

```bash
elytro update
elytro upgrade   # alias
```

Checks for a newer version and installs it automatically. Auto-detects the
package manager (npm, yarn, pnpm, bun) from the environment.

**Success result:**

```json
{
  "previousVersion": "0.4.0",
  "currentVersion": "0.5.0",
  "updateAvailable": false,
  "packageManager": "npm"
}
```

### Agent Update Protocol

1. Run `elytro --version` to detect the installed version.
2. If the command **succeeds** (v0.4.0+), use `elytro update check` to check for
   updates, then `elytro update` to upgrade if the user approves.
3. If `--version` **fails** with an unknown-command/flag error, the user is on
   v0.3.x or earlier which lacks both `--version` and `update`. Guide them to
   upgrade manually:
   ```bash
   npm install -g @elytro/cli@latest
   # or: pnpm add -g @elytro/cli@latest / yarn global add @elytro/cli@latest / bun add -g @elytro/cli@latest
   ```
   Inform the user: _"Your Elytro CLI version does not support auto-update.
   Please run the command above to upgrade to the latest version."_
4. Do **not** auto-upgrade without user consent — recommend the update and wait
   for approval before proceeding.

---

## Notes

- Elytro uses **EIP-4337 (Account Abstraction)** — transactions are submitted as UserOps via a Bundler, not regular EOA transactions. On-chain, all UserOps appear as bundler (0x4337001F) → EntryPoint (0x4337084D) transactions.
- Sponsored transactions are **gasless by default** when a paymaster is available. The paymaster covers gas fees only, not the transaction's ETH value.
- The wallet data directory is `~/.elytro/`. No plaintext key files exist on disk.
- `--chain` accepts both hex (`0xaa36a7`) and decimal (`11155111`) chain IDs.
- `data` field in `--tx` specs must be a valid hex string with `0x` prefix and even length. Use `data:0x` (or omit `data`) for plain ETH transfers.
- The Security Intent / Status two-phase model ensures SecurityHook is installed atomically during deployment. Intent is temporary (create → activate); Status is persistent (post-activate).

---

## Storage Layout

```
~/.elytro/
├── keyring.json      # AES-GCM encrypted EOA private key vault
├── accounts.json     # Account list (alias, address, chainId, index, owner,
│                     #   deployed, securityIntent?, securityStatus?)
└── config.json       # Chain config, current chain, API keys (persisted)
```

No plaintext key files on disk. The vault key lives in macOS Keychain or is injected
via `ELYTRO_VAULT_SECRET` environment variable. Deleting `~/.elytro/` resets local
state; on-chain contracts are unaffected.

---

## UX Rules

Detect the active channel from the system prompt or host context at session start.
Use the matching format consistently.

### Channel: telegram

Telegram supports Inline Buttons. Use them for every confirmation and every list.
Never ask for text input when a button is sufficient.

Rules:

- every confirmation prompt must include Inline Buttons
- every account list, token list, tx history list, or selectable detail list must use Inline Buttons
- use Telegram MarkdownV2 for message text
- escape MarkdownV2 special characters correctly
- **never use Markdown tables (`|...|`)** because Telegram renders them badly
- use labeled lines such as `Alias: value`, `Address: value`, `Gas: value`
- only request text input when the workflow genuinely requires freeform input, such as OTP entry

**Session start card:**

```text
🔐 *Elytro Wallet ready*

Wallet: `{alias}` \(`{address_short}`\)
Network: {chain}
Security: {hookInstalled ? "✅ Protected" : "⚠️ Not protected"}
Balance: {balance} ETH
{updateAvailable ? "⚠️ Update available: `" + latestVersion + "`" : ""}
```

Buttons: `[Check Balance]` `[Send]` `[Swap]` `[Accounts]`

**Account created card:**

```text
✅ *Account created*

Alias: `{alias}`
Address: `{address}`
Chain: {chain} \({chainId}\)
Deployed: ❌ Not yet
Email: {email}
Daily limit: \${dailyLimitUsd} USD

Next: verify your email to activate 2FA\.
```

Buttons: `[Verify Email Now]` `[Later]`
Callback data: `email_bind_{alias}` / `email_bind_skip_{alias}`

**Account activated card:**

```text
✅ *Account activated*

Alias: `{alias}`
Address: `{address_short}`
Chain: {chain}
Hook: {hookInstalled ? "✅ SecurityHook installed" : "❌ Hook missing"}
Tx: `{txHash_short}`
```

**Transaction confirmation card:**

```text
📤 *Confirm transaction*

From: `{alias}` \(`{address_short}`\)
To: `{to_short}`
Amount: {amount_line}
Gas: {sponsored ? "Sponsored \(free\)" : gasCost}
Network: {chain}
```

Buttons: `[✅ Confirm]` `[❌ Cancel]`
Callback data: `tx_confirm_{nonce}` / `tx_cancel_{nonce}`

**Transaction confirmed card:**

```text
✅ *Transaction confirmed*

Tx: `{txHash_short}`
Block: {block}
Gas: {sponsored ? "Sponsored \(free\)" : gasCost}
[View on Explorer]({explorerUrl})
```

**Error card:**

```text
❌ *Transaction failed*

Reason: {error\.message}
Code: `{error.code}`

{recovery_hint}
```

Buttons: `[Retry]` `[Cancel]`

**Account list:**

```text
🗂 *Your accounts*
Select an account:
```

Buttons: one row per account, for example
`[{alias} ({address_short}) {deployed ? "✅" : "⚠️"}]`

Callback data: `account_select_{alias}`

**Balance card:**

```text
💰 *Balance - {alias}*

ETH: {eth_balance}
Network: {chain}
```

Token buttons: one button per non-zero token, for example
`[{symbol}: {balance}]`

Callback data: `token_detail_{tokenAddress}`

**Token list:**

```text
🪙 *Token holdings - {alias}*
```

Buttons: `[{symbol} - {balance}]`
Callback data: `token_detail_{tokenAddress}`

**Security status card:**

```text
🔒 *Security status*

Hook: {hookInstalled ? "✅ Installed" : "❌ Not installed"}
Email: {emailVerified ? "✅ Verified" : "❌ Not verified"} \({email\_masked}\)
Daily limit: \${dailyLimitUsd}
```

Buttons when relevant: `[Setup Security]` `[Change Email]` `[Change Limit]`
Callback data: `security_setup` / `security_email_change` / `security_limit_change`

**OTP prompt:**

```text
📧 *Check your email*

A 6\-digit code was sent to {email\_masked}\.
Reply with the code to verify\.
```

**Update available card:**

```text
⚠️ *Update available*

Current: `{currentVersion}`
Latest: `{latestVersion}`
```

Buttons: `[✅ Update now]` `[Skip]`
Callback data: `update_confirm` / `update_skip`

**Swap confirmation card:**

```text
🔄 *Confirm swap*

From: {amountIn} {symbolIn}
To: >= {amountOutMin} {symbolOut}
Route: {route}
Gas: {sponsored ? "Sponsored \(free\)" : gasCost}
Network: {chain}
```

Buttons: `[✅ Confirm swap]` `[❌ Cancel]`
Callback data: `swap_confirm_{nonce}` / `swap_cancel_{nonce}`

### Channel: default

Use standard Markdown. Human-readable formatting is allowed, but still prefer
simple labeled blocks over tables for wallet and account summaries because small
models reproduce labeled blocks more reliably than tables.

Rules:

- confirmations must be explicit
- tables are allowed but discouraged for account creation, balances, and transaction previews
- prefer short sections with labeled lines
- keep output deterministic across repeated runs

Recommended account-created format:

```text
Account created

Alias: {alias}
Address: {address}
Chain: {chain} ({chainId})
Deployed: No
Email: {email}
Daily limit: ${dailyLimitUsd} USD

Next step: run `elytro security email bind {email}` and wait for the user's OTP.
```
