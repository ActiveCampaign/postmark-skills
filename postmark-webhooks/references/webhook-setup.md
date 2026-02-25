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

```javascript
await client.editWebhook(webhookId, {
  Url: 'https://yourdomain.com/webhooks/postmark-v2',
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

## Retry Schedule

Postmark retries webhook delivery when your endpoint does not return HTTP 200:

| Retry | Interval After Previous |
|-------|------------------------|
| 1 | 1 minute |
| 2 | 5 minutes |
| 3 | 10 minutes |
| 4 | 10 minutes |
| 5 | 10 minutes |
| 6 | 15 minutes |
| 7 | 30 minutes |
| 8 | 1 hour |
| 9 | 2 hours |
| 10 | 6 hours |

A **403 response** immediately stops all retries. Always return 200 and process asynchronously.
