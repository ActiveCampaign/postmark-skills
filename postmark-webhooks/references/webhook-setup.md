# Webhook Setup and Management

## Create a Webhook

**Endpoint:** `POST /webhooks`

### Node.js

```javascript
const postmark = require('postmark');
const client = new postmark.ServerClient(process.env.POSTMARK_SERVER_TOKEN);

const webhook = await client.createWebhook({
  Url: 'https://yourdomain.com/webhooks/postmark',
  MessageStream: 'outbound',
  Verify: true,
  HttpAuth: {
    Username: 'webhook-user',
    Password: process.env.WEBHOOK_SECRET
  },
  HttpHeaders: [
    { Name: 'X-Custom-Header', Value: 'my-value' }
  ],
  Triggers: {
    Delivery: { Enabled: true },
    Bounce: { Enabled: true, IncludeContent: false },
    SpamComplaint: { Enabled: true, IncludeContent: false },
    Open: { Enabled: true, PostFirstOpenOnly: true },
    Click: { Enabled: true },
    SubscriptionChange: { Enabled: true }
  }
});

console.log('Webhook ID:', webhook.ID);
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
      "Delivery": { "Enabled": true },
      "Bounce": { "Enabled": true, "IncludeContent": false },
      "SpamComplaint": { "Enabled": true, "IncludeContent": false },
      "Open": { "Enabled": true, "PostFirstOpenOnly": true },
      "Click": { "Enabled": true },
      "SubscriptionChange": { "Enabled": true }
    }
  }'
```

**Webhooks are per-stream.** Create separate webhooks for `outbound` and `broadcast` streams if you need events from both.

---

## List Webhooks

```javascript
const webhooks = await client.getWebhooks({ MessageStream: 'outbound' });

webhooks.Webhooks.forEach(w => {
  console.log(`${w.ID}: ${w.Url} (stream: ${w.MessageStream})`);
});
```

```bash
curl "https://api.postmarkapp.com/webhooks?MessageStream=outbound" \
  -H "Accept: application/json" \
  -H "X-Postmark-Server-Token: $POSTMARK_SERVER_TOKEN"
```

---

## Update a Webhook

Editing a webhook re-runs verification for every enabled event type. If any trigger fails, the edit is rejected with HTTP 422 and error code 1364, and **the existing webhook is left unchanged**.

```javascript
await client.editWebhook(webhookId, {
  Url: 'https://yourdomain.com/webhooks/postmark-v2',
  Verify: true, // default — re-test every enabled trigger before saving the edit
  Triggers: {
    Open: { Enabled: true, PostFirstOpenOnly: true },
    Click: { Enabled: true },
    Delivery: { Enabled: true },
    Bounce: { Enabled: true, IncludeContent: false },
    SpamComplaint: { Enabled: true },
    SubscriptionChange: { Enabled: true }
  }
});
```

---

## Delete a Webhook

```javascript
await client.deleteWebhook(webhookId);
```

---

## Endpoint Verification

When a webhook is created or edited, Postmark tests the endpoint for **every enabled event type**. Each test must return HTTP 200 before the webhook is saved as verified.

### The `Verify` field

| Field | Type | Default | Behavior |
|-------|------|---------|----------|
| `Verify` | boolean | `true` | `true` — test every enabled trigger before saving. `false` — save the webhook **without** testing; it stays unverified and receives no events until it passes verification later. |

Failed verification on create or edit is rejected with **HTTP 422** and **error code 1364**. Nothing is saved on create; on edit the existing webhook is left unchanged.

```javascript
try {
  await client.createWebhook({
    Url: 'https://yourdomain.com/webhooks/postmark',
    MessageStream: 'outbound',
    Verify: true,
    Triggers: { Delivery: { Enabled: true }, Bounce: { Enabled: true } }
  });
} catch (error) {
  if (error.statusCode === 422 && error.code === 1364) {
    // Verification failed — nothing was saved. Fix the endpoint and retry.
    console.error('Endpoint verification failed:', error.message);
  }
}
```

To save a webhook before its endpoint is live, set `Verify: false` and verify later:

```javascript
await client.createWebhook({
  Url: 'https://yourdomain.com/webhooks/postmark',
  MessageStream: 'outbound',
  Verify: false, // saved unverified — receives no events until it passes verification
  Triggers: { Delivery: { Enabled: true }, Bounce: { Enabled: true } }
});
```

---

## Verify on Demand

**Endpoint:** `POST /webhooks/{Id}/verify`

