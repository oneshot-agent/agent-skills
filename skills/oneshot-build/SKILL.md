---
name: oneshot-build
description: |
  Generate and deploy real websites with the OneShot SDK, paid in USDC via x402. Use when an agent
  needs to build a landing page, SaaS site, portfolio, funnel, or other site type from a product
  description — with optional brand styling, lead capture, source-URL inspiration, and a custom
  domain — then update it later. Requires OneShot wallet setup — see the `oneshot` skill first.
metadata:
  author: oneshotagent
  version: "2.0.0"
  homepage: "https://oneshotagent.com"
---

# OneShot — Website Build

Set up auth/wallet via the **`oneshot`** skill, then:

```typescript
import { OneShot } from '@oneshot-agent/sdk';
const agent = await OneShot.create({ cdp: true });
```

## Build a site — `agent.build(options)`

```typescript
const site = await agent.build({
  type: 'saas',  // 'saas' | 'portfolio' | 'agency' | 'personal' | 'product' | 'funnel' | 'restaurant' | 'event'
  product: {
    name: 'InboxZero AI',
    description: 'An AI assistant that triages and replies to your email automatically.', // min 10 chars
    industry: 'productivity',
    pricing: '$19/mo',
  },
  brand: { primary_color: '#4F46E5', font: 'Inter', tone: 'professional' }, // tone: professional|playful|bold|minimal
  sections: ['hero', 'features', 'pricing', 'faq'],
  lead_capture: { enabled: true, inbox_email: 'leads@myco.com' },  // inbox_email defaults to agent inbox
  images: { hero: 'https://.../hero.png', logo: 'https://.../logo.svg' },
  source_url: 'https://competitor.com',   // analyze for content/inspiration
  domain: 'inboxzero.ai',                  // optional custom domain
  maxCost: 30.00,
  memo: 'launch landing page',
});
// site.{ success, production_url?, preview_url?, design_score?, iterations? }
```

**Options** (`BuildOptions`): `product` is **required** (`name` + `description` ≥10 chars,
optional `industry`/`pricing`). `type` defaults to an inferred type. `brand`, `sections`,
`lead_capture`, `images`, `source_url`, `domain` all optional. Pricing varies with sections,
AI images, lead capture, source analysis, and custom domain — **always pass `maxCost`**.

## Update a site — `agent.updateBuild(options)`

```typescript
const updated = await agent.updateBuild({
  build_id: 'build_abc123',     // REQUIRED
  product: { name: 'InboxZero AI', description: 'Now with calendar scheduling built in.' },
  sections: ['hero', 'features', 'pricing', 'integrations', 'faq'],
  maxCost: 15.00,
});
```

`updateBuild` requires `build_id` and `product`; other fields optional and default to the
existing build. Updates are discounted vs a fresh build. Both are async jobs (poll to completion
by default).

## Pricing

| Action | Cost |
|--------|------|
| Build website | ~$10+ (scales with sections, AI images, lead capture, domain) |
| Update build | Discounted vs new build |
