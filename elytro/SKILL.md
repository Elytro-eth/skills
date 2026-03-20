---
name: elytro
description: >
  Elytro smart-account wallet CLI for agents: multi-chain ERC-4337, 2FA email OTP, spending limits.
  Teaches agents to simulate before send, get user approval on risky steps, and explain outcomes in a
  fixed, user-friendly format (no raw JSON unless asked). Deferred OTP completed with otp submit.
  Use for: accounts, transfers, contract calls, email/security setup. Node >= 24.
version: 0.6.1
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

**Purpose:** Operate the Elytro smart account wallet **safely and predictably**.

**Install:** `npm install -g @elytro/cli` (Node ≥ 24)

**Command reference & risk list:** [references/commands.md](references/commands.md)

---

## How to read command output

- Prefer the structured payload: **`success`**, then **`result`** (or **`error`** on failure).
- **To the user:** translate into the fixed phrases in **User-visible outcomes** below. Offer raw JSON only if they ask.
- On failure, use **`error.message`** and any **`error.data`** hints (e.g. `hint`, supported chains)—rephrase in plain language.

---

## Operating rules

1. **Session start:** `elytro update check` — tell the user if an update exists; do not upgrade without consent.
2. **On-chain facts:** use `elytro query`; do not assume balances or chain state.
3. **Risky commands:** require **explicit user OK** before running anything listed in `commands.md` → _Agent: user approval_.
4. **OTP:** if the result indicates email verification is pending, use the **OTP pending** phrase and stop; user runs `otp submit` with the code when ready (see `commands.md`).
5. **Chains:** 1, 10, 42161, 8453, 11155111, 11155420 · amounts in **ETH** · calldata hex with `0x`.
6. **Before normal sends:** account **deployed**, **hook installed**, **email verified** (`security status`). Treat `--no-hook` as exceptional—only with user approval.
7. **Automation:** pass **account alias/address** for `switch`, `tx simulate`, and `tx send` when multiple accounts exist (avoid interactive pickers).
8. **Secrets:** never paste vault keys or API keys back into chat.

---

## Agent Communication

Keep **one line of status**, then **one line of next step** when useful. Same wording every time.

| Situation                   | What you tell the user                                                                                                                                                                                       |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Success (generic)**       | **Done —** &lt;plain summary of what changed&gt;. **Next:** &lt;optional one action&gt;                                                                                                                      |
| **Simulated (before send)** | **Preview —** Estimated max cost: &lt;from CLI&gt;. Sponsored: &lt;yes/no&gt;. Account balance: &lt;…&gt;. **Warnings:** &lt;list each, or “none”&gt;. **Please confirm** you want to send this transaction. |
| **Transaction confirmed**   | **Sent —** &lt;amount / what&gt; to &lt;short address&gt;. **Tx:** &lt;hash&gt;. **Explorer:** &lt;link if Result had one&gt;                                                                                |
| **OTP pending**             | **Email verification needed —** We’ve sent a code to &lt;masked email&gt;. **When you have the code:** run &lt;paste submitCommand from result&gt;                                                           |
| **Blocked (security)**      | **Not ready —** 2FA email or hook setup is incomplete. **Next:** &lt;one concrete elytro command&gt;                                                                                                         |
| **Failed**                  | **Couldn’t complete —** &lt;reason in plain English&gt;. **Try:** &lt;one fix&gt;                                                                                                                            |

### Formats

| Operation   | Format                                                                                                                                                                   |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Tx sent     | `✅ Sent 0.05 ETH to 0xAbc…1234. Tx: 0xdef… Explorer: <url>`                                                                                                             |
| Simulated   | Short summary from `tx simulate` **result**: include `gas.maxCost`, `sponsored` (yes/no), `balance`, and **every** `warnings[]` line; if no warnings, say so explicitly. |
| Balance     | `💰 agent-primary: 0.482 ETH`                                                                                                                                            |
| OTP pending | `🔐 OTP sent to <maskedEmail>. Run: elytro otp submit <id> <code>` (user must paste the code from email).                                                                |
| Error       | `❌ <description> (code -32xxx). → <fix>` — use `error.data` from stderr when present (`hint`, `supportedChains`, etc.).                                                 |

**Principles**: Lead with outcome. Surface explorer links from `result` when present. Map error codes using `references/commands.md`. Flag security gaps (`hookInstalled`, `emailVerified`, limits) before suggesting sends. Show raw JSON only if the user asks.

---

## Account lifecycle (for advice only)

`create` → `activate` → email + spending limit → **protected** (hook + verified email + limit).

| Safe to send? | Human-facing check                                      |
| :-----------: | ------------------------------------------------------- |
|      No       | “Account not deployed yet.”                             |
|      No       | “Account deployed but email not verified for security.” |
|      Yes      | “Security profile looks ready for sending.”             |

---

## First-time setup

```bash
elytro init
elytro account create -c 11155420 -a agent-primary -e u@x.com -l 100
elytro account activate agent-primary    # confirm with user first (deploy)
elytro security email bind u@x.com       # then OTP flow
elytro security spending-limit 100        # may OTP
elytro security status                   # confirm all green for your policy
```

Before recommending **send:** confirm deployed + security status the user expects + enough balance.

---

## Workflows

**Every send**

1. `tx simulate` — same account, same `--tx` lines, same sponsor flags as the planned send.
2. Reply using **Preview —** row above; wait for **explicit yes**.
3. `tx send` with the same arguments.

**Batch:** same number and order of `--tx` on simulate and send.

**Swaps / contract calls:** obtain calldata off-chain, then same simulate → confirm → send.

**Deferred OTP** (email bind, spending-limit, tx send when limit exceeded, 2fa uninstall)

```bash
elytro security email bind u@x.com
# Parse result.otpPending.id, result.otpPending.submitCommand
# User checks email → elytro otp submit <id> <code>
```

---

## When something goes wrong

Use the human’s language first; see **Error recovery (human)** in [references/commands.md](references/commands.md) for uniform **Try:** lines.

---
