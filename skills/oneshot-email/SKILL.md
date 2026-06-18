---
name: oneshot-email
description: |
  Send and receive real email autonomously with the OneShot SDK, paid per-send in USDC via x402.
  Use when an agent needs to send transactional or outreach email (with attachments, custom
  sending domain, reply threading), read its inbound inbox, or manage its pool of warmed sending
  domains. Requires OneShot wallet setup — see the `oneshot` skill first.
metadata:
  author: oneshotagent
  version: "2.0.0"
  homepage: "https://oneshotagent.com"
---

# OneShot — Email

Send and receive email from the agent's own sending domains. Set up auth/wallet via the
**`oneshot`** skill, then:

```typescript
import { OneShot } from '@oneshot-agent/sdk';
const agent = await OneShot.create({ cdp: true });
```

## Send email — `agent.email(options)`

```typescript
const result = await agent.email({
  to: 'recipient@example.com',          // string | string[]; optional only when replying
  subject: 'Hello from my agent',        // optional only when replying
  body: 'This email was sent autonomously.',
  maxCost: 0.05,
  memo: 'intro outreach',
});
// result.status, result.email?.{ id, provider_message_id, status }, result.domain, result.warning
```

**Options** (`EmailToolOptions`):

| Field | Notes |
|-------|-------|
| `to` | `string \| string[]`. Required unless `reply_to_email_id` is set. |
| `subject` | Required unless replying (then defaults to `Re: <original>`). |
| `body` | **Required.** Plain text/markdown body. |
| `reply_to_email_id` | Reply within a thread using an inbound email id from `inboxList()`/`inboxGet()`. Server resolves `In-Reply-To`/`References` and derives `to`/`subject` if omitted. |
| `from_domain` | Pin a specific sending domain (bypasses warmup rotation — see below). |
| `from_mailbox` | Local-part, defaults to `agent` (i.e. `agent@from_domain`). |
| `from_name` | Display name, e.g. `"Jane Doe"` → `Jane Doe <jane@domain>`. |
| `attachments` | `Array<{ filename?, content?, url?, content_type? }>` — `content` is base64, or pass a `url`. |

Plus shared options (`maxCost`, `wait`, `memo`, `idempotencyKey`, …) from the `oneshot` skill.
**`email` honors `idempotencyKey`** — reuse the same key on retry to avoid double-sends.

### Sending domains & warmup (important)

OneShot protects deliverability with a pool of warmed sending domains:

- **Rotate mode (recommended):** omit `from_domain`/`from_mailbox`. The server picks a warmed
  domain from the agent's pool. The chosen address comes back on the response.
- **Pin mode:** set `from_domain` to force a specific domain — this bypasses warmup protection.

If a pinned domain is still warming or over its daily limit, the send still proceeds but
`result.warning` is set (`pinned_domain_warming` / `pinned_over_limit`) — back off accordingly.
An un-pinned send with **no** eligible owned domain is a hard `400 no_sending_domain` (there is
no shared fallback sender; the agent must own a sending domain — first send provisions one,
~$10 one-time).

### Manage the domain pool

```typescript
const pool = await agent.listDomains();      // { agent_id, domains: [{ domain, pool_status, warmup_score, daily_send_limit, daily_sent_count, ... }] }
await agent.pauseDomain('mydomain.com');     // remove from rotation
await agent.resumeDomain('mydomain.com');    // restore to rotation
```

## Inbox

```typescript
const inbox = await agent.inboxList({ limit: 20, include_body: true, since: '2026-06-01T00:00:00Z' });
// inbox.emails: [{ id, from, subject, received_at, thread_id, body?, attachments? }]

const email = await agent.inboxGet('email_id'); // full InboxEmail
```

`inboxList` and `inboxGet` are **free**. To reply, pass the email's `id` as `reply_to_email_id`
to `agent.email(...)`.

## Pricing

| Action | Cost |
|--------|------|
| Send email | ~$0.01/email (+ ~$10 one-time first domain provisioning) |
| Inbox list/get | Free |

## Errors

`ContentBlockedError` (policy), `ValidationError` (bad address/body), `ToolError` (e.g.
`no_sending_domain`). See the `oneshot` skill for the full set.
