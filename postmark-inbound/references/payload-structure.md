# Inbound Webhook Payload Structure

When an email arrives, Postmark POSTs the following JSON to your webhook URL:

```json
{
  "FromName": "John Doe",
  "MessageStream": "inbound",
  "From": "john@example.com",
  "FromFull": {
    "Email": "john@example.com",
    "Name": "John Doe",
    "MailboxHash": ""
  },
  "To": "support+ticket-456@yourdomain.com",
  "ToFull": [
    {
      "Email": "support+ticket-456@yourdomain.com",
      "Name": "",
      "MailboxHash": "ticket-456"
    }
  ],
  "Cc": "",
  "CcFull": [],
  "Bcc": "",
  "BccFull": [],
  "OriginalRecipient": "support+ticket-456@yourdomain.com",
  "Subject": "Re: Issue with my order",
  "MessageID": "73e6d360-66eb-11e1-8e72-a8206ea7d3ea",
  "ReplyTo": "",
  "MailboxHash": "ticket-456",
  "Date": "Thu, 5 Apr 2025 16:59:01 +0200",
  "TextBody": "I still haven't received my order.",
  "HtmlBody": "<p>I still haven't received my order.</p>",
  "StrippedTextReply": "I still haven't received my order.",
  "Tag": "",
  "Headers": [
    {
      "Name": "Received",
      "Value": "by mx.postmarkapp.com ..."
    },
    {
      "Name": "Message-ID",
      "Value": "<CAExample123@mail.example.com>"
    }
  ],
  "Attachments": [
    {
      "Name": "screenshot.png",
      "Content": "base64-encoded-content",
      "ContentType": "image/png",
      "ContentLength": 45892,
      "ContentID": ""
    }
  ]
}
```

## Field Reference

| Field | Type | Description |
|-------|------|-------------|
| `From` | string | Sender email address |
| `FromName` | string | Sender display name |
| `FromFull` | object | Full sender details including MailboxHash |
| `To` | string | Recipient address (your inbound address) |
| `ToFull` | array | Full recipient details with MailboxHash per recipient |
| `Cc` / `CcFull` | string / array | CC recipients |
| `Bcc` / `BccFull` | string / array | BCC recipients (usually empty for inbound) |
| `OriginalRecipient` | string | The address the email was originally sent to |
| `Subject` | string | Email subject line |
| `MessageID` | string | Unique Postmark message identifier |
| `ReplyTo` | string | Reply-To header, if set by the sender |
| `MailboxHash` | string | The `+` hash portion of the address — primary routing mechanism |
| `Date` | string | When the email was sent (from the email's Date header) |
| `TextBody` | string | Full plain text body |
| `HtmlBody` | string | Full HTML body |
| `StrippedTextReply` | string | Just the reply text, with quoted/forwarded content stripped |
| `Tag` | string | Tag, if any (usually empty for inbound) |
| `Headers` | array | All email headers as `[{Name, Value}]` |
| `Attachments` | array | File attachments (see below) |

## Attachment Fields

Each object in `Attachments`:

| Field | Type | Description |
|-------|------|-------------|
| `Name` | string | File name |
| `Content` | string | Base64-encoded file content |
| `ContentType` | string | MIME type (e.g., `image/png`, `application/pdf`) |
| `ContentLength` | integer | File size in bytes |
| `ContentID` | string | If set, this is an inline image embedded in HTML — not a standalone file |

## StrippedTextReply vs TextBody

Use `StrippedTextReply` when you want just the new reply text without quoted content:

| Field | Contains |
|-------|---------|
| `TextBody` | Full email body including all quoted replies |
| `StrippedTextReply` | Only the new text the sender wrote in this reply |

For support ticket systems and reply-by-email, always prefer `StrippedTextReply` to avoid storing duplicate quoted content.

## MailboxHash Routing

`MailboxHash` contains the `+` suffix from the recipient address:

```
support+ticket-456@yourdomain.com  →  MailboxHash: "ticket-456"
help+order-789@yourdomain.com      →  MailboxHash: "order-789"
app+user-123@yourdomain.com        →  MailboxHash: "user-123"
```

The `MailboxHash` appears in:
- Top-level `MailboxHash` field
- `ToFull[].MailboxHash` for each recipient
- `FromFull.MailboxHash` (if the sender address had a `+` hash)

## Headers Array

The `Headers` array contains all email headers. Useful headers for threading:

| Header Name | Use |
|------------|-----|
| `Message-ID` | Sender's original message ID |
| `In-Reply-To` | The Message-ID this email is replying to |
| `References` | Thread reference chain |

Example — find `In-Reply-To` for email threading:

```javascript
const inReplyTo = inbound.Headers.find(h => h.Name === 'In-Reply-To')?.Value;
```
