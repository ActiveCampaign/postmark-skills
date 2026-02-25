# Batch Email Examples

Complete examples for sending batch emails via `POST /email/batch` (up to 500 per call).

## Node.js / TypeScript

### Basic Batch

```javascript
const postmark = require('postmark');
const client = new postmark.ServerClient(process.env.POSTMARK_SERVER_TOKEN);

const results = await client.sendEmailBatch([
  {
    From: 'sender@yourdomain.com',
    To: 'user1@example.com',
    Subject: 'Welcome!',
    TextBody: 'Welcome to our service, User 1.',
    MessageStream: 'outbound'
  },
  {
    From: 'sender@yourdomain.com',
    To: 'user2@example.com',
    Subject: 'Welcome!',
    TextBody: 'Welcome to our service, User 2.',
    MessageStream: 'outbound'
  }
]);

results.forEach((result, index) => {
  if (result.ErrorCode === 0) {
    console.log(`Email ${index + 1}: sent (${result.MessageID})`);
  } else {
    console.error(`Email ${index + 1}: failed (${result.Message})`);
  }
});
```

### Batch with Attachments

```javascript
const fs = require('fs');

const invoiceContent = fs.readFileSync('./invoice-template.pdf').toString('base64');

const results = await client.sendEmailBatch([
  {
    From: 'billing@yourdomain.com',
    To: 'customer1@example.com',
    Subject: 'Invoice #001',
    TextBody: 'Please find your invoice attached.',
    MessageStream: 'outbound',
    Attachments: [
      {
        Name: 'invoice-001.pdf',
        Content: invoiceContent,
        ContentType: 'application/pdf'
      }
    ]
  },
  {
    From: 'billing@yourdomain.com',
    To: 'customer2@example.com',
    Subject: 'Invoice #002',
    TextBody: 'Please find your invoice attached.',
    MessageStream: 'outbound',
    Attachments: [
      {
        Name: 'invoice-002.pdf',
        Content: invoiceContent,
        ContentType: 'application/pdf'
      }
    ]
  }
]);
```

### Chunking Large Lists (500+ Emails)

```javascript
function chunkArray(array, size) {
  const chunks = [];
  for (let i = 0; i < array.length; i += size) {
    chunks.push(array.slice(i, i + size));
  }
  return chunks;
}

async function sendLargeBatch(client, emails) {
  const chunks = chunkArray(emails, 500);
  const allResults = [];

  for (const chunk of chunks) {
    const results = await client.sendEmailBatch(chunk);
    allResults.push(...results);

    // Log progress
    const sent = results.filter(r => r.ErrorCode === 0).length;
    const failed = results.filter(r => r.ErrorCode !== 0).length;
    console.log(`Chunk: ${sent} sent, ${failed} failed`);
  }

  return allResults;
}

// Usage
const emails = users.map(user => ({
  From: 'notifications@yourdomain.com',
  To: user.email,
  Subject: `Hello ${user.name}`,
  TextBody: `Hi ${user.name}, here is your weekly update.`,
  MessageStream: 'outbound',
  Tag: 'weekly-update',
  Metadata: { user_id: user.id }
}));

const results = await sendLargeBatch(client, emails);
```

### Handling Partial Failures

```javascript
const results = await client.sendEmailBatch(emails);

const succeeded = [];
const failed = [];

results.forEach((result, index) => {
  if (result.ErrorCode === 0) {
    succeeded.push({ index, messageId: result.MessageID });
  } else {
    failed.push({
      index,
      errorCode: result.ErrorCode,
      message: result.Message,
      email: emails[index].To
    });
  }
});

console.log(`Sent: ${succeeded.length}, Failed: ${failed.length}`);

if (failed.length > 0) {
  console.log('Failed emails:', failed);

  // Retry only retryable failures
  const retryable = failed.filter(f =>
    f.errorCode === 429 || f.errorCode === 500
  );

  if (retryable.length > 0) {
    const retryEmails = retryable.map(f => emails[f.index]);
    // Retry after delay...
  }
}
```

