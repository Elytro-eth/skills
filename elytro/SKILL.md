---
name: elytro
description: Ethereum EIP-4337 smart contract wallet designed for AI agents. Use this skill whenever the user wants to manage Ethereum accounts, check balances, send ETH/tokens, swap tokens on Uniswap, manage 2FA security hooks, or do anything with their Elytro wallet. Supports gasless transactions via sponsorship. For token swaps, combine this skill with `defi/uniswap` for planning and `defi/elytro` for execution.
homepage: https://elytro.com
metadata: {"clawdbot":{"emoji":"💎","requires":{"bins":["elytro"],"node":">=24.0.0"},"install":[{"id":"npm","kind":"npm","package":"@elytro/cli","bins":["elytro"],"label":"Install Elytro CLI (npm)"}]}}
---

# Elytro Wallet

Ethereum EIP-4337 smart contract wallet for agents. Supports gasless transactions, 2FA security hooks, and email OTP.

- Repo: https://github.com/Elytro-eth/Elytro
- Install: `npm install -g @elytro/cli` (requires Node >=24)

## RULES — Read before every action

1. **Never guess on-chain data.** Always run the appropriate `elytro query` command to fetch balances, transaction status, token holdings, etc. Do NOT infer or assume values.
2. **Always ask the user before confirming interactive prompts using Inline Buttons.** When any command outputs `(y/N)` or similar, STOP and present the user with **Telegram Inline Buttons** for confirmation (e.g., "Confirm", "Cancel" or "Approve", "Reject"). Example: `elytro tx send` shows "Sign and send this transaction? (y/N)" — pause and send an Inline Button message to the user with callback options instead of asking for text input. **ALL confirmations must use Inline Buttons, never text input.**
3. **Use Inline Buttons for all list displays.** Whenever displaying lists (account list, token list, balance details, transaction history, etc.), render each item as a **clickable Inline Button** with callback_data pointing to that item's action. Example: for account selection, create buttons like `[{"text": "myWallet (0x...)", "callback_data": "account_select_myWallet"}]` instead of showing plain text and asking the user to type.
4. **Chain IDs are hex or decimal.** Common values: Ethereum mainnet=`1`, Sepolia testnet=`0xaa36a7` (=11155111), Base=`8453`, Arbitrum=`42161`.
5. **`value` in tx specs is always in ETH**, not wei.
6. **Smart accounts must be deployed before sending transactions.** Check `elytro account info <account>` → "Deployed: Yes" before sending. If not deployed, run `elytro account activate <account>` first.
7. **Use alias or address interchangeably** wherever `<account>` appears.
8. **Inline Button Convention for Other Agents:** All agents reading this skill must adhere to the following callback_data naming scheme for consistency:
   - Confirmations: `confirm_yes`, `confirm_no`, `confirm_cancel`
   - Account selection: `account_select_<alias_or_address>`
   - Token selection: `token_select_<tokenAddress>`
   - Transaction approval: `tx_approve`, `tx_reject`, `tx_simulate`
   - Action menu: `action_<action_name>` (e.g., `action_send`, `action_swap`, `action_balance_check`)
   - This ensures all agents can parse and understand callbacks across the entire Elytro workflow ecosystem.

9. **Default Menu on Ambiguous Intent.** When the user opens or mentions the wallet without a specific action (e.g., "打开钱包", "open wallet", "show elytro", "check my wallet"), do NOT ask what they want. Instead, **immediately display the Main Menu as Inline Buttons**. Each button leads to a sub-menu or action flow as defined in the Navigation Flow Map below.

---

## Navigation Flow Map (Inline Button UI)

All navigation screens below **must** be rendered using Inline Buttons. When the user clicks a button, the agent proceeds to the corresponding sub-screen. Every sub-screen includes a "⬅️ Back" button to return to the parent menu.

### Level 0 — Main Menu (default entry point)

Trigger: user says anything ambiguous like "open wallet", "打开钱包", "elytro", "我的钱包", etc.

Agent must first run `elytro account list` silently in the background, then render:

