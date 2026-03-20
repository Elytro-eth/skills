# Elytro — Agent cheat sheet & user messaging

**Purpose:** Fast lookup for agents: what to run, what needs extra user consent, and **fixed wording** for humans—not a full CLI spec.

**Behaviour rules & templates:** [SKILL.md](SKILL.md)

---

## Commands (essentials)

### Account

| Run | Tell the user on success |
|-----|--------------------------|
| `account create -c <chainId> [-a alias] [-e email] [-l dailyLimitUsd]` | “Smart account created.” Share **alias** and **address**; note it’s not deployed until activate. |
| `account activate [alias\|address] [--no-sponsor]` | “Account deployed on-chain.” Mention **hookInstalled** if you surface it. |
| `account list [-c <chainId>]` | Short list: alias + chain + deployed yes/no. |
| `account info [alias\|address]` | Balance + **whether deployed** + security snapshot in plain language. |
| `account rename …` | “Renamed to &lt;new&gt;.” |
| `account switch <alias\|address>` | “Active account is now &lt;alias&gt;.” **Always pass alias/address** (avoid interactive pick). |

### Transaction

| Run | Tell the user |
|-----|---------------|
| `tx simulate [account] --tx <spec>… [--no-sponsor]` | Use **Preview —** template in SKILL.md (cost, sponsored, balance, **every** warning). |
| `tx send [account] --tx <spec>… [--no-sponsor] [--no-hook]` | Use **Sent —** or **OTP pending** template; never send without prior simulate + user OK. |
| `tx build …` | “Unsigned operation prepared” (only if user needs builds). |

`--tx` shape: `to:0x…,value:0.1,data:0x…` (`to` + either `value` or `data`; repeat `--tx` for batch).

### Query

| Run | Tell the user |
|-----|---------------|
| `query balance [account] [--token 0xAddr]` | One line: who + how much (+ symbol). |
| `query tokens …` | Short summary or “holdings listed in result” if many. |
| `query tx <hash>` | Confirmed / pending / not found—in plain words. |
| `query chain` | “You’re on &lt;name&gt; (&lt;id&gt;).” |
| `query address <0x…>` | Type + balance one-liner. |

### Security / OTP / config

| Run | Tell the user |
|-----|---------------|
| `security status` | Hook on/off, email verified or not, limit—**no jargon**. |
| `security 2fa install …` / `uninstall …` | Only after user approval; then outcome one-liner. |
| `security email bind|change <email>` | Often → **OTP pending** template. |
| `security spending-limit [usd]` | View: quote limit; set: may → **OTP pending**. |
| `otp submit <id> <code>` | Only when **user** supplies code. |
| `otp list` / `otp cancel` | Explain pending / cancelled in one sentence. |
| `config show|set|remove` | “Settings updated” or “key removed—RPC may use defaults.” |
| `update check` / `update` | Update available vs upgraded; no auto-update without OK. |

---

## Agent: user approval before running

Say what you will run, **wait for explicit yes**, then execute.

**Money & deploy:** `tx send` (especially `--no-hook`), `account activate`  
**Security hook:** `security 2fa install`, `security 2fa uninstall` (any variant)  
**Account safety:** `security email bind|change`, `security spending-limit` **when setting**  
**OTP / config:** `otp submit` (user provides code), `otp cancel`, `config remove`, `update`

---

## Error recovery (human)

Use this table for **uniform** reassurance and next steps (codes are optional in parentheses for debugging).

| User-visible situation | Suggested **Try:** line |
|------------------------|-------------------------|
| Wallet / init missing | “Run `elytro init` once on this machine.” |
| Not deployed | “Run `account activate` for this account after we confirm.” |
| Not enough balance | “Fund this smart account, then we’ll check balance again.” |
| Sponsorship / paymaster issue | “We can retry with `--no-sponsor` if you pay gas—only if you agree.” |
| Transaction build / estimation failed | “Check the transaction line (to, value, data) and chain.” |
| Send / network failed | “Temporary network or bundler issue—retry shortly or check RPC keys.” |
| On-chain revert | “The chain rejected the call—adjust amount/calldata or ask the dApp.” |
| 2FA / email / OTP issues | “Same account as when we started; check `otp list` or redo the step after email.” |
| Wrong or unsupported chain | “Use a supported chain ID from the error message.” |

---

## Optional: error codes (debug only)

If the user cares about numbers: `-32602` bad parameters · `-32002` wallet/account not ready · `-32005` send failed · `-32007` hook auth · `-32010`–`-32014` OTP family · `-32000` generic.

Full internal list is not required for day-to-day agent use.
