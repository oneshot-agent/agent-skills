---
name: oneshot-gov
description: |
  Find federal contract opportunities on SAM.gov with the OneShot SDK — Sources Sought and
  Presolicitation notices by NAICS code, with the contracting officer's published name and
  email, agency, response deadline, set-aside, and description. Use when the buyer is a
  government agency rather than a company, or to reach a requirement while it is still being
  written. Requires OneShot wallet setup — see the `oneshot` skill first.
metadata:
  author: oneshotagent
  version: "1.0.0"
  homepage: "https://oneshotagent.com"
---

# OneShot — Federal Solicitations

Set up auth/wallet via the **`oneshot`** skill, then:

```typescript
import { OneShot } from '@oneshot-agent/sdk';
const agent = await OneShot.create({ cdp: true });
```

## Why this instead of people search

For a government buyer there is nothing to enrich. The contracting officer's name and email
are *published on the notice* — a `peopleSearch` + `findEmail` + `verifyEmail` chain would
pay three times to guess at a contact that arrives here for free with the opportunity
attached.

The default notice types matter as much as the contact. `'r'` (Sources Sought) and `'p'`
(Presolicitation) are the window where the requirement is still being written and the agency
is explicitly asking industry what is possible. By the time a solicitation is final, the
specification usually describes somebody else's product.

## Search — `agent.govSolicitations(options)`

```typescript
const found = await agent.govSolicitations({
  naics: ['541511', '541512'],        // required: 1–20 six-digit codes
  notice_types: ['r', 'p'],           // default — Sources Sought + Presolicitation
  since_days: 30,                     // look-back on posted date, 1–365, default 30
  state: 'TX',                        // place-of-performance state code
  agencies: ['DEPT OF THE ARMY'],     // case-insensitive substring match on agency path
  keywords: ['cloud migration'],
  set_aside: 'SBA',                   // SBA, 8A, HZC, SDVOSBC, WOSB, …
  active_only: true,                  // default: drop archived + past-deadline notices
  has_contact: true,                  // only notices with a named contact AND an email
  include_description: true,          // default; capped per search — see `truncated`
  limit: 100,                         // 1–500, default 100
});
// found.{ results: Solicitation[], total_found, truncated, data_as_of, description_fetches }
```

NAICS codes are validated client-side: exactly six digits, or the call throws a
`ValidationError` before it costs anything.

Each `Solicitation` carries `notice_id`, `title`, `agency`, `naics_code`, `posted_date`,
`response_deadline`, `set_aside`, `active`, `place_of_performance`, `description`, the
sam.gov `url`, and — the reason to call this — `contact` (the contracting officer) plus a
full `contacts` array.

## Two fields that change what you should conclude

**`data_as_of`** is the snapshot behind the rows. Results usually come from SAM.gov's daily
extract, so a notice posted this morning typically appears tomorrow. Never tell a user "there
are no opportunities" — tell them "no opportunities as of `data_as_of`".

**`truncated`** means some rows came back without a `description` because the per-search
fetch cap was hit. The notices are real; only their bodies are missing. Narrow with
`keywords` or `naics` rather than concluding the descriptions do not exist.

Zero notices is a **completed search**, not a failure. Re-running it unchanged will return
zero again and cost the same. Widen `since_days`, drop `state`, or add NAICS codes.

## From notice to conversation

```typescript
const { results, data_as_of } = await agent.govSolicitations({
  naics: ['541511'], state: 'TX', has_contact: true, since_days: 14,
});

for (const notice of results) {
  if (!notice.contact?.email) continue;
  // Deadlines are the whole game — a Sources Sought response is worthless late.
  if (notice.response_deadline && new Date(notice.response_deadline) < new Date()) continue;

  // Reference the notice_id and title; contracting officers field many of these.
  // Send via the oneshot-email skill, from a warmed domain.
}
```

Two rules specific to this audience: reference the `solicitation_number` or `notice_id` in
the subject line, and respect the `response_deadline` — a Sources Sought response that
arrives after the window closes is not read. And a published government contact is published
for that solicitation. Use it for that; do not add it to a general list.

## Pricing

Flat price per search regardless of row count, so ask for the full look-back window in one
call rather than paginating by day. Current rates:
https://docs.oneshotagent.com/pricing.

## Related skills

- `oneshot-email` — respond to a notice from a warmed sending domain
- `oneshot-research` — read the agency's prior awards before responding
- `oneshot-local` — the other non-corporate buyer type
