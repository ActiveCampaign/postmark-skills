# Template Email Examples

Examples for sending emails using Postmark server-side templates.

## Endpoints

| Endpoint | Description |
|----------|-------------|
| `POST /email/withTemplate` | Send single email with template |
| `POST /email/batchWithTemplates` | Send batch with templates (up to 500) |

## Node.js / TypeScript

### Send with Template ID

```javascript
const postmark = require('postmark');
const client = new postmark.ServerClient(process.env.POSTMARK_SERVER_TOKEN);

const result = await client.sendEmailWithTemplate({
  From: 'sender@yourdomain.com',
  To: 'customer@example.com',
  TemplateId: 12345,
  TemplateModel: {
    name: 'Jane Doe',
    order_id: 'ORD-67890',
    items: [
      { name: 'Widget', quantity: 2, price: '$19.99' },
      { name: 'Gadget', quantity: 1, price: '$29.99' }
    ],
    total: '$69.97'
  },
  MessageStream: 'outbound'
});

console.log('Sent:', result.MessageID);
```

### Send with Template Alias (Recommended)

```javascript
const result = await client.sendEmailWithTemplate({
  From: 'sender@yourdomain.com',
  To: 'customer@example.com',
  TemplateAlias: 'order-confirmation',
  TemplateModel: {
    name: 'Jane Doe',
    order_id: 'ORD-67890'
  },
  MessageStream: 'outbound',
  Tag: 'order-confirmation',
  Metadata: {
    customer_id: '12345',
    order_id: 'ORD-67890'
  }
});
```

### Batch with Templates

```javascript
const results = await client.sendEmailBatchWithTemplates([
  {
    From: 'sender@yourdomain.com',
    To: 'user1@example.com',
    TemplateAlias: 'welcome-email',
    TemplateModel: {
      name: 'User 1',
      action_url: 'https://app.yourdomain.com/start'
    },
    MessageStream: 'outbound'
  },
  {
    From: 'sender@yourdomain.com',
    To: 'user2@example.com',
    TemplateAlias: 'welcome-email',
    TemplateModel: {
      name: 'User 2',
      action_url: 'https://app.yourdomain.com/start'
    },
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

## Python

### Send with Template

```python
from postmarker.core import PostmarkClient
import os

postmark = PostmarkClient(server_token=os.environ['POSTMARK_SERVER_TOKEN'])

result = postmark.emails.send_with_template(
    From='sender@yourdomain.com',
    To='customer@example.com',
    TemplateAlias='order-confirmation',
    TemplateModel={
        'name': 'Jane Doe',
        'order_id': 'ORD-67890',
        'items': [
            {'name': 'Widget', 'quantity': 2, 'price': '$19.99'},
            {'name': 'Gadget', 'quantity': 1, 'price': '$29.99'}
        ],
        'total': '$69.97'
    },
    MessageStream='outbound'
)

print('Sent:', result['MessageID'])
```

## Ruby

### Send with Template

```ruby
require 'postmark'

client = Postmark::ApiClient.new(ENV['POSTMARK_SERVER_TOKEN'])

result = client.deliver_with_template(
  from: 'sender@yourdomain.com',
  to: 'customer@example.com',
  template_alias: 'order-confirmation',
  template_model: {
    name: 'Jane Doe',
    order_id: 'ORD-67890'
  },
  message_stream: 'outbound'
)

puts "Sent: #{result[:message_id]}"
```

## cURL

### Send with Template

```bash
curl "https://api.postmarkapp.com/email/withTemplate" \
  -X POST \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -H "X-Postmark-Server-Token: $POSTMARK_SERVER_TOKEN" \
  -d '{
    "From": "sender@yourdomain.com",
    "To": "customer@example.com",
    "TemplateAlias": "order-confirmation",
    "TemplateModel": {
      "name": "Jane Doe",
      "order_id": "ORD-67890",
      "items": [
        { "name": "Widget", "quantity": 2, "price": "$19.99" }
      ],
      "total": "$19.99"
    },
    "MessageStream": "outbound"
  }'
```

### Batch with Templates

```bash
curl "https://api.postmarkapp.com/email/batchWithTemplates" \
  -X POST \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -H "X-Postmark-Server-Token: $POSTMARK_SERVER_TOKEN" \
  -d '{
    "Messages": [
      {
        "From": "sender@yourdomain.com",
        "To": "user1@example.com",
        "TemplateAlias": "welcome-email",
        "TemplateModel": { "name": "User 1" },
        "MessageStream": "outbound"
      },
      {
        "From": "sender@yourdomain.com",
        "To": "user2@example.com",
        "TemplateAlias": "welcome-email",
        "TemplateModel": { "name": "User 2" },
        "MessageStream": "outbound"
      }
    ]
  }'
```

## Handlebars Syntax Quick Reference

### Variables

```handlebars
{{name}}              <!-- Escaped output -->
{{{html_content}}}    <!-- Unescaped HTML -->
```

### Conditionals

```handlebars
{{#if premium_member}}
  <p>Premium features included.</p>
{{else}}
  <p>Upgrade for more features.</p>
{{/if}}
```

### Iteration

```handlebars
{{#each items}}
  <tr>
    <td>{{this.name}}</td>
    <td>{{this.price}}</td>
  </tr>
{{/each}}
```

### Nested Objects

```handlebars
{{customer.name}}
{{customer.address.city}}
```

## Key Notes

- Use `TemplateAlias` (string) over `TemplateId` (integer) for portability across environments
- Provide either `TemplateId` or `TemplateAlias`, not both
- `TemplateModel` keys must match the Handlebars variables in your template
- Missing variables render as empty strings — validate your model data
- Templates support the same optional parameters as regular sends (Tag, Metadata, Tracking, etc.)
- Batch templates support up to 500 messages per call
