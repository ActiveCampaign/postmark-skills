# Inbound Email Handler Examples

## Node.js / Express

```javascript
const express = require('express');
const app = express();
app.use(express.json({ limit: '50mb' })); // attachments can be large

app.post('/webhooks/inbound', (req, res) => {
  const inbound = req.body;

  console.log('From:', inbound.From);
  console.log('Subject:', inbound.Subject);
  console.log('MailboxHash:', inbound.MailboxHash);
  console.log('Text:', inbound.StrippedTextReply || inbound.TextBody);
  console.log('Attachments:', inbound.Attachments?.length || 0);

  // Route based on MailboxHash
  if (inbound.MailboxHash) {
    handleThreadedReply(inbound.MailboxHash, inbound);
  } else {
    handleNewInbound(inbound);
  }

  // Always respond 200 — even if processing happens asynchronously
  res.sendStatus(200);
});

function handleThreadedReply(hash, inbound) {
  const [type, id] = hash.split('-');
  console.log(`Threaded reply for ${type} #${id}`);
  // Look up the related record and add this reply
}

function handleNewInbound(inbound) {
  console.log('New inbound email from:', inbound.From);
  // Create a new ticket, record, etc.
}
```

## Python / Flask

```python
from flask import Flask, request

app = Flask(__name__)

@app.route('/webhooks/inbound', methods=['POST'])
def handle_inbound():
    inbound = request.get_json()

    print(f"From: {inbound['From']}")
    print(f"Subject: {inbound['Subject']}")
    print(f"MailboxHash: {inbound.get('MailboxHash', '')}")
    print(f"Text: {inbound.get('StrippedTextReply') or inbound.get('TextBody')}")
    print(f"Attachments: {len(inbound.get('Attachments', []))}")

    mailbox_hash = inbound.get('MailboxHash', '')
    if mailbox_hash:
        handle_threaded_reply(mailbox_hash, inbound)
    else:
        handle_new_inbound(inbound)

    return '', 200

def handle_threaded_reply(hash_value, inbound):
    parts = hash_value.split('-', 1)
    if len(parts) == 2:
        record_type, record_id = parts
        print(f"Threaded reply for {record_type} #{record_id}")

def handle_new_inbound(inbound):
    print(f"New inbound email from: {inbound['From']}")
```

## Processing Attachments

```javascript
const fs = require('fs');
const path = require('path');

function processAttachments(inbound) {
  if (!inbound.Attachments || inbound.Attachments.length === 0) return [];

  return inbound.Attachments.map(attachment => {
    // Skip inline images (they are embedded in HtmlBody, not standalone files)
    if (attachment.ContentID) {
      return { name: attachment.Name, isInline: true };
    }

    // Decode Base64 content
    const buffer = Buffer.from(attachment.Content, 'base64');

    // Save to disk or upload to cloud storage
    const filePath = path.join('/tmp/attachments', attachment.Name);
    fs.writeFileSync(filePath, buffer);

    return {
      name: attachment.Name,
      contentType: attachment.ContentType,
      size: attachment.ContentLength,
      path: filePath,
      isInline: false
    };
  });
}
```

**Note:** Set your body parser limit to `50mb` — inbound payloads can be large when attachments are included.

## Reply-by-Email Pattern

A complete implementation for threading email replies back to records.

### Step 1: Send the Original Email with a Hashed Reply-To

```javascript
const postmark = require('postmark');
const client = new postmark.ServerClient(process.env.POSTMARK_SERVER_TOKEN);

async function sendTicketEmail(ticketId, customerEmail) {
  await client.sendEmail({
    From: 'support@yourdomain.com',
    To: customerEmail,
    // Hash encodes the ticket ID so replies route back to it
    ReplyTo: `support+ticket-${ticketId}@yourdomain.com`,
    Subject: `[Ticket #${ticketId}] We received your request`,
    TextBody: 'Our team is looking into your issue and will reply shortly.',
    MessageStream: 'outbound'
  });
}
```

### Step 2: Handle the Reply in Your Inbound Webhook

```javascript
app.post('/webhooks/inbound', async (req, res) => {
  const { MailboxHash, StrippedTextReply, TextBody, From, Attachments } = req.body;

  if (MailboxHash && MailboxHash.startsWith('ticket-')) {
    const ticketId = MailboxHash.replace('ticket-', '');

    await addReplyToTicket(ticketId, {
      from: From,
      body: StrippedTextReply || TextBody, // prefer StrippedTextReply
      attachments: Attachments || []
    });
  }

  res.sendStatus(200);
});
```

## Async Processing Pattern

For slow operations (database writes, file uploads), respond 200 immediately and process in the background:

```javascript
const Queue = require('bull'); // or any queue library
const inboundQueue = new Queue('inbound-email');

app.post('/webhooks/inbound', (req, res) => {
  // Queue the work — do not block the response
  inboundQueue.add(req.body);

  // Respond immediately to prevent Postmark from retrying
  res.sendStatus(200);
});

inboundQueue.process(async (job) => {
  const inbound = job.data;
  // Slow operations are fine here
  await processInboundEmail(inbound);
});
```