```bash
curl "https://api.postmarkapp.com/webhooks/12345/verify" \
  -X POST \
  -H "Accept: application/json" \
  -H "X-Postmark-Server-Token: $POSTMARK_SERVER_TOKEN"
```

**HTTP 200 means the check RAN — not that it passed.** `Success` can be `false` inside a 200 response. Always read the body.

```json
{
  "Id": 12345,
  "Url": "https://yourdomain.com/webhooks/postmark",
  "Success": false,
  "Results": [
    { "TriggerType": "Delivery", "Success": true, "StatusCode": 200, "Message": "OK" },
    { "TriggerType": "Open", "Success": true, "StatusCode": 200, "Message": "OK" },
    { "TriggerType": "Click", "Success": true, "StatusCode": 200, "Message": "OK" },
    { "TriggerType": "SubscriptionChange", "Success": true, "StatusCode": 200, "Message": "OK" },
    { "TriggerType": "Bounce", "Success": false, "StatusCode": 500, "Message": "Internal Server Error" }
  ],
  "Message": "4/5 triggers verified successfully"
}
```

| Field | Meaning |
|-------|---------|
| `Id` | Webhook ID that was checked |
| `Url` | Endpoint that was checked |
| `Success` | `true` only if **every** enabled trigger returned 200 |
| `Results[]` | Per-trigger outcome — `TriggerType`, `Success`, `StatusCode`, `Message` |
| `Message` | Human-readable summary, e.g. `"4/5 triggers verified successfully"` |

Check the body, not the status code:

```javascript
const result = await verifyWebhook(webhookId); // POST /webhooks/{Id}/verify

// Wrong: a 200 response does not mean the endpoint verified
// if (response.status === 200) console.log('verified');

if (!result.Success) {
  const failed = result.Results.filter(r => !r.Success);
  failed.forEach(r => console.error(`${r.TriggerType}: ${r.StatusCode} ${r.Message}`));
}
```

### Persistent Failure Pauses One Event Type

If an event type keeps failing across many events, Postmark marks **that event type** unverified and pauses delivery for it until the endpoint is fixed and re-verified.

Tracking is **per event type** — Bounce failing does not pause Delivery, Open, Click, SpamComplaint, or SubscriptionChange. Existing webhooks start as verified at launch cutover.

---

## Webhook Statistics

**Endpoint:** `GET /webhooks/{Id}/statistics`

One webhook per call, keyed by ID. There is no all-webhooks or per-stream variant and no status filter. The window is a fixed rolling **24 hours** — not filterable in v1.

```bash
curl "https://api.postmarkapp.com/webhooks/12345/statistics" \
  -H "Accept: application/json" \
  -H "X-Postmark-Server-Token: $POSTMARK_SERVER_TOKEN"
```

```json
{
  "WebhookId": 12345,
  "Url": "https://example.com/webhook",
  "Statuses": { "Bounce": "unverified", "Delivery": "verified" },
  "TimeRange": {
    "StartTime": "2025-04-04T16:33:54.9070259Z",
    "EndTime": "2025-04-05T16:33:54.9070259Z",
    "Hours": 24
  },
  "Metrics": {
    "TotalRequests": 1000,
    "SuccessCount": 950,
    "FailureCount": 50,
    "RetryCount": 62,
    "SuccessRate": 95.0,
    "SlowCount": 18,
    "VerySlowCount": 4,
    "AverageTerminalResponseTimeMs": 240,
    "AverageRetryResponseTimeMs": 1850
  },
  "MetricsByTrigger": {
    "Bounce": {
      "TotalRequests": 100,
      "SuccessCount": 50,
      "FailureCount": 50,
      "RetryCount": 62,
      "SuccessRate": 50.0,
      "SlowCount": 12,
      "VerySlowCount": 4,
      "AverageTerminalResponseTimeMs": 900,
      "AverageRetryResponseTimeMs": 1850
    },
    "Delivery": {
      "TotalRequests": 900,
      "SuccessCount": 900,
      "FailureCount": 0,
      "RetryCount": 0,
      "SuccessRate": 100.0,
      "SlowCount": 6,
      "VerySlowCount": 0,
      "AverageTerminalResponseTimeMs": 165,
      "AverageRetryResponseTimeMs": null
    }
  }
}
```

| Field | Meaning |
|-------|---------|
| `WebhookId` | The webhook these statistics describe |
| `Url` | Endpoint URL |
| `Statuses` | Verification status per event type (`verified` / `unverified`) |
| `TimeRange` | `StartTime`, `EndTime`, `Hours` — always a rolling 24 hours |
| `Metrics` | Top-level aggregate across all event types |
| `MetricsByTrigger` | The same metric fields, broken out per event type |

