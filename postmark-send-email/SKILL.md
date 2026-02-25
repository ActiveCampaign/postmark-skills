---
name: postmark-send-email
description: Use when sending transactional or broadcast emails through Postmark — single sends, batch (up to 500), bulk, or template-based emails with support for attachments, tracking, and message streams.
license: MIT
metadata:
  author: postmark
  version: "1.0.0"
---

# Send Email with Postmark

## Overview

Postmark provides multiple endpoints for sending emails:

| Approach | Endpoint | Use Case | Limits |
|----------|----------|----------|--------|
| **Single** | `POST /email` | Individual transactional emails | 1 email, 10 MB payload including attachments |
| **Batch** | `POST /email/batch` | Up to 500 emails in one request | 500 emails, 50 MB payload including attachments |
| **Template** | `POST /email/withTemplate` | Dynamic content with server-side templates | 1 email, 10 MB payload including attachments |
| **Batch Template** | `POST /email/batchWithTemplates` | Bulk templated emails | 500 emails, 50 MB payload including attachments |
| **Bulk** | `POST /email/bulk` | Broadcast stream campaigns | No fixed recipient cap, 50 MB payload including attachments |

**Choose batch when:** sending 2+ distinct emails at once, performance matters, or attachments are needed.
**Choose single when:** sending one email or real-time per-message error handling is needed.

## Message Streams (CRITICAL)

Postmark separates emails by intent. **Always specify MessageStream**:

| Stream | Value | Purpose | SMTP Endpoint |
|--------|-------|---------|---------------|
| **Transactional** | `outbound` | 1:1 triggered emails (default) | smtp.postmarkapp.com |
| **Broadcast** | `broadcast` | Marketing, newsletters | smtp-broadcasts.postmarkapp.com |

Never mix transactional and broadcast in the same stream — it damages deliverability. Servers can have up to 10 message streams.

## Quick Start

