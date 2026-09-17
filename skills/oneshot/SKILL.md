---
name: oneshot
description: |
  Set up and authenticate the OneShot SDK/MCP so an AI agent can execute real-world,
  paid actions — email, SMS, voice calls, research, person enrichment, commerce, browser
  automation, website builds, and autonomous compute goals — settled in USDC via the x402
  protocol on Base. Use this skill FIRST to install, choose a wallet (Coinbase CDP or raw
  private key), fund the agent, set spend budgets, and understand shared options (maxCost, wait,
  idempotency). Then load the capability-specific skills: oneshot-email, oneshot-messaging,
  oneshot-research, oneshot-enrichment, oneshot-local, oneshot-gov, oneshot-commerce,
  oneshot-physical-mail, oneshot-browser, oneshot-build, oneshot-compute.
metadata:
  author: oneshotagent
  version: "2.2.0"
  homepage: "https://oneshotagent.com"
---

# OneShot — Setup & Core

OneShot is infrastructure for autonomous AI agents to execute real-world commercial actions.
There are two ways to pay for a call, and this SDK uses the first:

- **x402 — USDC on Base.** The agent's wallet signs an EIP-3009 transfer authorization, the
  server verifies it, runs the tool, and settles on-chain. You hold a wallet with USDC; the
  SDK handles quoting, signing, and settlement.
- **Stripe.** Agents that speak Stripe's **Agentic Commerce Protocol** buy the same tool
  catalog with a card — discover products at `/.well-known/acp/manifest.json`, open a checkout
  session, complete it with a SharedPaymentToken, and the tool runs. Priced in USD, no wallet
  and no crypto anywhere. That is a direct HTTP integration, not this SDK.

Both rails spend a **prepaid credit balance first** when the agent has one, and a fixed-price
call fully covered by credits skips the payment step entirely — so a `getBalance()` of zero
does not always mean a call will fail.

This is the **core setup skill**. Install once, then use the focused skills for each capability:

| Skill | Covers |
|-------|--------|
| `oneshot-email` | Send email, inbox, sending-domain pool & warmup |
| `oneshot-messaging` | SMS send/inbox, autonomous voice calls |
| `oneshot-research` | Deep research, web search, read-a-URL |
| `oneshot-enrichment` | People & company search, profile/company enrichment, find/verify email, person intelligence |
| `oneshot-local` | Local business discovery and name+address → domain/phone resolution |
| `oneshot-gov` | Federal solicitations (SAM.gov) with the contracting officer's contact |
| `oneshot-physical-mail` | Printed letters and postcards — artwork, address validation, approval, tracking |
| `oneshot-commerce` | Product search and autonomous purchase |
| `oneshot-browser` | Autonomous browser tasks + persistent profiles |
| `oneshot-build` | Generate & deploy websites |
| `oneshot-compute` | Multi-step autonomous goals with a budget + spend analytics |

## Install

```bash
npm install @oneshot-agent/sdk
```

