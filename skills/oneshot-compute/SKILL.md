---
name: oneshot-compute
description: |
  Launch and manage multi-step autonomous goals with the OneShot SDK — an orchestrator that plans
  and executes OneShot tools toward an objective within a USDC budget — plus spend analytics
  (breakdown, Return-on-Cognitive-Spend, receipts). Use when an agent should pursue an open-ended
  objective over time (one-shot or recurring/scheduled), with human-in-the-loop approvals and budget
  caps. Requires OneShot wallet or access-token setup — see the `oneshot` skill first.
metadata:
  author: oneshotagent
  version: "2.2.0"
  homepage: "https://oneshotagent.com"
---

# OneShot — Compute Goals & Analytics

Set up auth/wallet via the **`oneshot`** skill, then:

```typescript
import { OneShot } from '@oneshot-agent/sdk';
const agent = await OneShot.create({ cdp: true });
```

## Launch a goal — `agent.compute(options)`

```typescript
const goal = await agent.compute({
  objective: 'Find 20 qualified leads at Series-B fintechs, enrich them, and email a tailored intro.',
  budget_usdc: 25,                 // server estimates if omitted
  deadline: '2026-07-01T00:00:00Z',// optional ISO deadline
  params: { region: 'US', titles: ['Head of Payments'] }, // optional constraints
  maxCost: 25,
  memo: 'lead-gen campaign',
});
// goal.{ goal_id, status, goal: { objective, budget_usdc, deadline?, ... } }
```

The orchestrator plans phases/tasks and executes OneShot tools (research, enrichment, email, …)
against the budget. Optional routing: `soul_slug` / `soul_service_slug` to route to a Soul agent
(see the `soul-markets` skill).

### Recurring goals

```typescript
await agent.compute({
  objective: 'Daily digest of new x402 merchants, emailed to me.',
  schedule: { cron: '0 9 * * *', budget_per_run: 2, max_runs: 30 }, // min interval 15 min
});
```

## Monitor & control

```typescript
const status = await agent.getComputeGoal(goalId);   // phase, plan, budget { total, spent, reserved, remaining }
const tasks  = await agent.getComputeTasks(goalId);   // ComputeTask[] with progress + results
const budget = await agent.getComputeBudget(goalId);  // detailed spend entries

await agent.pauseComputeGoal(goalId, 'budget review');
await agent.resumeComputeGoal(goalId);
await agent.cancelComputeGoal(goalId, 'no longer needed'); // returns remaining_budget
await agent.fundComputeGoal(goalId, 10);                   // top up budget (paid)
```

### From an MCP client

Every one of those is also an MCP tool, so a client on `@oneshot-agent/mcp-server`
reaches them without importing the SDK:

| SDK | MCP tool | |
|---|---|---|
| `getComputeGoal` | `oneshot_compute_status` | free |
| `getComputeTasks` | `oneshot_compute_tasks` | free |
| `pauseComputeGoal` | `oneshot_compute_pause` | free |
| `resumeComputeGoal` | `oneshot_compute_resume` | free |
| `cancelComputeGoal` | `oneshot_compute_cancel` | free |
| `respondToComputeTask` | `oneshot_compute_respond` | free |
| `fundComputeGoal` | `oneshot_compute_fund` | **paid** |

`oneshot_compute_pause`, `_resume` and `_fund` need **mcp-server ≥ 0.21.0**; older
versions can start and cancel a goal but have no move in between.

Free does not mean harmless here. `oneshot_compute_cancel` ends the goal and its
plan — the budget refunds, the work does not — and `oneshot_compute_respond`
answers a human-in-the-loop question *as the developer*, unblocking paid work.
Confirm both before calling them. `oneshot_compute_fund` moves USDC into the goal
and quotes before charging, the same confirm-before-spend contract as every other
paid tool.

### Human-in-the-loop

When a task needs approval, respond:

```typescript
await agent.respondToComputeTask(goalId, {
  task_id: 'task_123',
  approved: true,
  response: 'Yes, proceed with the $200 ad spend.',
});
```

## Spend analytics

```typescript
const breakdown = await agent.spendBreakdown({ period: 30 });   // categories[], total, period_days
const rocs       = await agent.rocs({ period: 30 });             // { rocs, total_spend, total_value, period_days }
const receipts   = await agent.receiptsList({ period: 30, category: 'email', limit: 50 });
await agent.tagReceiptValue('receipt_id', { type: 'revenue', amount: 500, label: 'closed deal' });

// Return on compute spend broken down per goal — which objectives actually paid off
const byGoal = await agent.rocsByGoal({ period: 30 });        // all goals in the window
const one    = await agent.rocsByGoal({ goalId: 'goal_...' }); // a single goal
```

**RoCS** (Return on Cognitive Spend) compares value produced vs USDC spent — tag receipts with
`valueTag` (on any tool call) or `tagReceiptValue` so RoCS can attribute value.

## Pricing

Compute goals charge for the budget you set plus the underlying tool calls they make; analytics
calls (`spendBreakdown`, `rocs`, `receiptsList`, `tagReceiptValue`) are free. See current
per-tool pricing at https://docs.oneshotagent.com/pricing.

### Funding with an access token

`fundComputeGoal` and `oneshot_compute_fund` also work in token sessions when prepaid
credits cover the full top-up quote. Stored spending caps apply. If credits are
insufficient, refill from a wallet session with `topUpCredits` and retry; the token
cannot sign an on-chain payment or buy credits. Keep wallet credentials out of hosted
clients such as Grok Bot. See the core `oneshot` skill for setup.
