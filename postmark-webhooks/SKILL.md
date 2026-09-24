---
name: postmark-webhooks
description: Use when setting up Postmark outbound webhooks for tracking email delivery, bounces, opens, clicks, spam complaints, or subscription changes — includes webhook configuration, endpoint verification, per-webhook statistics, retry behavior, payload handling, and security.
license: MIT
metadata:
  author: postmark
  version: "1.1.0"
---

# Postmark Webhooks

## Overview

Postmark webhooks deliver real-time event data to your endpoint via HTTP POST. Use webhooks to track what happens after you send an email.

This skill covers **outbound** webhooks — the six event types below. Inbound email webhooks run on separate infrastructure with different retry behavior; see `postmark-inbound` for those.

| Event | Trigger | Common Use |
|-------|---------|------------|
| **Delivery** | Email accepted by recipient server | Confirm delivery, update status |
| **Bounce** | Email rejected by recipient server | Clean lists, alert support |
| **SpamComplaint** | Recipient marked as spam | Remove from lists, investigate |
| **Open** | Recipient opened email (tracking pixel) | Engagement analytics |
| **Click** | Recipient clicked a tracked link | Engagement analytics, conversion tracking |
| **SubscriptionChange** | Recipient unsubscribed | Update preferences, comply with regulations |

## Quick Start

