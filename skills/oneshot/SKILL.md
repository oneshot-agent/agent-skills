---
name: oneshot
description: |
  Set up and authenticate the OneShot SDK/MCP so an AI agent can execute real-world,
  paid actions — email, SMS, voice calls, research, person enrichment, commerce, browser
  automation, website builds, and autonomous compute goals — settled in USDC via the x402
  protocol on Base. Use this skill FIRST to install, choose a wallet (Coinbase CDP or raw
  private key), fund the agent, and understand shared options (maxCost, wait, idempotency).
  Then load the capability-specific skills: oneshot-email, oneshot-messaging, oneshot-research,
  oneshot-enrichment, oneshot-commerce, oneshot-browser, oneshot-build, oneshot-compute.
metadata:
  author: oneshotagent
  version: "2.0.0"
  homepage: "https://oneshotagent.com"
---

# OneShot — Setup & Core

OneShot is infrastructure for autonomous AI agents to execute real-world commercial actions.
Every paid call settles in **USDC on Base** using the **x402** payment protocol — the agent's
wallet signs an EIP-3009 transfer authorization, the server verifies it, runs the tool, and
settles on-chain. You hold a wallet with USDC; the SDK handles quoting, signing, and settlement.

This is the **core setup skill**. Install once, then use the focused skills for each capability:

| Skill | Covers |
|-------|--------|
| `oneshot-email` | Send email, inbox, sending-domain pool & warmup |
| `oneshot-messaging` | SMS send/inbox, autonomous voice calls |
| `oneshot-research` | Deep research, web search, read-a-URL |
| `oneshot-enrichment` | People search, profile enrichment, find/verify email, person intelligence |
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
| `idempotencyKey?: string` | Replay protection (24h). Currently honored by `email`; other tools ignore it until their routes opt in. |

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
        "CDP_WALLET_SECRET": "your-wallet-secret"
      }
    }
  }
}
```

(Use `ONESHOT_WALLET_PRIVATE_KEY` instead of the `CDP_*` vars for raw-key auth.) The server
exposes the same tools as the SDK, namespaced `oneshot_<action>` (e.g. `oneshot_email`,
`oneshot_research`, `oneshot_commerce_buy`).

## Resources

- Docs: https://docs.oneshotagent.com
- Pricing: https://docs.oneshotagent.com/pricing
- Soul.Markets (monetize an agent): https://soul.mds.markets — see the `soul-markets` skill
