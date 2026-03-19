# Elytro Command Reference

Full command reference. See SKILL.md for rules, workflows, and error recovery.

## Account

| Command | Returns |
|---------|---------|
| `account create -c <chainId> [-a <alias>] [-e <email>] [-l <dailyLimitUsd>]` | `{ alias, address, deployed: false, security }` |
| `account activate [alias\|address] [--no-sponsor]` | `{ transactionHash, hookInstalled, emailPending, dailyLimitPending }` |
| `account list [-c <chainId>]` | `{ accounts, total }` |
| `account info [alias\|address]` | `{ deployed, balance, securityStatus }` |
| `account rename <alias\|address> <newAlias>` | `{ alias, address }` |
| `account switch <alias\|address>` | Always pass alias/address; no interactive selector |

## Transaction

| Command | Returns |
|---------|---------|
| `tx send [account] --tx <spec> [--no-sponsor] [--no-hook]` | `status: "confirmed"` or `"otp_pending"` or `"cancelled"` |
| `tx build [account] --tx <spec> [--no-sponsor]` | Unsigned UserOp |
| `tx simulate [account] --tx <spec> [--no-sponsor]` | `{ gas, sponsored, balance, warnings }` |

`--tx` spec: `to:0xAddr,value:0.1,data:0x...`. `to` required; `value` or `data` required. Multiple `--tx` = batch.

## Query

| Command | Returns |
|---------|---------|
| `query balance [alias\|address] [--token 0xAddr]` | ETH or ERC-20 balance |
| `query tokens [alias\|address]` | All ERC-20 holdings |
| `query tx <hash>` | Transaction receipt |
| `query chain` | Current chain |
| `query address <0xAddress>` | Address type + balance |

## Security

| Command | Returns |
|---------|---------|
| `security status` | `{ hookInstalled, profile: { email, emailVerified, dailyLimitUsd } }` |
| `security 2fa install [--capability 1\|2\|3]` | Install hook |
| `security 2fa uninstall [--force [--execute]]` | Deferred OTP if normal path |
| `security email bind <email>` | `otp_pending` → user runs `otp submit` |
| `security email change <email>` | `otp_pending` → user runs `otp submit` |
| `security spending-limit [usd]` | View or set; set returns `otp_pending` |

## OTP

| Command | Returns |
|---------|---------|
| `otp submit <id> <code>` | Completes pending; `id` from `otpPending.id`. Current account must match initiator. |
| `otp list` | Pending OTPs for current account |
| `otp cancel [id]` | Cancel; omit id = all for current account |

## Config & Update

| Command | Returns |
|---------|---------|
| `config show` | Current config |
| `config set alchemy-key\|pimlico-key <KEY>` | Save key |
| `config remove <key>` | Remove key |
| `update check` | `{ updateAvailable, upgradeCommand }` |
| `update` | Install latest |

## Error Codes

| Code | Meaning |
|------|---------|
| -32000 | Internal error |
| -32002 | Account not ready |
| -32005 | Send failed |
| -32007 | Hook auth failed |
| -32010 | Email not bound |
| -32012 | OTP verification failed |
| -32013 | OTP id not found |
| -32014 | OTP expired |
| -32602 | Invalid parameters |
