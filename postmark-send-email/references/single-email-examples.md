# Single Email Examples

Complete examples for sending a single email via `POST /email` in all supported SDKs.

## Node.js / TypeScript

### Basic Send

```javascript
const postmark = require('postmark');
const client = new postmark.ServerClient(process.env.POSTMARK_SERVER_TOKEN);

const result = await client.sendEmail({
  From: 'sender@yourdomain.com',
  To: 'recipient@example.com',
  Subject: 'Hello from Postmark',
  TextBody: 'This is a plain text email.',
  HtmlBody: '<p>This is an <strong>HTML</strong> email.</p>',
  MessageStream: 'outbound'
});

console.log('Sent:', result.MessageID);
```

### With All Options

```javascript
const result = await client.sendEmail({
  From: 'sender@yourdomain.com',
  To: 'recipient@example.com',
  Cc: 'cc@example.com',
  Bcc: 'bcc@example.com',
  ReplyTo: 'reply@yourdomain.com',
  Subject: 'Order Confirmation',
  TextBody: 'Your order #12345 is confirmed.',
  HtmlBody: '<h1>Order Confirmed</h1><p>Your order #12345 is confirmed.</p>',
  MessageStream: 'outbound',
  Tag: 'order-confirmation',
  Metadata: {
    customer_id: '12345',
    order_id: 'ORD-67890'
  },
  TrackOpens: true,
  TrackLinks: 'HtmlAndText',
  Headers: [
    { Name: 'X-Custom-Header', Value: 'custom-value' }
  ]
});
```

### With Attachments

```javascript
const fs = require('fs');

const result = await client.sendEmail({
  From: 'sender@yourdomain.com',
  To: 'recipient@example.com',
  Subject: 'Your Invoice',
  TextBody: 'Please find your invoice attached.',
  MessageStream: 'outbound',
  Attachments: [
    {
      Name: 'invoice.pdf',
      Content: fs.readFileSync('./invoice.pdf').toString('base64'),
      ContentType: 'application/pdf'
    }
  ]
});
```

### With Inline Image

```javascript
const result = await client.sendEmail({
  From: 'sender@yourdomain.com',
  To: 'recipient@example.com',
  Subject: 'Newsletter',
  HtmlBody: '<h1>Our Newsletter</h1><img src="cid:logo">',
  MessageStream: 'broadcast',
  Attachments: [
    {
      Name: 'logo.png',
      Content: fs.readFileSync('./logo.png').toString('base64'),
      ContentType: 'image/png',
      ContentID: 'cid:logo'
    }
  ]
});
```

## Python

### Basic Send

```python
from postmarker.core import PostmarkClient
import os

postmark = PostmarkClient(server_token=os.environ['POSTMARK_SERVER_TOKEN'])

result = postmark.emails.send(
    From='sender@yourdomain.com',
    To='recipient@example.com',
    Subject='Hello from Postmark',
    TextBody='This is a plain text email.',
    HtmlBody='<p>This is an <strong>HTML</strong> email.</p>',
    MessageStream='outbound'
)

print('Sent:', result['MessageID'])
```

### With All Options

```python
result = postmark.emails.send(
    From='sender@yourdomain.com',
    To='recipient@example.com',
    Cc='cc@example.com',
    Bcc='bcc@example.com',
    ReplyTo='reply@yourdomain.com',
    Subject='Order Confirmation',
    TextBody='Your order #12345 is confirmed.',
    HtmlBody='<h1>Order Confirmed</h1><p>Your order #12345 is confirmed.</p>',
    MessageStream='outbound',
    Tag='order-confirmation',
    Metadata={
        'customer_id': '12345',
        'order_id': 'ORD-67890'
    },
    TrackOpens=True,
    TrackLinks='HtmlAndText',
    Headers=[
        {'Name': 'X-Custom-Header', 'Value': 'custom-value'}
    ]
)
```

### With Attachments

```python
import base64

with open('./invoice.pdf', 'rb') as f:
    content = base64.b64encode(f.read()).decode('utf-8')

result = postmark.emails.send(
    From='sender@yourdomain.com',
    To='recipient@example.com',
    Subject='Your Invoice',
    TextBody='Please find your invoice attached.',
    MessageStream='outbound',
    Attachments=[
        {
            'Name': 'invoice.pdf',
            'Content': content,
            'ContentType': 'application/pdf'
        }
    ]
)
```

## Ruby

### Basic Send

```ruby
require 'postmark'

client = Postmark::ApiClient.new(ENV['POSTMARK_SERVER_TOKEN'])

result = client.deliver(
  from: 'sender@yourdomain.com',
  to: 'recipient@example.com',
  subject: 'Hello from Postmark',
  text_body: 'This is a plain text email.',
  html_body: '<p>This is an <strong>HTML</strong> email.</p>',
  message_stream: 'outbound'
)

puts "Sent: #{result[:message_id]}"
```

## PHP

### Basic Send

```php
use Postmark\PostmarkClient;

$client = new PostmarkClient(getenv('POSTMARK_SERVER_TOKEN'));

$result = $client->sendEmail(
    'sender@yourdomain.com',       // From
    'recipient@example.com',        // To
    'Hello from Postmark',          // Subject
    '<p>This is an <strong>HTML</strong> email.</p>', // HtmlBody
    'This is a plain text email.',  // TextBody
    null,                           // Tag
    true,                           // TrackOpens
    null,                           // ReplyTo
    null,                           // Cc
    null,                           // Bcc
    null,                           // Headers
    null,                           // Attachments
    'outbound'                      // MessageStream
);

echo 'Sent: ' . $result->MessageID;
```

## .NET

### Basic Send

```csharp
using PostmarkDotNet;

var client = new PostmarkClient(Environment.GetEnvironmentVariable("POSTMARK_SERVER_TOKEN"));

var message = new PostmarkMessage
{
    From = "sender@yourdomain.com",
    To = "recipient@example.com",
    Subject = "Hello from Postmark",
    TextBody = "This is a plain text email.",
    HtmlBody = "<p>This is an <strong>HTML</strong> email.</p>",
    MessageStream = "outbound"
};

var result = await client.SendMessageAsync(message);
Console.WriteLine($"Sent: {result.MessageID}");
```

## cURL

### Basic Send

```bash
curl "https://api.postmarkapp.com/email" \
  -X POST \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -H "X-Postmark-Server-Token: $POSTMARK_SERVER_TOKEN" \
  -d '{
    "From": "sender@yourdomain.com",
    "To": "recipient@example.com",
    "Subject": "Hello from Postmark",
    "TextBody": "This is a plain text email.",
    "HtmlBody": "<p>This is an <strong>HTML</strong> email.</p>",
    "MessageStream": "outbound"
  }'
```

### With Metadata and Tracking

```bash
curl "https://api.postmarkapp.com/email" \
  -X POST \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -H "X-Postmark-Server-Token: $POSTMARK_SERVER_TOKEN" \
  -d '{
    "From": "sender@yourdomain.com",
    "To": "recipient@example.com",
    "Subject": "Order Shipped",
    "TextBody": "Your order has shipped.",
    "HtmlBody": "<p>Your order has shipped.</p>",
    "MessageStream": "outbound",
    "Tag": "order-shipped",
    "TrackOpens": true,
    "TrackLinks": "HtmlAndText",
    "Metadata": {
      "customer_id": "12345",
      "order_id": "ORD-67890"
    }
  }'
```
