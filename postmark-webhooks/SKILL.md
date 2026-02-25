---
name: postmark-webhooks
description: Use when setting up Postmark webhooks for tracking email delivery, bounces, opens, clicks, spam complaints, or subscription changes — includes webhook configuration, payload handling, and security.
license: MIT
metadata:
  author: postmark
  version: "1.0.0"
---

# Postmark Webhooks

## Overview

Postmark webhooks deliver real-time event data to your endpoint via HTTP POST. Use webhooks to track what happens after you send an email.

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
4. **Handle payloads** — parse the JSON body for each event type
5. **Respond with 200** — acknowledge receipt immediately

## Webhook API

### Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/webhooks` | `GET` | List all webhooks for a message stream |
| `/webhooks/{webhookid}` | `GET` | Get a specific webhook |
| `/webhooks` | `POST` | Create a webhook |
| `/webhooks/{webhookid}` | `PUT` | Update a webhook |
| `/webhooks/{webhookid}` | `DELETE` | Delete a webhook |

### Create a Webhook

```javascript
const postmark = require('postmark');
const client = new postmark.ServerClient(process.env.POSTMARK_SERVER_TOKEN);

const webhook = await client.createWebhook({
  Url: 'https://yourdomain.com/webhooks/postmark',
  MessageStream: 'outbound',
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

## Security

Always verify that requests are genuinely from Postmark using HTTP Basic Auth, custom headers, or IP allowlisting.

See [references/security.md](references/security.md) for full implementation examples.

## Bounce Management

Use the Bounces API and Suppression Management API alongside webhooks for comprehensive bounce handling.

See [references/bounce-management.md](references/bounce-management.md) for the Bounces API, suppression management, and bounce rate thresholds.

## Webhook Management

See [references/webhook-setup.md](references/webhook-setup.md) for list, update, delete, and retry schedule details.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Not responding 200 | Always return HTTP 200 — even if processing fails. Process asynchronously. |
| Slow webhook handling | Respond 200 immediately, then process in background (queue, worker) |
| No authentication | Use HTTP Basic Auth or custom headers to verify webhook source |
| Ignoring bounce types | Handle `HardBounce` differently from `SoftBounce` — hard bounces require permanent suppression |
| Not handling partial data | Some fields may be missing — always check for presence before accessing |
| Duplicate handling | Webhooks may be delivered more than once — use `MessageID` for deduplication |
| Missing MessageStream filter | Specify `MessageStream` when creating webhooks to avoid cross-stream events |
| Not tracking metadata | Include `Metadata` when sending to correlate webhook events with your records |

## Notes

- Webhooks are configured per message stream — create separate webhooks for `outbound` and `broadcast`
- Always respond HTTP 200 immediately — process webhook data asynchronously
- Postmark retries failed webhook deliveries up to **10 times** over ~10.5 hours with escalating intervals: 1 min, 5 min, 10 min, 10 min, 10 min, 15 min, 30 min, 1 hr, 2 hrs, 6 hrs. A **403 response** immediately stops all retries. This retry schedule cannot be customized
- Use `MessageID` to correlate webhook events with sent emails
- `Metadata` from the original send is included in all webhook payloads
- Open tracking requires a tracking pixel in HTML — it does not work with plain text emails
- Click tracking requires `TrackLinks` to be enabled on the sent email
- Bounce webhooks fire for bounces and blocks — check the `Type` field to distinguish
- Spam complaints, unsubscribes, and manual deactivations have their own event types (not Bounce)
- Individual open/click data is stored for 45 days; aggregated statistics are stored indefinitely