```json
{
  "text": "💎 Elytro Wallet\n\nWelcome! What would you like to do?",
  "reply_markup": {
    "inline_keyboard": [
      [
        { "text": "👛 My Accounts", "callback_data": "menu_accounts" },
        { "text": "💰 Check Balance", "callback_data": "menu_balance" }
      ],
      [
        { "text": "💸 Send", "callback_data": "menu_send" },
        { "text": "🔄 Swap", "callback_data": "menu_swap" }
      ],
      [
        { "text": "🔐 Security / 2FA", "callback_data": "menu_security" },
        { "text": "⚙️ Settings", "callback_data": "menu_settings" }
      ],
      [
        { "text": "➕ Create New Account", "callback_data": "action_create_account" }
      ]
    ]
  }
}
```

### Level 1 — Sub-menus

**`menu_accounts`** → Run `elytro account list`, render each account as a button:
```json
{
  "text": "👛 Your Accounts:",
  "reply_markup": {
    "inline_keyboard": [
      [{ "text": "myWallet (0x1234...5678) - Sepolia ✅", "callback_data": "account_select_myWallet" }],
      [{ "text": "tradingBot (0x8765...4321) - Base ⏳", "callback_data": "account_select_tradingBot" }],
      [{ "text": "➕ Create New Account", "callback_data": "action_create_account" }],
      [{ "text": "⬅️ Back", "callback_data": "menu_main" }]
    ]
  }
}
```
- ✅ = deployed, ⏳ = not yet deployed
- Upon `account_select_<alias>` callback → run `elytro account info <alias>` and display account detail with action buttons:
```json
{
  "text": "👛 myWallet\n📍 Sepolia (0xaa36a7)\n🏷 0x1234...5678\n✅ Deployed\n💎 0.5 ETH",
  "reply_markup": {
    "inline_keyboard": [
      [
        { "text": "💸 Send", "callback_data": "action_send_from_myWallet" },
        { "text": "💰 Tokens", "callback_data": "action_tokens_myWallet" }
      ],
      [
        { "text": "🔄 Swap", "callback_data": "action_swap_from_myWallet" },
        { "text": "🔐 Security", "callback_data": "menu_security" }
      ],
      [{ "text": "⬅️ Back", "callback_data": "menu_accounts" }]
    ]
  }
}
```

**`menu_balance`** → If multiple accounts, show account selector first (same as `menu_accounts` but callback leads to balance view). If only one account, go straight to balance display with action buttons (as defined in the "Query" section below).

**`menu_send`** → If multiple accounts, show account selector (callback: `action_send_from_<alias>`). Then prompt for recipient address and amount using the confirmation Inline Button flow (tx summary → Approve / Reject / Simulate).

**`menu_swap`** → Kick off the Uniswap workflow (`defi/uniswap`). If multiple accounts, show account selector first, then gather tokens/amount before pinging the planner.

**`menu_security`** → Run `elytro security status`, display:
```json
{
  "text": "🔐 Security Status\n\n2FA: Installed ✅\nEmail OTP: user@example.com\nSpending Limit: $100/day",
  "reply_markup": {
    "inline_keyboard": [
      [
        { "text": "Install/Uninstall 2FA", "callback_data": "action_2fa_toggle" },
        { "text": "Change Email", "callback_data": "action_email_change" }
      ],
      [
        { "text": "Set Spending Limit", "callback_data": "action_spending_limit" }
      ],
      [{ "text": "⬅️ Back", "callback_data": "menu_main" }]
    ]
  }
}
```

**`menu_settings`** → Run `elytro config show`, display:
```json
{
  "text": "⚙️ Configuration\n\nAlchemy Key: ✅ Set\nPimlico Key: ❌ Not set\nActive Account: myWallet",
  "reply_markup": {
    "inline_keyboard": [
      [
        { "text": "Set Alchemy Key", "callback_data": "action_set_alchemy" },
        { "text": "Set Pimlico Key", "callback_data": "action_set_pimlico" }
      ],
      [
        { "text": "Switch Account", "callback_data": "menu_accounts" }
      ],
      [{ "text": "⬅️ Back", "callback_data": "menu_main" }]
    ]
  }
}
```

**`menu_main`** → Return to Level 0 Main Menu.

### Navigation Rules for Agents