For use inside an MCP client (Claude Desktop, Claude Code, Cursor, …) instead of the SDK, see
[MCP Server](#mcp-server) below.

## Authentication (pick one wallet)

OneShot needs a wallet to sign x402 payments. Two options:

> **Read endpoints** (inbox, SMS inbox, notifications, balance, browser profiles) return private, per-agent data. The SDK automatically signs a short-lived EIP-712 **read proof** (`x-agent-proof` header) on each read so the API can confirm you control the wallet — no extra code. Use `@oneshot-agent/sdk` ≥ 0.25.0 (or `oneshot-python` ≥ 0.17.0).

### Option A — Coinbase CDP Wallet (recommended, no private keys)

The wallet is managed server-side by Coinbase; signing happens in a secure enclave. Set:

```bash
export CDP_API_KEY_ID="your-api-key-id"
export CDP_API_KEY_SECRET="your-api-key-secret"
export CDP_WALLET_SECRET="your-wallet-secret"
```

```typescript
import { OneShot } from '@oneshot-agent/sdk';

// async factory — reads CDP_* from env, auto-creates a wallet on first use
const agent = await OneShot.create({ cdp: true });
console.log(agent.address); // the agent's Base address
```

Get credentials at the [Coinbase Agentic Wallet](https://docs.cdp.coinbase.com/agentic-wallet/welcome).
Optional peer dep for CDP: `npm install @coinbase/cdp-sdk`.

### Option B — Raw private key (advanced)

```bash
export ONESHOT_WALLET_PRIVATE_KEY="0xYourPrivateKey"
```

```typescript
import { OneShot } from '@oneshot-agent/sdk';

const agent = new OneShot({ privateKey: process.env.ONESHOT_WALLET_PRIVATE_KEY });
```

### Option C — Bring your own signer

```typescript
const agent = await OneShot.create({
  walletProvider: {
    address: '0x...',
    signTypedData: async (domain, types, value) => '0xsignature',
  },
});
```

### Config options

`OneShot.create(config)` / `new OneShot(config)` accept:

| Field | Purpose |
|-------|---------|
| `privateKey` | Raw key (Option B) |
| `cdp` | `true` or `{ address }` for CDP (Option A) |
| `walletProvider` | Custom signer (Option C) |
| `baseUrl` | Override API URL (default `https://win.oneshotagent.com`) |
| `rpcUrl` | Override Base RPC |
| `currency` | `'USDC'` (default) or `'ETH'` (auto-swaps ETH→USDC via Uniswap V3 before paying) |
| `slippage` | Swap slippage when `currency: 'ETH'` (default `0.01` = 1%) |
| `debug` / `logger` | Verbose logging |

## Funding the agent

Paid tools draw USDC from the agent's Base wallet:

1. Get the address: `console.log(agent.address)`
2. Send USDC (Base mainnet, chain 8453) to that address, or fund via https://oneshotagent.com
3. Check balances:

```typescript
const usdc = await agent.getBalance();         // returns a string, e.g. "12.50"
const unified = await agent.getUnifiedBalance(); // { on_chain_balance, credits_balance, currency, address, chain_id }
```

## Spend budgets (set these before the agent runs unattended)

`maxCost` caps one call. A budget caps the agent. Set both — an agent in a retry loop can
spend a lot of small, individually-reasonable amounts.

```bash
export ONESHOT_BUDGET_DAILY="25"            # max USDC per UTC day
export ONESHOT_BUDGET_PER_TRANSACTION="2"   # max USDC for any single call
export ONESHOT_BUDGET_ALERT_AT="0.8"        # warn at 80% of daily (default 0.8)
export ONESHOT_BUDGET_PAUSE_AT="1.0"        # stop paid calls at 100% (default 1.0)
export ONESHOT_BUDGET_ALERT_EMAIL="you@example.com"
```

Caps are enforced **server-side**, not in the client: once the daily cap is reached, paid
calls are rejected with `budget_exceeded` no matter what the caller does. Read the current
state at any time:

```typescript
const b = await agent.budgets();
// { daily_usdc, per_transaction_usdc, alert_at, pause_at,
//   spent_today_usdc, remaining_usdc, pct_used, resets_at }
```

Two things worth knowing:

- **A blank value is treated as unset**, deliberately — `${VAR}` templating in Docker Compose
  and CI yields `""` for an unset variable, and failing startup there would break every
  deployment that templates optional vars. A *set but invalid* value (`"abc"`, `-5`) fails
  startup loudly rather than silently becoming "no cap".
- **In the MCP server, the budget is read-only to the model.** The only budget tool exposed is
  `oneshot_budget_status`. There is no tool to raise a cap — a guardrail the agent can lift
  itself is not a guardrail. Set the limits in the environment, where the model cannot reach
  them.

## Shared options (every paid tool)

All paid methods accept these on their options object:

| Option | Meaning |
|--------|---------|
| `maxCost?: number` | Hard ceiling — fails fast client-side before signing if the quote exceeds it. **Always set this on variable-price tools** (voice, commerce, build, browser, research). |
| `wait?: boolean` | Poll an async job to completion (default `true`). Set `false` to get a `request_id` back immediately. |
| `timeout?: number` | Client timeout in seconds. |
| `onStatusUpdate?: (status, requestId) => void` | Progress callback for long jobs. |
| `signal?: AbortSignal` | Cancel before payment is signed. |
| `memo?: string` | Human-readable reason, stored on the receipt for audit (≤1000 chars). |
| `valueTag?: { type, amount?, label? }` | Tag the receipt's value for RoCS analytics. |
| `decisionContext?: { goal?, goalId?, alternatives?, confidence? }` | Machine-readable why, for supervisor agents. |
| `idempotencyKey?: string` | Replay protection (24h). Honored by `email` and the durable enrichment endpoints — `enrichProfile`, `findEmail`, `verifyEmail` — where the SDK attaches one automatically so a timed-out call can be recovered rather than repaid. Other tools ignore it until their routes opt in. |

Example with guards:

```typescript
const res = await agent.research(
  { topic: 'agent commerce in 2026', depth: 'deep' },
  // options below are merged onto the same object:
);
// In practice pass them together:
await agent.email({ to: 'x@y.com', subject: 'Hi', body: 'Hello', maxCost: 0.05, memo: 'cold outreach reply' });
```

## Universal tool call

Any endpoint can be called generically — useful for tools newer than your SDK version:

```typescript
const result = await agent.tool('email', { to: 'user@example.com', subject: 'Hi', body: 'Hello' });
```

## Error handling

```typescript
import {
  OneShot,
  OneShotError,        // base class
  ToolError,           // tool returned an error
  JobError,            // async job failed
  JobTimeoutError,     // job didn't finish before timeout
  ValidationError,     // bad input
  ContentBlockedError, // content policy violation
  EmergencyNumberError // voice/sms to an emergency number
} from '@oneshot-agent/sdk';

try {
  await agent.email({ to, subject, body });
} catch (err) {
  if (err instanceof ContentBlockedError) {/* rephrase */}
  else if (err instanceof JobTimeoutError) {/* retry with idempotencyKey */}
  else if (err instanceof OneShotError) {/* generic handling */}
}
```

> Note: there is no `InsufficientBalanceError` — a low balance surfaces as a `ToolError`/`OneShotError`.
> Check `getBalance()` before expensive calls instead.

### Recovering a timed-out call

A `JobTimeoutError` means your client stopped waiting — not that the work stopped, and not
that you were not charged. Blindly retrying pays twice. For the reliability-tracked endpoints
(`enrich/profile`, `enrich/email`, `verify/email`) the SDK attaches an idempotency key
automatically; recover the original request with it instead of re-running:

```typescript
const key = crypto.randomUUID();
try {
  await agent.enrichProfile({ linkedin_url: url, idempotencyKey: key });
} catch (err) {
  if (err instanceof JobTimeoutError) {
    const rec = await agent.recoverRequest({ endpoint: 'enrich/profile', idempotencyKey: key });
    // rec.{ request_id, receipt_id, status, settlement_status }
  }
}
```

Before assuming a failure is yours, check whether it is ours:

```typescript
const status = await agent.getServiceStatus(); // { status: 'healthy' | 'degraded', observed_at, dependencies }
```

`degraded` means an upstream vendor is struggling. Retrying into a degraded dependency burns
budget on empty results — wait it out instead.

## Notifications

Free, read-only, and how the platform tells the agent about things it did not ask for —
budget alerts, domain warmup state changes, job failures.

```typescript
const { notifications } = await agent.notifications({ unread: true, limit: 50 });
await agent.markNotificationRead(notificationId);
```

Poll these on a long-running agent rather than discovering a blocked sending domain from a
run of failed sends.

## Test mode

By default the SDK targets production (Base mainnet, real USDC). To experiment on Base Sepolia
testnet, point `baseUrl` at a staging deployment and use a testnet-funded wallet.

## MCP Server

To expose OneShot tools inside an MCP client instead of importing the SDK:

```bash
npm install -g @oneshot-agent/mcp-server
```

Add to your client config (Claude Desktop `claude_desktop_config.json`, Claude Code
`~/.claude/settings.json`, or Cursor `.cursor/mcp.json`):

```json
{
  "mcpServers": {
    "oneshot": {
      "command": "npx",
      "args": ["-y", "@oneshot-agent/mcp-server"],
      "env": {
        "CDP_API_KEY_ID": "your-api-key-id",
        "CDP_API_KEY_SECRET": "your-api-key-secret",
        "CDP_WALLET_SECRET": "your-wallet-secret",
        "ONESHOT_BUDGET_DAILY": "25",
        "ONESHOT_BUDGET_PER_TRANSACTION": "2"
      }
    }
  }
}
```

(Use `ONESHOT_WALLET_PRIVATE_KEY` instead of the `CDP_*` vars for raw-key auth.) The server
exposes the same tools as the SDK, namespaced `oneshot_<action>` (e.g. `oneshot_email`,
`oneshot_research`, `oneshot_commerce_buy`) — 61 of them. Set the `ONESHOT_BUDGET_*` vars
here: an MCP client hands the tools to a model, and the model cannot raise a cap it can only
read through `oneshot_budget_status`.

For Cursor specifically, the **OneShot Agent plugin** in the Cursor marketplace ships this
config plus spend-safety rules, workflow skills, and a read-only research subagent:
https://github.com/tormine/oneshot-cursor-plugin

### Hosted endpoint (Grok Bot, cloud runners, any client without a local process)

The local server above signs x402 payments with your wallet, so it needs a process on your
machine. Agents that run in someone else's cloud — **Grok Bot**, hosted runners, agent
platforms — use the same 61 tools over Streamable HTTP at
`https://win.oneshotagent.com/mcp`, authenticated with an **agent access token** and billed
to the agent's prepaid **credits** (never x402, never a wallet key in a third-party cloud).

1. Mint a token from any wallet session of the SDK (≥ 0.33.0):

   ```typescript
   const { token, id } = await agent.createAccessToken({ name: 'grok-bot' });
   // token is oneshot_… — shown once; agent.revokeAccessToken(id) kills it
   ```

2. Add credits to the agent (operator grant today; self-serve top-up is not yet available).
3. In the client, add an MCP server with URL `https://win.oneshotagent.com/mcp` and header
   `Authorization: Bearer oneshot_…`. For Grok Bot that is **Add server** → name `oneshot`,
   the URL, the header — no OAuth step.

A token session can only spend credits, under the agent's stored `ONESHOT_BUDGET_*`-equivalent
caps (set from a wallet session; read-only from the token via `oneshot_budget_status`). A call
the balance cannot cover returns `insufficient_credits` with the shortfall; a call over the
daily cap returns `budget_exceeded` and debits nothing. Full guide:
https://docs.oneshotagent.com/sdk/remote-mcp

## Resources

- Docs: https://docs.oneshotagent.com
- Pricing: https://docs.oneshotagent.com/pricing
- Soul.Markets (monetize an agent): https://soul.mds.markets — see the `soul-markets` skill
