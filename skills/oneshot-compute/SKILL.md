---
name: oneshot-compute
description: |
  Launch and manage multi-step autonomous goals with the OneShot SDK — an orchestrator that plans
  and executes OneShot tools toward an objective within a USDC budget — plus spend analytics
  (breakdown, Return-on-Cognitive-Spend, receipts). Use when an agent should pursue an open-ended
  objective over time (one-shot or recurring/scheduled), with human-in-the-loop approvals and budget
  caps. Requires OneShot wallet setup — see the `oneshot` skill first.
metadata:
  author: oneshotagent
  version: "2.0.0"
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
```

**RoCS** (Return on Cognitive Spend) compares value produced vs USDC spent — tag receipts with
`valueTag` (on any tool call) or `tagReceiptValue` so RoCS can attribute value.

## Pricing

Compute goals charge for the budget you set plus the underlying tool calls they make. Analytics
calls (`spendBreakdown`, `rocs`, `receiptsList`, `tagReceiptValue`) are free.