1. **Always show the Main Menu when user intent is ambiguous.** Do not ask "what do you want to do?" — show the menu.
2. **Every sub-screen must include a "⬅️ Back" button** (callback_data: parent menu ID, e.g., `menu_main`, `menu_accounts`).
3. **Fetching real data is mandatory before rendering.** Run the corresponding `elytro` CLI command to get live data (accounts, balances, security status, config), then render buttons with actual values. Never use placeholder data in production.
4. **Multi-step flows** (e.g., Send ETH): guide the user through each step using successive Inline Button screens: account selection → recipient input → amount input → tx summary with Approve/Reject/Simulate.
5. **Callback routing**: parse the callback_data prefix to determine the handler:
   - `menu_*` → render the corresponding sub-menu
   - `account_select_*` → fetch and show account details
   - `token_select_*` → fetch and show token details
   - `action_*` → execute the corresponding action or start an action flow
   - `tx_*` → handle transaction lifecycle (approve, reject, simulate)
   - `confirm_*` → handle generic yes/no/cancel prompts

---

## Setup (first-time)

```bash
# 1. Initialize wallet (once per machine)
elytro init

# 2. Create a smart account on a chain
elytro account create --chain <chainId> --alias <alias>
# Example: elytro account create --chain 0xaa36a7 --alias myWallet

# 3. Deploy the smart contract on-chain (gasless by default)
elytro account activate <alias-or-address>
```

---

## Account Management

```bash
elytro account list                        # List all accounts
elytro account list --chain <chainId>      # Filter by chain
elytro account info <account>              # On-chain details (balance, deployed status, etc.)
elytro account switch <account>            # Set active account
```

**Inline Button Display for Account List:**
After running `elytro account list`, render results as Inline Buttons instead of plain text. Example:

```json
{
  "text": "Select an account to manage:",
  "reply_markup": {
    "inline_keyboard": [
      [
        { "text": "myWallet (0x1234...5678) - Sepolia", "callback_data": "account_select_myWallet" },
        { "text": "tradingBot (0x8765...4321) - Base", "callback_data": "account_select_tradingBot" }
      ],
      [
        { "text": "➕ Create New Account", "callback_data": "action_create_account" }
      ]
    ]
  }
}
```

Upon callback (e.g., `account_select_myWallet`), fetch account details via `elytro account info` and present next actions (send, check balance, swap, etc.) also as Inline Buttons.

---

## Query (read-only, no confirmation needed)

```bash
# ETH balance — ALWAYS run this; never guess the balance
elytro query balance <account>

# ERC-20 token balance
elytro query balance --token <tokenAddress> <account>

# All ERC-20 holdings
elytro query tokens <account>

# Transaction status by hash
elytro query tx <hash>

# Current chain info
elytro query chain
```

**Inline Button Display for Balance & Token Lists:**

When displaying balances or token holdings, format as Inline Buttons for quick action:

Example balance display with action buttons:
```json
{
  "text": "💰 Account: myWallet\n\n💎 ETH: 0.5 ETH (~$1,200)\n🪙 USDC: 2,000 USDC\n🌈 DAI: 1,500 DAI",
  "reply_markup": {
    "inline_keyboard": [
      [
        { "text": "💸 Send ETH", "callback_data": "action_send_eth" },
        { "text": "🔄 Swap", "callback_data": "action_swap" }
      ],
      [
        { "text": "🪙 Select Token", "callback_data": "action_select_token" },
        { "text": "📊 History", "callback_data": "action_history" }
      ],
      [
        { "text": "🔐 Security", "callback_data": "action_security" }
      ]
    ]
  }
}
```

Token selection example:
```json
{
  "text": "Select a token to view or manage:",
  "reply_markup": {
    "inline_keyboard": [
      [
        { "text": "ETH (0.5)", "callback_data": "token_select_ETH" },
        { "text": "USDC (2000)", "callback_data": "token_select_0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48" }
      ],
      [
        { "text": "DAI (1500)", "callback_data": "token_select_0x6B175474E89094C44Da98b954EedeAC495271d0F" }
      ]
    ]
  }
}
```

---

## Multi-Step Action Flows

### Send ETH / Token Flow (Inline Button Driven)

**Step 1: Account Selection** (if multiple accounts exist)
- Render `menu_accounts` buttons.
- Upon `account_select_<alias>` callback, proceed to Step 2.

**Step 2: Recipient Input**
- Send a message to the user requesting recipient address.
- User replies with address.
- Agent validates the address and proceeds to Step 3.

**Step 3: Amount Input**
- Send a message requesting the amount to send.
- User replies with amount.
- Agent validates the amount against account balance and proceeds to Step 4.

