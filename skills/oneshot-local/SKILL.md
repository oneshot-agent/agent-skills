---
name: oneshot-local
description: |
  Discover and resolve local main-street businesses with the OneShot SDK — restaurants,
  contractors, dental practices, auto shops — by category × location, and turn a business
  name plus an address into a website domain, phone number, and operating status. Use when
  the target is an independent local business rather than a corporate employee, where
  people search returns nothing useful. Requires OneShot wallet setup — see the `oneshot`
  skill first.
metadata:
  author: oneshotagent
  version: "1.0.0"
  homepage: "https://oneshotagent.com"
---

# OneShot — Local Business Discovery

Set up auth/wallet via the **`oneshot`** skill, then:

```typescript
import { OneShot } from '@oneshot-agent/sdk';
const agent = await OneShot.create({ cdp: true });
```

## When to use this instead of `peopleSearch`

`peopleSearch` (the `oneshot-enrichment` skill) indexes people by corporate role. The owner
of a two-truck HVAC company is not in that index under "CEO" — they are in a local business
directory under "hvac contractor, Austin TX". If your target has a storefront, a service
area, or a Google listing rather than a LinkedIn title, use these tools.

## Discover businesses — `agent.localSearch(options)`

Flat price per search, not per row: a 200-row search costs the same as a 5-row one, so ask
for the whole area at once rather than paginating by hand.

```typescript
const found = await agent.localSearch({
  category: ['hvac contractor', 'plumber'],  // or keywords: ['emergency ac repair']
  location: ['Austin, TX'],                   // required
  min_rating: 4.0,
  min_review_count: 10,
  is_chain: false,          // exclude detected chains — undetected rows survive this filter
  operating_status: 'open', // default; 'any' includes closed businesses
  has_domain: true,         // only rows with a resolvable website, i.e. contactable
  limit: 100,               // 1–500, default 100
});
// found.{ results: LocalResult[], total_found, truncated, vendor_calls }
```

`category` or `keywords` is required, and so is `location`. Each `LocalResult` carries:

| Field | Notes |
|---|---|
| `id` | Stable hash of normalized name + address. **Dedupe across runs on this**, not on name |
| `domain` | Bare domain (`"franklinbbq.com"`) or `null` when the business only has a social page |
| `operating_status` | `'open'` or `'closed'` |
| `phone`, `address`, `category`, `name`, `website` | Contact and identity |
| `is_chain` | `true` when detected; `null` when not detected. **Never `false`** — absence of detection is not evidence of independence |
| `socials`, `rating`, `review_count`, `latitude`, `longitude` | Context |

Two flags decide whether you got the whole area: `truncated` is `true` when the vendor-call
cap stopped the search before `limit`. Treat a truncated result as a sample, not a census —
narrow the location or category and search again rather than assuming the market is that
small.

## Resolve one business — `agent.localResolve(options)`

Name plus at least one locating field. Use it to attach a domain and phone to a row you got
from somewhere else — a customer list, a spreadsheet, a scraped directory.

```typescript
const res = await agent.localResolve({
  name: 'Franklin Barbecue',
  address: '900 E 11th St',   // or city / postal_code / phone — phone is the strongest signal
  region: 'TX',
});
// res.{ found, confidence, result: LocalResult | null, closest_match?, candidates_considered }
```

`found: false` is a **completed result, not an error** — same contract as `findEmail`. Read
`confidence` before trusting a match, and `closest_match` to see what nearly cleared the
threshold. Do not retry a miss with the same inputs; add a locating field instead.

## The pattern that pays for itself

Local businesses rarely publish an email, so discovery and contact are two steps:

```typescript
// 1. Find them (one flat-price call for the whole area)
const { results } = await agent.localSearch({
  category: ['dental practice'], location: ['Austin, TX'], has_domain: true, limit: 200,
});

// 2. Turn each domain into a contactable address (oneshot-enrichment skill)
for (const biz of results) {
  if (!biz.domain) continue;
  const { email } = await agent.findEmail({ full_name: 'Office Manager', company_domain: biz.domain });
  if (email && (await agent.verifyEmail({ email })).deliverable) {
    // 3. Send (oneshot-email skill) — verify first, always
  }
}
```

Step 2 is priced per row while step 1 is not, so filter hard in step 1 —
`has_domain: true`, `min_rating`, `is_chain: false` — before you start paying per business.
Every row you drop before `findEmail` is money you do not spend.

## Pricing

`localSearch` is a flat price per search regardless of row count; `localResolve` is priced
per call. Current rates: https://docs.oneshotagent.com/pricing. `maxCost` still applies, and
because search is flat-priced it is the rare paid tool where asking for more rows costs
nothing extra.

## Related skills

- `oneshot-enrichment` — turn a domain into a person and an address
- `oneshot-gov` — government buyers, which need a different source entirely
- `oneshot-email` — send to what you found, after verifying it