## Python

### Basic Batch

```python
from postmarker.core import PostmarkClient
import os

postmark = PostmarkClient(server_token=os.environ['POSTMARK_SERVER_TOKEN'])

results = postmark.emails.send_batch(
    {
        'From': 'sender@yourdomain.com',
        'To': 'user1@example.com',
        'Subject': 'Welcome!',
        'TextBody': 'Welcome to our service, User 1.',
        'MessageStream': 'outbound'
    },
    {
        'From': 'sender@yourdomain.com',
        'To': 'user2@example.com',
        'Subject': 'Welcome!',
        'TextBody': 'Welcome to our service, User 2.',
        'MessageStream': 'outbound'
    }
)

for i, result in enumerate(results):
    if result.get('ErrorCode') == 0:
        print(f"Email {i + 1}: sent ({result['MessageID']})")
    else:
        print(f"Email {i + 1}: failed ({result.get('Message', 'Unknown error')})")
```

### Chunking in Python

```python
def chunk_list(lst, size):
    for i in range(0, len(lst), size):
        yield lst[i:i + size]

emails = [
    {
        'From': 'sender@yourdomain.com',
        'To': user['email'],
        'Subject': f'Hello {user["name"]}',
        'TextBody': f'Hi {user["name"]}, here is your update.',
        'MessageStream': 'outbound'
    }
    for user in users
]

all_results = []
for chunk in chunk_list(emails, 500):
    results = postmark.emails.send_batch(*chunk)
    all_results.extend(results)
```

## Ruby

### Basic Batch

```ruby
require 'postmark'

client = Postmark::ApiClient.new(ENV['POSTMARK_SERVER_TOKEN'])

results = client.deliver_messages([
  {
    from: 'sender@yourdomain.com',
    to: 'user1@example.com',
    subject: 'Welcome!',
    text_body: 'Welcome to our service, User 1.',
    message_stream: 'outbound'
  },
  {
    from: 'sender@yourdomain.com',
    to: 'user2@example.com',
    subject: 'Welcome!',
    text_body: 'Welcome to our service, User 2.',
    message_stream: 'outbound'
  }
])
```

## cURL

### Batch Request

```bash
curl "https://api.postmarkapp.com/email/batch" \
  -X POST \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -H "X-Postmark-Server-Token: $POSTMARK_SERVER_TOKEN" \
  -d '[
    {
      "From": "sender@yourdomain.com",
      "To": "user1@example.com",
      "Subject": "Welcome!",
      "TextBody": "Welcome to our service.",
      "MessageStream": "outbound"
    },
    {
      "From": "sender@yourdomain.com",
      "To": "user2@example.com",
      "Subject": "Welcome!",
      "TextBody": "Welcome to our service.",
      "MessageStream": "outbound"
    }
  ]'
```

## Batch with Templates

**Endpoint:** `POST /email/batchWithTemplates`

```javascript
const results = await client.sendEmailBatchWithTemplates([
  {
    From: 'sender@yourdomain.com',
    To: 'user1@example.com',
    TemplateAlias: 'welcome-email',
    TemplateModel: { name: 'User 1', action_url: 'https://app.yourdomain.com' },
    MessageStream: 'outbound'
  },
  {
    From: 'sender@yourdomain.com',
    To: 'user2@example.com',
    TemplateAlias: 'welcome-email',
    TemplateModel: { name: 'User 2', action_url: 'https://app.yourdomain.com' },
    MessageStream: 'outbound'
  }
]);
```

## Key Notes

- Maximum **500 emails** per batch call
- Maximum **50 MB** total payload size
- Each email is independently validated — one failure does not affect others
- Attachments are supported in batch (unlike some alternatives)
- Always check each result's `ErrorCode` — partial failures are possible
- For more than 500 emails, chunk into groups of 500 and send sequentially