**Step 4: Transaction Summary & Approval**
- Render the tx summary as Inline Buttons (Approve, Reject, Simulate):
```json
{
  "text": "📋 Transaction Summary\n\n💰 Account: myWallet\n📤 To: 0xabcd...ef01\n💵 Amount: 0.1 ETH\n⛽ Gas: ~0.001 ETH (sponsored)\n\nProceed?",
  "reply_markup": {
    "inline_keyboard": [
      [
        { "text": "✅ Approve", "callback_data": "tx_approve" },
        { "text": "❌ Reject", "callback_data": "tx_reject" }
      ],
      [
        { "text": "🔍 Simulate First", "callback_data": "tx_simulate" }
      ],
      [
        { "text": "⬅️ Back", "callback_data": "menu_send" }
      ]
    ]
  }
}
```

**Step 5: Execution**
- Upon `tx_approve` callback, run `elytro tx send` and report result.
- Upon `tx_reject` callback, abort and return to Main Menu.
- Upon `tx_simulate` callback, run `elytro tx simulate`, show gas estimate and paymaster availability, then re-display approval buttons.

### Swap Flow (Inline Button Driven)

**Step 1: Account Selection** (if multiple accounts exist)
- Render `menu_accounts` buttons.
- Upon `account_select_<alias>` callback, proceed to Step 2.

**Step 2: Token Pair Selection**
- Render token list as Inline Buttons for "From" token.
- Upon `token_select_<fromToken>` callback, render list for "To" token.
- Upon second `token_select_<toToken>` callback, proceed to Step 3.

**Step 3: Amount Input**
- Request swap amount from user (text input).
- User replies with amount.
- Proceed to Step 4.

**Step 4: Planner Request (`defi/uniswap`)**
- Invoke the `defi/uniswap` skill with account alias, chain, from/to tokens, and amount.
- The planner returns a summary plus calldata or a UserOp that Elytro can execute.
- Display the preview (values pulled from the planner output) before asking for approval:
```json
{
  "text": "🔄 Swap Preview\n\n💵 From: 0.5 ETH\n💎 To: ~1,200 USDC\n📊 Price: 1 ETH = 2400 USDC\n⛽ Gas: ~0.001 ETH (sponsored)\n\nReady to swap?",
  "reply_markup": {
    "inline_keyboard": [
      [
        { "text": "✅ Confirm Swap", "callback_data": "tx_approve" },
        { "text": "❌ Cancel", "callback_data": "tx_reject" }
      ],
      [
        { "text": "⬅️ Adjust Amount", "callback_data": "menu_swap" }
      ]
    ]
  }
}
```

**Step 5: Execution (`defi/elytro`)**
- Upon `tx_approve` callback, feed the planner output into the `defi/elytro` workflow: run `elytro tx simulate` first, then `elytro tx send` if the simulation passes.
- Upon `tx_reject`, abort and surface the Main Menu.
- Upon `tx_simulate`, rerun `elytro tx simulate`, show gas/sponsorship, then re-display approval buttons.

---

## Transactions

### Send (requires user confirmation)

```bash
# ⚠ STOP before confirming — ask the user first!
elytro tx send --tx "to:<address>,value:<ethAmount>,data:<hex>"

# Multiple calls in one UserOp (batched)
elytro tx send --tx "to:0xA,value:0.1,data:0x" --tx "to:0xB,value:0,data:0xcafe"

# Skip sponsorship (user pays gas)
elytro tx send --no-sponsor --tx "to:<address>,value:<ethAmount>,data:0x"

# Skip 2FA hook check
elytro tx send --no-hook --tx "to:<address>,value:<ethAmount>,data:0x"

# Send a pre-built UserOp
elytro tx send --userop '<userOpJson>'
```

### Simulate (safe, no confirmation)

```bash
# Preview gas estimate and sponsor check without sending
elytro tx simulate --tx "to:<address>,value:<ethAmount>,data:<hex>"
```

### Build unsigned UserOp (no send)

```bash
elytro tx build --tx "to:<address>,value:<ethAmount>,data:<hex>" <account>
```

---

## Token Swaps (Uniswap planner + Elytro execution)

Elytro still relies on Uniswap AI for routing logic, but swaps are executed through the Elytro CLI. Combine two skills:

- `defi/uniswap` – collects intent and returns calldata/UserOps plus a price/route summary.
- `defi/elytro` – simulates and sends the planner artifact from your Elytro account.

### Workflow