**Metric fields** — identical in `Metrics` and in each `MetricsByTrigger` entry:

| Field | Meaning |
|-------|---------|
| `TotalRequests` | Requests attempted in the window |
| `SuccessCount` | Requests that succeeded |
| `FailureCount` | Requests that failed |
| `RetryCount` | Retry attempts made |
| `SuccessRate` | Percentage of requests that succeeded |
| `SlowCount` | Slow responses |
| `VerySlowCount` | Very slow responses |
| `AverageTerminalResponseTimeMs` | Average response time of terminal attempts |
| `AverageRetryResponseTimeMs` | Average response time of retry attempts — `null` when the trigger had no retries |

Use `Statuses` to spot a paused event type, and per-trigger `SuccessRate` to find which endpoint path is failing:

```javascript
const stats = await getWebhookStatistics(webhookId); // GET /webhooks/{Id}/statistics

Object.entries(stats.Statuses).forEach(([trigger, status]) => {
  if (status === 'unverified') {
    console.warn(`${trigger} is paused — fix the endpoint and re-verify`);
  }
});

Object.entries(stats.MetricsByTrigger).forEach(([trigger, m]) => {
  console.log(`${trigger}: ${m.SuccessRate}% success, ${m.RetryCount} retries`);
  // AverageRetryResponseTimeMs is null when there were no retries
  if (m.AverageRetryResponseTimeMs !== null) {
    console.log(`  avg retry response: ${m.AverageRetryResponseTimeMs}ms`);
  }
});
```

---

## Retry Schedule (Outbound Webhooks)

This schedule applies to **outbound** webhooks — delivery, bounce, open, click, spam complaint, and subscription change. Inbound email webhooks run on separate infrastructure with a different, longer schedule; see the `postmark-inbound` skill. Do not apply one to the other.

### Retryable vs Permanent Failures

Postmark retries only **temporary** failures:

| Response | Classification | Behavior |
|----------|---------------|----------|
| `5xx` | Temporary | Retried |
| `408` Request Timeout | Temporary | Retried |
| `429` Too Many Requests | Temporary | Retried |
| Network timeout | Temporary | Retried |
| Every other `4xx` — including `400`, `401`, `403`, `404`, `405`, `410`, `422` | **Permanent** | Dropped on the first attempt — no retries |
| `3xx` | Temporary | Redirects are followed by default (up to 10 hops), so Postmark sees the **final** status and classifies that. A bare `3xx` is treated as temporary and retried — it is not a permanent failure. Don't rely on this: return `200` from the endpoint Postmark posts to. |

### Schedule

When a failure is retryable, a webhook gets **up to 9 retries over approximately 72 minutes**. The intervals are not a single flat list — delivery runs in tiers, and a webhook that starts failing works through two rungs in order:

**Rung 1 — standard (3 retries, 21 minutes)**

| Retry | Interval After Previous | Cumulative |
|-------|------------------------|------------|
| 1 | 1 minute | 1 min |
| 2 | 5 minutes | 6 min |
| 3 | 15 minutes | 21 min |

**Rung 2 — backoff (6 retries, 51 minutes)** — entered after rung 1 is exhausted:

| Retry | Interval After Previous | Cumulative |
|-------|------------------------|------------|
| 4 | 1 minute | 22 min |
| 5 | 5 minutes | 27 min |
| 6 | 10 minutes | 37 min |
| 7 | 10 minutes | 47 min |
| 8 | 10 minutes | 57 min |
| 9 | 15 minutes | 72 min |

After retry 9 the event is dropped permanently.

Note that the gap does **not** widen monotonically — it resets to 1 minute at the start of rung 2. If you are reconciling delivery gaps against these numbers, expect the 1m/5m pattern to appear twice.

The **same 9-retry ladder applies to every outbound event type** — delivery, bounce, open, click, spam complaint, and subscription change all retry identically, and the schedule cannot be customized per webhook or per event type.

### Reading `X-PM-Retries-Remaining`

Every delivery attempt carries an **`X-PM-Retries-Remaining`** header. It starts at **9** on the first failure and counts down to `0` on the final attempt, spanning both rungs — so a `9` early in an incident is expected, not a sign of misconfiguration.

Always return 200 and process asynchronously. Because a retry can redeliver an event your endpoint already processed, handlers must be idempotent — see [handler-examples.md](handler-examples.md).