1. **Create a webhook** via API or [Postmark dashboard](https://account.postmarkapp.com) (Server → Webhooks)
2. **Set your endpoint URL** — must accept HTTP POST and return 200
3. **Select event triggers** — choose which events to receive
4. **Pass verification** — Postmark tests your endpoint for every enabled event type on create and edit; each must return HTTP 200 before the webhook is saved as verified
5. **Handle payloads** — parse the JSON body for each event type
6. **Respond with 200** — acknowledge receipt immediately, then process

## Webhook API

### Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/webhooks` | `GET` | List all webhooks for a message stream |
| `/webhooks/{webhookid}` | `GET` | Get a specific webhook |
| `/webhooks` | `POST` | Create a webhook |
| `/webhooks/{webhookid}` | `PUT` | Update a webhook |
| `/webhooks/{webhookid}` | `DELETE` | Delete a webhook |
| `/webhooks/{Id}/verify` | `POST` | Run endpoint verification on demand |
| `/webhooks/{Id}/statistics` | `GET` | Delivery statistics for one webhook (rolling 24 hours) |

### Create a Webhook

```javascript
const postmark = require('postmark');
const client = new postmark.ServerClient(process.env.POSTMARK_SERVER_TOKEN);

const webhook = await client.createWebhook({
  Url: 'https://yourdomain.com/webhooks/postmark',
  MessageStream: 'outbound',
  Verify: true, // default — test every enabled trigger before saving
  HttpAuth: {
    Username: 'webhook-user',
    Password: 'webhook-secret'
  },
  HttpHeaders: [
    { Name: 'X-Custom-Header', Value: 'my-value' }
  ],
  Triggers: {
    Open: { Enabled: true, PostFirstOpenOnly: false },
    Click: { Enabled: true },
    Delivery: { Enabled: true },
    Bounce: { Enabled: true, IncludeContent: true },
    SpamComplaint: { Enabled: true, IncludeContent: true },
    SubscriptionChange: { Enabled: true }
  }
});

console.log('Webhook created:', webhook.ID);
```

### cURL

```bash
curl "https://api.postmarkapp.com/webhooks" \
  -X POST \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -H "X-Postmark-Server-Token: $POSTMARK_SERVER_TOKEN" \
  -d '{
    "Url": "https://yourdomain.com/webhooks/postmark",
    "MessageStream": "outbound",
    "Verify": true,
    "Triggers": {
      "Open": { "Enabled": true, "PostFirstOpenOnly": false },
      "Click": { "Enabled": true },
      "Delivery": { "Enabled": true },
      "Bounce": { "Enabled": true, "IncludeContent": true },
      "SpamComplaint": { "Enabled": true, "IncludeContent": true },
      "SubscriptionChange": { "Enabled": true }
    }
  }'
```

### Trigger Options

| Trigger | Options |
|---------|---------|
| **Open** | `Enabled`, `PostFirstOpenOnly` (true = only first open per recipient) |
| **Click** | `Enabled` |
| **Delivery** | `Enabled` |
| **Bounce** | `Enabled`, `IncludeContent` (include original email content) |
| **SpamComplaint** | `Enabled`, `IncludeContent` |
| **SubscriptionChange** | `Enabled` |

## Endpoint Verification

When a webhook is created or edited, Postmark tests the endpoint for **every enabled event type**. Each test must return HTTP 200 before the webhook is saved as verified.

### The `Verify` field

| Field | Type | Default | Behavior |
|-------|------|---------|----------|
| `Verify` | boolean | `true` | `true` — test every enabled trigger before saving. `false` — save the webhook **without** testing; it stays unverified and receives no events until it passes verification later. |

If verification fails on create or edit, the save is rejected with **HTTP 422** and **error code 1364**. Nothing is saved on create; on edit the existing webhook is left unchanged.

### Verify on demand

`POST /webhooks/{Id}/verify` runs the check on demand.

**A 200 response means the check RAN — not that it passed.** Read the `Success` field to determine the result: `Success` can be `false` inside an HTTP 200 response. Never treat the status code as "verified."

The response body reports per-trigger results — `Id`, `Url`, `Success`, `Results[]` (each with `TriggerType`, `Success`, `StatusCode`, `Message`), and a summary `Message` such as `"4/5 triggers verified successfully"`. Top-level `Success` is `true` only if **every** enabled trigger returned 200.

See [references/webhook-setup.md](references/webhook-setup.md) for the full response shape and field reference.

### Persistent failure pauses one event type

If an event type keeps failing across many events, Postmark marks **that event type** unverified and pauses delivery for it until the endpoint is fixed and re-verified.

This is tracked **per event type** — Bounce failing does not pause Delivery or Open. Existing webhooks start as verified at launch cutover.

## Retries and Failure Handling

Retries apply to **outbound** webhooks. Inbound email webhooks use a different, longer schedule — see `postmark-inbound`. Do not apply one schedule to the other.

### Which failures are retried

| Response | Classification | Behavior |
|----------|---------------|----------|
| `5xx` | Temporary | Retried |
| `408` Request Timeout | Temporary | Retried |
| `429` Too Many Requests | Temporary | Retried |
| Network timeout | Temporary | Retried |
| Every other `4xx` (400, 401, 403, 404, 405, 410, 422) | **Permanent** | Dropped on the first attempt — no retries |
| `3xx` | Temporary | Redirects are followed by default, so Postmark classifies the **final** status. A bare `3xx` is retried, not dropped |

### Retry schedule

When a failure is retryable, a webhook gets **up to 9 retries over approximately 72 minutes**. Delivery runs in tiers, so a failing webhook works through two rungs in order:

| Rung | Retries | Intervals | Elapsed |
|------|---------|-----------|---------|
| Standard | 3 | 1 min, 5 min, 15 min | 21 min |
| Backoff (entered after standard is exhausted) | 6 | 1 min, 5 min, 10 min, 10 min, 10 min, 15 min | 51 min |
| **Total** | **9** | | **~72 min** |

After the 9th retry the event is dropped permanently. The gap does not widen monotonically — it resets to 1 minute when the backoff rung starts.

The same 9-retry ladder applies to **every outbound event type** and cannot be customized per webhook or per event type.

Each attempt carries an **`X-PM-Retries-Remaining`** header. It starts at **9** on the first failure and counts down across both rungs — a `9` early in an incident is expected, not a misconfiguration.

## Webhook Statistics

`GET /webhooks/{Id}/statistics` returns delivery statistics for **one** webhook, keyed by ID. There is no all-webhooks or per-stream variant and no status filter. The window is a fixed rolling **24 hours** — not filterable in v1.

The response contains:

| Field | Contents |
|-------|----------|
| `WebhookId`, `Url` | Which webhook the statistics describe |
| `Statuses` | Verification status per event type (`verified` / `unverified`) |
| `TimeRange` | `StartTime`, `EndTime`, `Hours` — always a rolling 24 hours |
| `Metrics` | Top-level aggregate: `TotalRequests`, `SuccessCount`, `FailureCount`, `RetryCount`, `SuccessRate`, `SlowCount`, `VerySlowCount`, `AverageTerminalResponseTimeMs`, `AverageRetryResponseTimeMs` |
| `MetricsByTrigger` | The same metric fields, broken out per event type |

- `AverageRetryResponseTimeMs` is `null` when a trigger had no retries.
- The same metric fields appear in both `Metrics` and each `MetricsByTrigger` entry.

Use `Statuses` to spot a paused event type and per-trigger `SuccessRate` to find which one is failing.

See [references/webhook-setup.md](references/webhook-setup.md) for the full response shape and field reference.

## Webhook Payloads

All payloads include `RecordType`, `MessageID`, `MessageStream`, and `Metadata` (from the original send). Use `RecordType` to route events:

```javascript
app.post('/webhooks/postmark', (req, res) => {
  res.sendStatus(200); // respond immediately

  const event = req.body;
  switch (event.RecordType) {
    case 'Delivery':          handleDelivery(event); break;
    case 'Bounce':            handleBounce(event); break;
    case 'SpamComplaint':     handleSpamComplaint(event); break;
    case 'Open':              handleOpen(event); break;
    case 'Click':             handleClick(event); break;
    case 'SubscriptionChange': handleSubscriptionChange(event); break;
  }
});
```

### Bounce Types

| Type | Code | Action |
|------|------|--------|
| `HardBounce` | 1 | Permanent — remove address from all lists |
| `SoftBounce` | 4096 | Temporary — Postmark retries; monitor |
| `Transient` | 2 | Temporary — retry may succeed |
| `SpamNotification` | 512 | Marked as spam at recipient's server |
| `Blocked` | 16 | Blocked by recipient server |
| `DMARCPolicy` | 100000 | Rejected due to DMARC policy |

See [references/payload-examples.md](references/payload-examples.md) for full JSON payloads for all 6 event types.

See [references/handler-examples.md](references/handler-examples.md) for complete Node.js and Python implementations, async processing, deduplication, and metadata correlation.

## Idempotency

A retry can redeliver an event your endpoint already processed — for example if it processed the event but responded too slowly. **Handlers must be idempotent.**

Deduplicate on the retry-stable **`X-PM-Webhook-Trace-Id`** header, falling back to `MessageID` when the header is absent. Respond 200 first, then process.

See [references/handler-examples.md](references/handler-examples.md) for Node.js and Python deduplication implementations.

## Security

Postmark does **not** sign webhook requests — there is no HMAC signature or signing header to verify. Do not write or trust signature-verification logic in a webhook handler.

Protect your endpoint with HTTP Basic Auth (credentials in the webhook URL, or the `HttpAuth` field), IP allowlisting, and payload-shape validation.

See [references/security.md](references/security.md) for full implementation examples.

## Bounce Management

Use the Bounces API and Suppression Management API alongside webhooks for comprehensive bounce handling.

See [references/bounce-management.md](references/bounce-management.md) for the Bounces API, suppression management, and bounce rate thresholds.

## Webhook Management

See [references/webhook-setup.md](references/webhook-setup.md) for list, update, delete, verification, statistics, and retry schedule details.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Not responding 200 | Always return HTTP 200 — even if processing fails. Process asynchronously. |
| Slow webhook handling | Respond 200 immediately, then process in background (queue, worker) |
| Treating a 200 from `/verify` as "verified" | A 200 means the check ran. Read the `Success` field — it can be `false` in a 200 response |
| Verifying an HMAC signature | Postmark does not sign webhooks — there is no signature header. Use Basic Auth, IP allowlisting, and payload-shape validation |
| No authentication | Use HTTP Basic Auth or custom headers to verify webhook source |
| Ignoring bounce types | Handle `HardBounce` differently from `SoftBounce` — hard bounces require permanent suppression |
| Not handling partial data | Some fields may be missing — always check for presence before accessing |
| Non-idempotent handlers | Retries can redeliver an already-processed event — dedupe on `X-PM-Webhook-Trace-Id`, falling back to `MessageID` |
| Expecting a 4xx to be retried | Only 5xx, 408, 429, and network timeouts are retried. Every other 4xx is permanent and dropped on the first attempt |
| Assuming `X-PM-Retries-Remaining: 9` is a bug | 9 is the correct starting value — the ladder is 3 standard retries plus 6 backoff retries |
| Missing MessageStream filter | Specify `MessageStream` when creating webhooks to avoid cross-stream events |
| Not tracking metadata | Include `Metadata` when sending to correlate webhook events with your records |

## Notes

- Webhooks are configured per message stream — create separate webhooks for `outbound` and `broadcast`
- Always respond HTTP 200 immediately — process webhook data asynchronously
- Postmark tests every enabled event type on create and edit; each must return 200 before the webhook is saved as verified. `"Verify": false` skips the test and leaves the webhook unverified, receiving no events
- A failed verification on create or edit is rejected with HTTP 422 and error code 1364
- Outbound retries: up to **9 retries** over ~72 minutes, in two rungs — standard (1 min, 5 min, 15 min) then backoff (1 min, 5 min, 10 min, 10 min, 10 min, 15 min). Same ladder for every outbound event type; not customizable. Only 5xx, 408, 429, and network timeouts are retried; every other 4xx is permanent and dropped on the first attempt. `X-PM-Retries-Remaining` starts at 9 and counts down across both rungs. Inbound webhooks use a different schedule — see `postmark-inbound`
- Persistent failure on one event type marks that event type unverified and pauses delivery for it only — other event types keep flowing
- Handlers must be idempotent — dedupe on `X-PM-Webhook-Trace-Id`, falling back to `MessageID`
- Use `MessageID` to correlate webhook events with sent emails
- `Metadata` from the original send is included in all webhook payloads
- Open tracking requires a tracking pixel in HTML — it does not work with plain text emails
- Click tracking requires `TrackLinks` to be enabled on the sent email
- Bounce webhooks fire for bounces and blocks — check the `Type` field to distinguish
- Spam complaints, unsubscribes, and manual deactivations have their own event types (not Bounce)
- Individual open/click data is stored for 45 days; aggregated statistics are stored indefinitely
