---
name: oneshot-browser
description: |
  Run autonomous browser tasks with the OneShot SDK, paid in USDC via x402. Use when an agent needs
  to navigate websites, fill forms, extract structured data, or operate logged-in sessions — with
  persistent profiles that retain cookies/localStorage and domain-scoped secrets for auto-login.
  Requires OneShot wallet setup — see the `oneshot` skill first.
metadata:
  author: oneshotagent
  version: "2.0.0"
  homepage: "https://oneshotagent.com"
---

# OneShot — Browser Automation

Set up auth/wallet via the **`oneshot`** skill, then:

```typescript
import { OneShot } from '@oneshot-agent/sdk';
const agent = await OneShot.create({ cdp: true });
```

## Run a browser task — `agent.browser(options)`

```typescript
const res = await agent.browser({
  task: 'Go to the pricing page and extract every plan name and monthly price.', // min 10 chars
  start_url: 'https://example.com',
  output_schema: {                       // optional — force structured extraction
    type: 'object',
    properties: { plans: { type: 'array', items: { type: 'object' } } },
  },
  allowed_domains: ['example.com'],      // optional — restrict navigation
  max_steps: 40,                         // optional, default 50, max 100
  maxCost: 1.50,
  memo: 'scrape pricing',
});
// res.{ output (string | object), steps: [{ number, goal, url }], session_id, output_files? }
```

**Options** (`BrowserTaskOptions`):

| Field | Notes |
|-------|-------|
| `task` | **Required**, min 10 chars. Natural-language instruction. |
| `start_url` | Initial URL. |
| `output_schema` | JSON schema for structured output extraction. |
| `allowed_domains` | Restrict browsing to these domains. |
| `session_id` | Reuse an existing browser session. |
| `profile_id` | Reuse a persistent profile (cookies/localStorage) — see below. |
| `secrets` | Domain-scoped creds for auto-login, e.g. `{ "github.com": "user:token" }`. |
| `max_steps` | Default 50, max 100. |

## Persistent profiles (logged-in sessions)

Create a profile once, then pass its `id` as `profile_id` to reuse cookies/localStorage across
tasks — e.g. stay logged in to a dashboard.

```typescript
const profile = await agent.createBrowserProfile('acme-dashboard'); // { id, name } — free
const profiles = await agent.listBrowserProfiles();                 // BrowserProfile[] — free
await agent.deleteBrowserProfile(profile.id);                       // free

await agent.browser({ task: 'Open the dashboard and download this month\'s invoice.', profile_id: profile.id });
```

Profiles are scoped to your wallet — you only see and can attach your own. The SDK signs a read proof (`x-agent-proof`) so the API can enforce that ownership; automatic with `@oneshot-agent/sdk` ≥ 0.25.0.

Profile create/list/delete are **free**; running a task is paid (priced by estimated steps).
Always set `maxCost`.

## Pricing

Browser tasks are paid and scale with steps (guard with `maxCost`); profile create/list/delete
are free. See current per-tool pricing at https://docs.oneshotagent.com/pricing.