1. **Get API Token** from your [Postmark server settings](https://account.postmarkapp.com/servers)
2. **Verify sender** domain or email address
3. **Install SDK** — see [references/installation.md](references/installation.md)
4. **Choose endpoint** based on the decision matrix above

## Authentication

All API requests require the Server API Token:

```
X-Postmark-Server-Token: your-server-token-here
```

Store the token in `POSTMARK_SERVER_TOKEN`. For testing without sending, use `POSTMARK_API_TEST` as the token value.

## Single Email

**Endpoint:** `POST https://api.postmarkapp.com/email`

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `From` | string | Sender address (must be a verified domain or sender signature) |
| `To` | string | Recipients (comma-separated, max 50 total with Cc/Bcc) |
| `Subject` | string | Email subject line |
| `TextBody` or `HtmlBody` | string | Message content (at least one required) |

### Optional Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `Cc` | string | CC recipients |
| `Bcc` | string | BCC recipients |
| `ReplyTo` | string | Reply-to address |
| `MessageStream` | string | `outbound` (default) or `broadcast` |
| `Tag` | string | Category for statistics (one per message, max 1000 chars) |
| `Metadata` | object | Key-value pairs for custom tracking data (returned in webhook payloads) |
| `TrackOpens` | boolean | Enable open tracking |
| `TrackLinks` | string | `None`, `HtmlAndText`, `HtmlOnly`, `TextOnly` |
| `Headers` | array | Custom email headers `[{Name, Value}]` |
| `Attachments` | array | File attachments (max 10 MB total) |

```javascript
const postmark = require('postmark');
const client = new postmark.ServerClient(process.env.POSTMARK_SERVER_TOKEN);

const result = await client.sendEmail({
  From: 'notifications@yourdomain.com',
  To: 'customer@example.com',
  Subject: 'Your order has shipped',
  TextBody: 'Your order #12345 is on its way!',
  HtmlBody: '<p>Your order <strong>#12345</strong> is on its way!</p>',
  MessageStream: 'outbound',
  Tag: 'order-shipped'
});

console.log('MessageID:', result.MessageID);
```

Response includes `MessageID` — use it for tracking via webhooks or the Messages API.

See [references/single-email-examples.md](references/single-email-examples.md) for Python, Ruby, PHP, .NET, and cURL examples.

## Batch Email

**Endpoint:** `POST https://api.postmarkapp.com/email/batch`

Send up to **500 emails** in a single API call. Each message is independently validated and can have its own attachments, tags, and metadata.

```javascript
const results = await client.sendEmailBatch([
  {
    From: 'sender@yourdomain.com',
    To: 'user1@example.com',
    Subject: 'Order Shipped',
    TextBody: 'Your order has shipped!',
    MessageStream: 'outbound'
  },
  {
    From: 'sender@yourdomain.com',
    To: 'user2@example.com',
    Subject: 'Order Confirmed',
    TextBody: 'Your order is confirmed!',
    MessageStream: 'outbound'
  }
]);

// Check individual results — always handle partial failures
results.forEach((result, index) => {
  if (result.ErrorCode === 0) {
    console.log(`Email ${index + 1} sent: ${result.MessageID}`);
  } else {
    console.error(`Email ${index + 1} failed: ${result.Message}`);
  }
});
```

For more than 500 emails, chunk the array into groups of 500 and send sequentially. See [references/batch-email-examples.md](references/batch-email-examples.md) for chunking patterns, attachments, and Python/Ruby/cURL examples.

## Send with Template

**Endpoint:** `POST https://api.postmarkapp.com/email/withTemplate`

Use server-side Handlebars templates — no client-side rendering needed. Always use `TemplateAlias` over `TemplateId` — aliases survive re-creation and work across environments.

```javascript
const result = await client.sendEmailWithTemplate({
  From: 'sender@yourdomain.com',
  To: 'customer@example.com',
  TemplateAlias: 'order-confirmation',
  TemplateModel: {
    customer_name: 'Jane Doe',
    order_number: 'ORD-67890',
    items: [
      { name: 'Widget', price: '$19.99' },
      { name: 'Gadget', price: '$29.99' }
    ]
  },
  MessageStream: 'outbound'
});
```

For batch template sends (up to 500), use `POST /email/batchWithTemplates` via `client.sendEmailBatchWithTemplates([...])`. See [references/template-examples.md](references/template-examples.md) for full examples.

## Attachments

Include attachments as Base64-encoded content:

```json
{
  "Attachments": [
    {
      "Name": "invoice.pdf",
      "Content": "base64-encoded-content-here",
      "ContentType": "application/pdf"
    }
  ]
}
```

Embed inline images using `ContentID`:

```json
{
  "HtmlBody": "<img src=\"cid:logo123\">",
  "Attachments": [
    {
      "Name": "logo.png",
      "Content": "base64-encoded-image",
      "ContentType": "image/png",
      "ContentID": "cid:logo123"
    }
  ]
}
```

**Size limits:** TextBody/HtmlBody 5 MB each · Single message with attachments 10 MB · Batch payload 50 MB · Base64 encoding adds ~33% · Certain file types blocked (.exe, .bat)

## Tracking

Configure per-email or at server level:

```json
{ "TrackOpens": true, "TrackLinks": "HtmlAndText" }
```

**TrackLinks options:** `None` | `HtmlAndText` | `HtmlOnly` | `TextOnly`

Disable tracking for sensitive transactional emails (password resets, security alerts) to maximize deliverability.

## Testing

| Method | Address/Token | Result |
|--------|---------------|--------|
| **API Test Token** | `POSTMARK_API_TEST` | Validates request without sending |
| **Black Hole** | `test@blackhole.postmarkapp.com` | Dropped but appears in activity |
| **Sandbox Server** | Create sandbox server in dashboard | Full processing, no delivery |
| **Hard Bounce** | `hardbounce@bounce-testing.postmarkapp.com` | Simulates hard bounce |
| **Soft Bounce** | `softbounce@bounce-testing.postmarkapp.com` | Simulates soft bounce |

**Never** test with fake addresses at real providers (e.g., test@gmail.com) — damages sender reputation.

## Error Handling

| Code | Meaning | Action |
|------|---------|--------|
| 200 | Success | Continue |
| 401 | Unauthorized | Check API token — do not retry |
| 406 | Inactive recipient | Check suppression list — do not retry |
| 409 | JSON required | Fix `Accept`/`Content-Type` headers |
| 410 | Too many batch messages | Reduce to 500 or fewer per batch |
| 413 | Payload too large | Reduce payload (10 MB single, 50 MB batch) |
| 422 | Validation error | Fix request parameters — do not retry |
| 429 | Rate limited | Retry with exponential backoff |
| 500 | Server error | Retry with exponential backoff |

```javascript
async function sendWithRetry(client, email, maxRetries = 3) {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await client.sendEmail(email);
    } catch (error) {
      const isRetryable = error.statusCode === 429 || error.statusCode === 500;
      if (!isRetryable || attempt === maxRetries) throw error;
      await new Promise(r => setTimeout(r, Math.pow(2, attempt) * 1000));
    }
  }
}
```

See [references/error-handling.md](references/error-handling.md) for complete error patterns including batch partial failures and typed error classes.

## SMTP

Postmark supports SMTP for legacy integrations — host `smtp.postmarkapp.com` (transactional) or `smtp-broadcasts.postmarkapp.com` (broadcast), ports 25/2525/587, Server API Token as username and password. See [references/smtp-migration.md](references/smtp-migration.md) for migration examples and custom header reference.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Missing MessageStream | Always specify `outbound` or `broadcast` |
| Using broadcast for transactional | Use separate streams for different email types |
| Testing with real addresses | Use `POSTMARK_API_TEST` or sandbox mode |
| Retrying 422 errors | These are validation errors — fix request, don't retry |
| Not handling partial batch failures | Check each result in batch response array |
| Tracking on sensitive transactional emails | Disable for password resets, security alerts, receipts |
| Exceeding 50 recipients per email | Split into multiple emails or use batch |
| Not verifying sender | Domain or address must be verified before sending |

## Notes

- `From` address must use a verified domain or sender signature
- Store API key in `POSTMARK_SERVER_TOKEN` environment variable
- Maximum 50 recipients total per email (To + Cc + Bcc)
- `MessageID` returned in response is used for bounce/webhook/API correlation
- For broadcast campaigns to large lists, use the Bulk API endpoint (`POST /email/bulk`)
- Use `POSTMARK_API_TEST` as the token value in development and CI — validates without sending
- New domains require gradual volume warm-up — see [`postmark-email-best-practices`](../postmark-email-best-practices/) for the schedule
