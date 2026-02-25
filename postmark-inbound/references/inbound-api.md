# Inbound Rules and Messages API

## Inbound Rules

Block unwanted messages by email address or domain before they reach your webhook.

### API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/triggers/inboundrules` | `GET` | List all inbound rules |
| `/triggers/inboundrules` | `POST` | Create a new block rule |
| `/triggers/inboundrules/{ruleid}` | `DELETE` | Delete a rule |

### List Rules

```bash
curl "https://api.postmarkapp.com/triggers/inboundrules" \
  -H "Accept: application/json" \
  -H "X-Postmark-Server-Token: $POSTMARK_SERVER_TOKEN"
```

### Create a Block Rule

Block a specific address:

```bash
curl "https://api.postmarkapp.com/triggers/inboundrules" \
  -X POST \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -H "X-Postmark-Server-Token: $POSTMARK_SERVER_TOKEN" \
  -d '{"Rule": "spammer@example.com"}'
```

Block an entire domain:

```bash
curl "https://api.postmarkapp.com/triggers/inboundrules" \
  -X POST \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -H "X-Postmark-Server-Token: $POSTMARK_SERVER_TOKEN" \
  -d '{"Rule": "spamdomain.com"}'
```

Rules accept exact email addresses (`user@example.com`) or entire domains (`example.com`).

### Delete a Rule

```bash
curl "https://api.postmarkapp.com/triggers/inboundrules/{ruleid}" \
  -X DELETE \
  -H "Accept: application/json" \
  -H "X-Postmark-Server-Token: $POSTMARK_SERVER_TOKEN"
```

---

## Messages API (Inbound)

Query and manage processed inbound messages.

### API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/messages/inbound` | `GET` | Search inbound messages |
| `/messages/inbound/{messageid}/details` | `GET` | Get full details for one message |
| `/messages/inbound/{messageid}/bypass` | `PUT` | Bypass block rules for a specific message |
| `/messages/inbound/{messageid}/retry` | `PUT` | Retry a failed webhook delivery |

### Search Inbound Messages

```bash
curl "https://api.postmarkapp.com/messages/inbound?count=50&offset=0" \
  -H "Accept: application/json" \
  -H "X-Postmark-Server-Token: $POSTMARK_SERVER_TOKEN"
```

Query parameters:

| Parameter | Description |
|-----------|-------------|
| `count` | Number of results (max 500) |
| `offset` | Pagination offset |
| `recipient` | Filter by recipient address |
| `fromemail` | Filter by sender address |
| `subject` | Filter by subject |
| `mailboxhash` | Filter by MailboxHash value |
| `status` | `blocked`, `processed`, `queued`, `failed`, `scheduled` |
| `fromdate` | Start date (YYYY-MM-DD) |
| `todate` | End date (YYYY-MM-DD) |

### Retry a Failed Delivery

If your webhook failed to respond with 200 and all retries were exhausted:

```bash
curl "https://api.postmarkapp.com/messages/inbound/{messageid}/retry" \
  -X PUT \
  -H "Accept: application/json" \
  -H "X-Postmark-Server-Token: $POSTMARK_SERVER_TOKEN"
```

### Node.js Examples

```javascript
const postmark = require('postmark');
const client = new postmark.ServerClient(process.env.POSTMARK_SERVER_TOKEN);

// Search inbound messages
const messages = await client.getInboundMessages({
  count: 50,
  offset: 0,
  status: 'processed'
});

messages.InboundMessages.forEach(msg => {
  console.log(`${msg.From} → ${msg.Subject} (${msg.Status})`);
});

// Retry a failed delivery
await client.retryInboundMessage(messageId);
```
