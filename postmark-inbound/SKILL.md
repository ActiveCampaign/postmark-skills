---
name: postmark-inbound
description: Use when processing incoming emails with Postmark inbound webhooks — building reply-by-email, email-to-ticket, document extraction, or any workflow that receives and parses email.
license: MIT
metadata:
  author: postmark
  version: "1.0.0"
---

# Process Inbound Email with Postmark

## Overview

Postmark's inbound processing parses incoming emails and delivers them as structured JSON to your webhook endpoint. This enables workflows like:

- **Reply-by-email** — Threading replies back to conversations
- **Email-to-ticket** — Converting emails into support tickets
- **Document extraction** — Processing email attachments automatically
- **Command processing** — Parsing structured data from emails
- **Forwarding/routing** — Routing emails to different services based on content

## How It Works

1. **Configure** an inbound address or domain in your Postmark server
2. **Set webhook URL** where Postmark will POST parsed email data
3. **Receive JSON** — Postmark processes the raw email and delivers structured data
4. **Respond with 200** — Your endpoint must return HTTP 200 to acknowledge receipt

```
Sender → Email → Postmark → Parses email → POST JSON → Your webhook endpoint
```

## Quick Start

1. **Set up inbound domain** — Configure MX records or email forwarding for your domain
2. **Set webhook URL** — In your Postmark server settings, set the Inbound webhook URL
3. **Build your endpoint** — Create an HTTP POST handler that accepts the inbound JSON payload
4. **Return 200** — Always respond with HTTP 200 to confirm receipt

### Error Handling and Retries

If your endpoint returns a **non-200 status code**, Postmark will automatically retry delivery up to **10 times** over approximately **10.5 hours** with escalating intervals:

| Retry | Interval After Previous Attempt |
|-------|-------------------------------|
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

**Important:** A **403 response** immediately stops all retries — Postmark interprets this as intentional rejection. After all retries are exhausted, the message is marked as "Failed" and appears as an "Inbound Error" in your activity page. You can manually retry failed messages via the API (`PUT /messages/inbound/{messageid}/retry`).

## Inbound Configuration

Two setup options — MX record (recommended) or email forwarding. Constraints: one inbound stream per server, one domain per stream, one webhook URL per stream.

See [references/inbound-setup.md](references/inbound-setup.md) for full DNS steps, forwarding caveats, retry schedule, and how to set your webhook URL.

## Webhook Payload

Key fields in the JSON Postmark POSTs to your endpoint:

| Field | Description |
|-------|-------------|
| `From` | Sender email address |
| `Subject` | Email subject line |
| `MailboxHash` | The `+` hash from the recipient address — primary routing mechanism |
| `TextBody` | Full plain text body (includes quoted replies) |
| `StrippedTextReply` | Reply text only — quoted content stripped |
| `HtmlBody` | Full HTML body |
| `Attachments` | Array of `{Name, Content, ContentType, ContentLength, ContentID}` |
| `MessageID` | Unique Postmark message identifier |
| `Headers` | All email headers as `[{Name, Value}]` |

See [references/payload-structure.md](references/payload-structure.md) for the full payload JSON, attachment fields, and header threading examples.

## MailboxHash for Routing

Use `+` addressing to route emails to specific records or conversations:

```
support+ticket-456@yourdomain.com  →  MailboxHash: "ticket-456"
notifications+order-789@yourdomain.com → MailboxHash: "order-789"
```

This is the primary mechanism for threading replies back to conversations or routing to specific records.

## Basic Endpoint

```javascript
const express = require('express');
const app = express();
app.use(express.json({ limit: '50mb' }));

app.post('/webhooks/inbound', (req, res) => {
  const { From, Subject, MailboxHash, StrippedTextReply, TextBody } = req.body;

  if (MailboxHash) {
    // Threaded reply — parse the hash to find the related record
    const [type, id] = MailboxHash.split('-');
    console.log(`Reply for ${type} #${id} from ${From}`);
  } else {
    console.log(`New inbound email from ${From}: ${Subject}`);
  }

  // Always prefer StrippedTextReply for replies
  const replyText = StrippedTextReply || TextBody;

  res.sendStatus(200); // Must return 200
});
```

See [references/handler-examples.md](references/handler-examples.md) for Node.js, Python, attachment processing, reply-by-email, and async processing patterns.

## Inbound Rules and Messages API

Block unwanted senders by address or domain, and query/retry processed messages via the API.

See [references/inbound-api.md](references/inbound-api.md) for inbound rules endpoints and the Messages API.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Not returning HTTP 200 | Always respond 200 — even if you process asynchronously |
| Returning 403 accidentally | This permanently stops retries for that message |
| Not parsing MailboxHash | Use `+` addressing for routing — it's the primary threading mechanism |
| Using `TextBody` instead of `StrippedTextReply` | `StrippedTextReply` removes quoted content from replies |
| No size limit on body parser | Set body parser limit to `50mb` for messages with attachments |
| Slow webhook processing | Process async (queue the work) and respond 200 immediately |
| Ignoring `ContentID` on attachments | Attachments with `ContentID` are inline images, not standalone files |

## Notes

- One Inbound Stream per server — use separate servers for different inbound domains
- Inbound webhook payloads can be large due to attachments — set appropriate body size limits
- `StrippedTextReply` strips quoted content, giving you just the new reply text
- The `MailboxHash` field is the portion after `+` in the recipient address — use it for routing
- Headers array contains all original email headers for advanced processing
- Same server can handle both inbound and outbound email
- Inbound processing is separate from outbound — different streams, different configuration