1. **Load `defi/uniswap`** – give it the Elytro account alias/address, chain, tokens, and amount. Ask for either `{chainId, target, calldata, valueEth}` or `{userOperation}` output plus a human-readable summary.
2. **Validate the response** – confirm router/contract addresses are legitimate, the chain ID matches the user request, and slippage/deadlines align with requirements.
3. **Preview for the user** – render the summary (amount in/out, route, gas sponsorship) with Inline Buttons (`tx_approve`, `tx_reject`, `tx_simulate`).
4. **Execute through `defi/elytro`** – on approval, run `elytro tx simulate <alias> ...` followed by `elytro tx send ...` using the planner’s calldata/UserOp.
5. **Report results** – share bundler hash / transaction hash plus refreshed balances (`elytro query balance ...`) back to the user.

### Rules for swaps

- **Never guess swap outputs.** Always surface the exact minimums, routes, and slippage from `defi/uniswap`.
- **Verify contract addresses** before simulating or sending.
- **Document sponsorship** – note whether `elytro tx simulate` reports `Sponsored: Yes/No` so the user knows who pays gas.
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
  --chain "$CHAIN" \
  --tx "to:$ROUTER,value:$VALUE_ETH,data:$CALLDATA"
elytro tx send demo-arb \
  --chain "$CHAIN" \
  --tx "to:$ROUTER,value:$VALUE_ETH,data:$CALLDATA"
```

---

## Security / 2FA

```bash
# View current security status
elytro security status

# Install SecurityHook (2FA)
elytro security 2fa install --capability 2     # 1=SIGNATURE_ONLY, 2=USER_OP_ONLY, 3=BOTH

# Uninstall SecurityHook
elytro security 2fa uninstall
elytro security 2fa uninstall --force          # Start force-uninstall countdown
elytro security 2fa uninstall --execute        # Execute after safety delay

# Email OTP
elytro security email bind <email>             # Bind email for OTP delivery
elytro security email change <email>           # Change bound email

# Spending limit (USD/day)
elytro security spending-limit                 # View current limit
elytro security spending-limit 100             # Set to $100/day
```

---

## Configuration

```bash
elytro config show                             # Show current endpoint config
elytro config set alchemy-key <apiKey>         # Set Alchemy API key
elytro config set pimlico-key <apiKey>         # Set Pimlico API key
elytro config remove alchemy-key               # Remove key, revert to public endpoint
elytro config remove pimlico-key
```

---

## Common Workflows

### Check ETH balance
```bash
elytro query balance <account>
# Returns JSON: { "result": { "balance": "0.01", "symbol": "ETH", ... } }
```

### Send ETH to an address
```bash
# Step 1: Simulate first (optional but recommended)
elytro tx simulate --tx "to:<toAddress>,value:<amount>,data:0x"

