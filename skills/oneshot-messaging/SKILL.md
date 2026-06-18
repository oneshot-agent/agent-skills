---
name: oneshot-messaging
description: |
  Send SMS and place autonomous AI phone calls with the OneShot SDK, paid per-message/per-minute
  in USDC via x402. Use when an agent needs to text one or many recipients (E.164), read its SMS
  inbox, or make an outbound voice call that pursues an objective (scheduling, follow-up, support)
  and returns a transcript + summary. Requires OneShot wallet setup — see the `oneshot` skill first.
metadata:
  author: oneshotagent
  version: "2.0.0"
  homepage: "https://oneshotagent.com"
---

# OneShot — SMS & Voice

Set up auth/wallet via the **`oneshot`** skill, then:

```typescript
import { OneShot } from '@oneshot-agent/sdk';
const agent = await OneShot.create({ cdp: true });
```

## SMS — `agent.sms(options)`

```typescript
const res = await agent.sms({
  message: 'Your appointment is confirmed for Tuesday 2pm.', // max 1600 chars
  to_number: '+15551234567',     // string | string[] (max 10 recipients), E.164
  maxCost: 0.20,
  memo: 'appointment confirmation',
});
// res.{ sent, failed, total, details: [{ to, status, message_sid?, error? }] }
```

### SMS inbox

```typescript
const inbox = await agent.smsInboxList({ limit: 50, from: '+15551234567', since: '2026-06-01T00:00:00Z' });
// inbox.messages: [{ id, from, to, body, media_urls?, thread_id?, received_at }]

const msg = await agent.smsInboxGet('message_id');
```

`smsInboxList`/`smsInboxGet` are **free**.

## Voice — `agent.voice(options)`

Places an outbound call where the OneShot agent pursues your objective, then returns the outcome.

```typescript
const call = await agent.voice({
  objective: 'Schedule a 30-minute demo for next Tuesday or Wednesday afternoon.',
  target_number: '+15551234567',           // string | string[]; array → conference-mode analysis
  caller_persona: 'A friendly scheduling assistant for Acme Corp.', // optional
  context: 'Following up on our email exchange about the demo.',     // optional
  max_duration_minutes: 10,                 // optional, 1–30
  maxCost: 4.00,
  memo: 'demo scheduling call',
});
// call.{ status, duration_seconds?, transcript?, summary?, success_evaluation?, structured_data? }
```

**Options** (`VoiceCallOptions`):

| Field | Notes |
|-------|-------|
| `objective` | **Required.** What the call should accomplish. |
| `target_number` | **Required.** `string \| string[]` in E.164. An array triggers conference-mode. |
| `caller_persona` | Optional persona for the caller. |
| `context` | Optional background for the call. |
| `max_duration_minutes` | Optional, 1–30. |

Voice and SMS are **async** — they poll to completion by default (`wait: true`); set `wait: false`
to receive a `request_id` and poll yourself. Both may include a one-time **phone registration fee**
the first time the agent provisions a number (surfaced in the quote / `needs_phone_registration`).

> Calling/texting an emergency number throws `EmergencyNumberError`.

## Pricing

| Action | Cost |
|--------|------|
| SMS | ~$0.035 per segment (+ one-time phone registration on first use) |
| Voice | ~$0.25/minute (+ one-time phone registration on first use) |
| SMS/voice inbox | Free |

Always pass `maxCost` on `voice` (duration-based) and on multi-recipient `sms`.
