---
name: oneshot-research
description: |
  Run deep web research, web search, and read-any-URL-as-markdown with the OneShot SDK, paid in
  USDC via x402. Use when an agent needs a cited research report on a topic, a quick list of
  search results, or the clean text + screenshot of a specific web page. Requires OneShot wallet
  setup — see the `oneshot` skill first.
metadata:
  author: oneshotagent
  version: "2.0.0"
  homepage: "https://oneshotagent.com"
---

# OneShot — Research & Web

Set up auth/wallet via the **`oneshot`** skill, then:

```typescript
import { OneShot } from '@oneshot-agent/sdk';
const agent = await OneShot.create({ cdp: true });
```

## Deep research — `agent.research(options)`

```typescript
const report = await agent.research({
  topic: 'State of AI agent commerce and x402 adoption in 2026',
  depth: 'deep',          // 'quick' | 'deep'
  maxCost: 2.50,
  memo: 'market landscape report',
});
// report.{ report_content, sources: [{ url, title? }], sources_count, topic, depth, report_gcs_uri }
console.log(report.report_content);
report.sources.forEach(s => console.log(s.url));
```

**Options** (`ResearchToolOptions`): `topic` (**required**) and `depth` (`'quick' | 'deep'`, default `deep`).
This is an async job — polls to completion by default; pass `wait: false` for a `request_id`.

## Web search — `agent.webSearch(options)`

```typescript
const out = await agent.webSearch({ query: 'x402 payment protocol spec', max_results: 10 });
// out.results: [{ url, title, description }], out.result_count
```

`max_results` defaults to 5.

## Read a URL — `agent.webRead(options)`

```typescript
const page = await agent.webRead({ url: 'https://example.com/article' });
// page.{ markdown, screenshot_url?, metadata: { title, description, statusCode? }, truncated? }
```

Returns the page rendered to clean markdown plus a screenshot URL — useful for grounding the
agent on a specific source before acting.

## Pricing

| Action | Cost |
|--------|------|
| Research (quick) | ~$0.50 |
| Research (deep) | ~$2.00 |
| Web search | low per-query |
| Web read | low per-page |

Pass `maxCost` on `research` (variable by depth/sources). For broader person-specific
intelligence (dossiers, social profiles, articles about a person), use the `oneshot-enrichment`
skill.