# Step 2: Send — PAUSE at confirmation prompt, ask user before proceeding
elytro tx send --tx "to:<toAddress>,value:<amount>,data:0x"
```

### First-time full setup on Sepolia
```bash
elytro init
elytro account create --chain 0xaa36a7 --alias myWallet
elytro account activate myWallet
elytro account info myWallet    # Verify Deployed: Yes
```

---

## Confirmation Prompt Protocol (Inline Button Version)

Some commands are **interactive** and will pause for user input. **All confirmations MUST use Telegram Inline Buttons, never plain text prompts.**

| Command | Prompt | Action |
|---------|--------|--------|
| `tx send` | `Sign and send this transaction? (y/N)` | **Send Inline Buttons:** `Approve` (callback_data: `tx_approve`), `Reject` (callback_data: `tx_reject`), `Simulate First` (callback_data: `tx_simulate`) |
| `account activate` | activation confirmation | **Send Inline Buttons:** `Activate Now` (callback_data: `confirm_yes`), `Cancel` (callback_data: `confirm_cancel`) |
| `security 2fa uninstall` | uninstall confirmation | **Send Inline Buttons:** `Uninstall 2FA` (callback_data: `confirm_yes`), `Keep 2FA` (callback_data: `confirm_no`) |

**Inline Button Template for Confirmations:**

```json
{
  "text": "Transaction Summary:\n• To: 0x1234...5678\n• Amount: 0.5 ETH\n• Gas: ~0.001 ETH (sponsorship available)\n\nDo you approve this transaction?",
  "reply_markup": {
    "inline_keyboard": [
      [
        { "text": "✅ Approve", "callback_data": "tx_approve" },
        { "text": "❌ Reject", "callback_data": "tx_reject" }
      ],
      [
        { "text": "🔍 Simulate First", "callback_data": "tx_simulate" }
      ]
    ]
  }
}
```

**Rules:**
1. **Never auto-confirm.** Always surface the transaction summary and wait for explicit Inline Button approval.
2. **Callback handling:** Upon receiving a callback (e.g., `tx_approve`), execute the corresponding action and report the result via another message (or update the existing message with the result).
3. **Audit logging:** Log every confirmation decision (approve/reject/simulate) with timestamp, callback_data, and user ID for compliance and troubleshooting.

---

## Agent Implementation Checklist

When implementing Elytro wallet interactions, ensure:

- [ ] **Ambiguous intent detection**: When user says "open wallet", "打开钱包", "elytro", etc., immediately show Main Menu (Level 0) without asking.
- [ ] **Real-time data fetching**: Always run `elytro account list`, `elytro query balance`, etc. before rendering UI. Never use placeholder data.
- [ ] **Inline Button rendering**: All menus, lists, confirmations, and action flows use Telegram Inline Buttons exclusively.
- [ ] **Back button on all sub-menus**: Every screen below Main Menu includes a "⬅️ Back" button.
- [ ] **Callback routing**: Parse callback_data prefixes (menu_*, account_select_*, action_*, tx_*, confirm_*) and route to correct handler.
- [ ] **Multi-step action flows**: For Send and Swap, guide user through account → input → summary → approval steps using Inline Buttons.
- [ ] **Audit logging**: Log all callback events (callback_data, timestamp, user_id, action taken, result).
- [ ] **Error handling**: If a command fails (e.g., `elytro tx send` returns error), display error message with a "⬅️ Back" button to retry or return to menu.

---

## Inter-Agent Communication & Inline Button Convention

**All agents using the Elytro skill must follow this Inline Button standard to ensure seamless integration and user experience consistency.**

### Callback Data Naming Convention
Agents must adhere to the following callback_data format for cross-agent compatibility:

| Use Case | Callback Format | Example |
|----------|-----------------|---------|
| Confirmations | `confirm_yes`, `confirm_no`, `confirm_cancel` | `{ "text": "Approve", "callback_data": "confirm_yes" }` |
| Account Selection | `account_select_<alias_or_address>` | `{ "text": "myWallet", "callback_data": "account_select_myWallet" }` |
| Token Selection | `token_select_<tokenAddress_or_symbol>` | `{ "text": "USDC", "callback_data": "token_select_0xA0b8..." }` |
| Transaction Actions | `tx_approve`, `tx_reject`, `tx_simulate` | `{ "text": "Send", "callback_data": "tx_approve" }` |
| Action Menu | `action_<action_name>` | `{ "text": "Send ETH", "callback_data": "action_send_eth" }` |

### For Other AI Agents Reading This Skill

**Important:** If you are another AI agent tasked with Elytro wallet operations:

1. **Always use Inline Buttons for confirmations and list displays** — never fall back to text input prompts.
2. **Follow the callback_data naming convention above** — this ensures consistency across all agents using Elytro.
3. **When displaying lists** (accounts, tokens, transactions, etc.), render each item as a clickable button with the appropriate callback format.
4. **Upon receiving a callback**, parse the callback_data and execute the corresponding action (e.g., `confirm_yes` → proceed with tx, `account_select_<alias>` → fetch account details).
5. **Log all callback events** for audit purposes: timestamp, user_id, callback_data, action taken, result.
6. **Update this section** if you introduce new callback patterns — keep the standard extensible and well-documented for future agents.

---

## Notes

- Elytro uses **EIP-4337 (Account Abstraction)** — transactions are submitted as UserOps via a Bundler, not regular EOA transactions.
- Sponsored transactions are **gasless by default** when a paymaster is available.
- The wallet data directory is `~/.elytro`.
- `--chain` accepts both hex (`0xaa36a7`) and decimal (`11155111`) chain IDs.
- `data` field in tx spec must be a valid hex string; use `0x` for plain ETH transfers.
- **Inline Button usage is mandatory** for all user-facing confirmations and list displays in Elytro workflows. This ensures a consistent, modern UX across all agents and reduces friction in high-stakes wallet operations.
