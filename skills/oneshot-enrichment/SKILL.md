---
name: oneshot-enrichment
description: |
  Find, verify, and enrich people and companies with the OneShot SDK, paid in USDC via x402.
  Use when an agent needs to search for people by title/company/skills, search companies by
  industry/size/funding, enrich a LinkedIn/email/name/domain/ticker into a full profile, find or
  verify a work email, or build deep person intelligence — dossiers, social profiles, articles,
  newsfeed, interests, and follower/following interactions. Requires OneShot wallet setup — see
  the `oneshot` skill first.
metadata:
  author: oneshotagent
  version: "2.1.0"
  homepage: "https://oneshotagent.com"
---

# OneShot — People, Companies & Person Intelligence

Set up auth/wallet via the **`oneshot`** skill, then:

```typescript
import { OneShot } from '@oneshot-agent/sdk';
const agent = await OneShot.create({ cdp: true });
```

## Pick the right source first

These tools index people by corporate role and companies by firmographics. Two audiences are
not in that index at all, and searching for them here costs money and returns nothing:

| Target | Skill |
|---|---|
| Independent local business (contractor, practice, restaurant) | `oneshot-local` |
| Government buyer / federal contract opportunity | `oneshot-gov` |

## People search — `agent.peopleSearch(options)`

```typescript
const found = await agent.peopleSearch({
  job_titles: ['CTO', 'VP Engineering'],
  companies: ['Acme Corp'],
  company_domains: ['acme.com'],
  location: ['San Francisco', 'Remote US'],
  skills: ['Rust', 'distributed systems'],
  seniority: ['executive'],
  industry: ['software'],
  keywords: ['payments'],
  company_size: '51-200',
  limit: 25,
});
// found.{ results: PersonResult[], total_found }
```

All filters optional; combine to narrow. `PersonResult` includes name, title, company,
`linkedin_url`, `email`, `phone`, `skills`, `experience`, `education`, normalized
`best_work_email`/`best_personal_email`.

## Enrich a profile — `agent.enrichProfile(options)`

```typescript
const { profile } = await agent.enrichProfile({
  linkedin_url: 'https://linkedin.com/in/janedoe', // or email / name + company_domain
});
```

Options: `linkedin_url`, `email`, `name`, `company_domain` (provide what you have).

## Find email — `agent.findEmail(options)`

```typescript
const res = await agent.findEmail({
  full_name: 'Jane Doe',        // or first_name + last_name
  company_domain: 'acme.com',   // REQUIRED
});
// res.{ email, found, full_name?, company_domain? }
```

## Verify email — `agent.verifyEmail(options)`

```typescript
const v = await agent.verifyEmail({ email: 'jane@acme.com' });
// v.{ valid, deliverable, catch_all, disposable }
```

Pair with the `oneshot-email` skill: verify before sending to protect deliverability.

## Company search — `agent.companySearch(options)`

```typescript
const found = await agent.companySearch({
  industry: ['software', 'fintech'],
  location: ['San Francisco', 'New York'],
  size: '51-200',                 // or min_employee_count / max_employee_count
  funding_stage: 'series_b',
  tags: ['payments'],
  name: 'Acme',                   // narrow by name or domain when you have one
  domain: 'acme.com',
  limit: 10,                      // default 10
});
```

All filters optional. Use it to build a target account list, then `peopleSearch` with
`company_domains` to find the humans inside the accounts worth pursuing — that order costs
less than searching people first and filtering companies after.

## Enrich a company — `agent.enrichCompany(options)`

```typescript
const company = await agent.enrichCompany({ domain: 'acme.com' });
// or { name } / { linkedin_url } / { ticker: 'CRM' }
```

At least one of `domain`, `name`, `linkedin_url`, or `ticker` is required — the SDK throws a
`ValidationError` client-side before spending if you pass none. `domain` is the most reliable
identifier; `name` alone is ambiguous for anything but a well-known brand.

## Deep person intelligence

All take at least one identifier (email, `social_media_url`, or name/company) and return rich
structured results. Each is a paid async tool.

```typescript
// Full dossier: career, interests, connections (~2–5 min)
await agent.deepResearchPerson({ email: 'jane@acme.com', name: 'Jane Doe', company: 'Acme' });

// Discover all social accounts (LinkedIn, X, GitHub, YouTube, …)
await agent.socialProfiles({ email: 'jane@acme.com' }); // or { social_media_url }

// Articles / publications about a person
await agent.articleSearch({ name: 'Jane Doe', company: 'Acme', sort: 'recent', limit: 10 });

// Recent social posts with engagement metrics
await agent.personNewsfeed({ social_media_url: 'https://x.com/janedoe' });

// Interest analysis across categories
await agent.personInterests({ social_media_url: 'https://x.com/janedoe' }); // or { email } / { phone }

// Map followers / following / replies
await agent.personInteractions({ social_media_url: 'https://x.com/janedoe', type: 'followers,following', max_results: 100 });
```

> Phone numbers sometimes arrive via an async webhook *after* enrichment completes. By default
> these tools return as soon as the profile is ready (`phones_pending: true`). To keep polling
> until phones land, pass `waitForPhones: true` (and optionally `phoneTimeoutSec`, default 360s).

## Pricing

These are paid per-call (people search scales with the number of results). See current per-tool
pricing at https://docs.oneshotagent.com/pricing. Set `maxCost` on `peopleSearch` (scales with
`limit`) and the deep-intel tools.
