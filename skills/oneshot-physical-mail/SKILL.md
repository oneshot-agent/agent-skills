---
name: oneshot-physical-mail
description: |
  Send real printed letters and postcards through the OneShot SDK — upload artwork, validate
  the recipient's postal address, preview the rendered mailpiece, approve a quote, and mail it,
  with a cancellation window and idempotent recovery. Use when the agent needs to reach someone
  by post rather than email or SMS. Requires OneShot wallet setup — see the `oneshot` skill first.
metadata:
  author: oneshotagent
  version: "1.0.0"
  homepage: "https://oneshotagent.com"
---

# OneShot — Physical Mail

Set up auth/wallet via the **`oneshot`** skill, then reach it through `agent.physicalMail`:

```typescript
import { OneShot } from '@oneshot-agent/sdk';
const agent = await OneShot.create({ cdp: true });
```

## Read this before the first send

Physical mail is the least reversible thing in the SDK. Once a mailpiece is submitted to the
printer it is paper in a truck: no recall, no edit, and it costs multiples of an email. The API
is deliberately shaped to slow you down — you cannot send without previewing, approving an
exact quote, and persisting an idempotency key first. Do not work around those steps.

**Delivery is not readership.** Every quote and order carries a literal
`delivery_proves_readership: false`. A delivered status means the postal service handled the
piece — nothing more. Never report a delivery as "they read it", and never build follow-up
logic that assumes it.

## The sequence

### 1. Upload artwork

```typescript
const asset = await agent.physicalMail.uploadArtwork(pdfBytes, 'application/pdf');
// { asset_id, content_hash, mime_type }   — also accepts image/png, image/jpeg
```

### 2. Validate the address before spending anything

```typescript
const check = await agent.physicalMail.validateAddress({
  name: 'Jane Doe',
  address_line1: '900 E 11th St',
  address_city: 'Austin',
  address_state: 'TX',
  address_zip: '78702',
});
// { deliverable, deliverability, address }  — `address` is the normalized form; send that one
```

Stop here if `deliverable` is false. An undeliverable address still costs a full mailpiece to
discover the hard way, and the returned normalized `address` is what you should carry forward.

### 3. Preview → a quote

```typescript
const quote = await agent.physicalMail.preview({
  to: check.address,
  from: senderAddress,
  artwork: { kind: 'letter', file: asset.asset_id, color: true, double_sided: false },
  // or: { kind: 'postcard', front: frontAssetId, back: backAssetId }
});
// { quote_id, input_hash, status: 'rendering' | 'ready', preview: { url, thumbnails },
//   total_usdc, service_fee_usdc, expires_at, approval_id }
```

`status` starts `rendering`. Poll `getQuote(quote_id)` until it is `ready` — `total_usdc` is
`null` until then, and approving a price you do not have yet is not possible.

**Show the preview URL to the developer.** It is a picture of exactly what gets printed, and it
is the last point at which a mistake is free.

### 4. Approve the exact quote

```typescript
const approval = await agent.physicalMail.approve({
  quote_id: quote.quote_id,
  input_hash: quote.input_hash,   // binds the approval to this artwork + these addresses
  total_usdc: quote.total_usdc,
  approved: true,                  // must be literally true; the SDK throws otherwise
});
// { quote_id, approval_id }
```

`input_hash` is what makes this an approval of *this* mailpiece rather than a blanket one. If
the artwork or either address changes, preview again — the old approval will not carry.

### 5. Send

```typescript
const key = crypto.randomUUID();   // persist this BEFORE the call
const order = await agent.physicalMail.send({
  quote_id: quote.quote_id,
  approval_id: approval.approval_id,
  idempotencyKey: key,
  maxCost: 3.00,
  memo: 'Q3 renewal notice',
});
// { order_id, order_status, payment_status, fulfillment_status,
//   receipt_id, signed_receipt, total_usdc, events[], cancel_before, ... }
```

The SDK refuses to send without both `approval_id` and `idempotencyKey`. Write the key to
durable storage *before* you call, not after — its whole purpose is to survive the crash that
loses your response.

### 6. Track, recover, cancel

```typescript
await agent.physicalMail.getOrder(order.order_id);   // order_status + events[] timeline
await agent.physicalMail.recover(key);               // lost the response? fetch by idempotency key
await agent.physicalMail.cancel(order.order_id);     // only before `cancel_before`
```

`order_status` moves through `pending` → `submitting` → `accepted`, or ends at `canceled`,
`failed`, or `needs_reconciliation`. That last one means the order and the printer disagree —
surface it to a human rather than retrying, and never re-send on it: that is how one letter
becomes two.

**If a send times out, call `recover(key)` — never re-send.** The order may well exist and be
paid for. `cancel` works only inside the `cancel_before` window and sets
`cancellation_requested`; check `cancellation_error` and `refunded_at` to see whether it
actually landed, rather than assuming the request succeeded.

## Pricing

Priced per mailpiece: printing, postage, and a service fee, quoted before you approve.
`total_usdc` on the quote is the number you approve and the number you are charged. Always pass
`maxCost` — the SDK rejects a non-positive value, and it is the client-side stop if a quote
comes back higher than you expect. Current rates: https://docs.oneshotagent.com/pricing.

## Not in the MCP server

`physicalMail` is SDK-only today. An MCP client has no `oneshot_physical_mail` tool — reach it
through `@oneshot-agent/sdk` directly.

## Related skills

- `oneshot-email` — reach the same person for a fraction of the cost, first
- `oneshot-enrichment` — turn a name into a mailing address
- `oneshot-local` — main-street businesses, which are often best reached by post
