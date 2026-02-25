# Webhook Handler Examples

## Node.js / Express — Full Handler

```javascript
const express = require('express');
const app = express();
app.use(express.json());

app.post('/webhooks/postmark', (req, res) => {
  // Respond 200 immediately — process asynchronously if needed
  res.sendStatus(200);

  const event = req.body;

  switch (event.RecordType) {
    case 'Delivery':     handleDelivery(event); break;
    case 'Bounce':       handleBounce(event); break;
    case 'SpamComplaint': handleSpamComplaint(event); break;
    case 'Open':         handleOpen(event); break;
    case 'Click':        handleClick(event); break;
    case 'SubscriptionChange': handleSubscriptionChange(event); break;
    default:
      console.log('Unknown event type:', event.RecordType);
  }
});

function handleDelivery(event) {
  console.log(`Delivered to ${event.Recipient} at ${event.DeliveredAt}`);
  // db.emails.update({ messageId: event.MessageID }, { status: 'delivered' });
}

function handleBounce(event) {
  console.log(`Bounce (${event.Type}) for ${event.Email}: ${event.Description}`);

  if (event.Type === 'HardBounce') {
    // Permanently remove — address does not exist
    // db.contacts.update({ email: event.Email }, { status: 'invalid' });
  }

  if (event.Inactive) {
    // Postmark has deactivated this recipient
    console.log(`Recipient ${event.Email} marked inactive by Postmark`);
  }
}

function handleSpamComplaint(event) {
  console.log(`Spam complaint from ${event.Email}`);
  // Permanently suppress — never send again
  // db.suppressions.insert({ email: event.Email, reason: 'spam_complaint' });
}

function handleOpen(event) {
  if (event.FirstOpen) {
    console.log(`First open by ${event.Recipient} on ${event.Platform}`);
  }
  // db.emailEvents.insert({ messageId: event.MessageID, type: 'open' });
}

function handleClick(event) {
  console.log(`Click by ${event.Recipient}: ${event.OriginalLink}`);
  // db.emailEvents.insert({ messageId: event.MessageID, type: 'click', url: event.OriginalLink });
}

function handleSubscriptionChange(event) {
  console.log(`Unsubscribe: ${event.Recipient} (stream: ${event.MessageStream})`);
  if (event.SuppressSending) {
    // Sync to your own system
    // db.contacts.update({ email: event.Recipient }, { subscribed: false });
  }
}
```

## Python / Flask — Full Handler

```python
from flask import Flask, request

app = Flask(__name__)

@app.route('/webhooks/postmark', methods=['POST'])
def handle_webhook():
    event = request.get_json()
    record_type = event.get('RecordType')

    handlers = {
        'Delivery': handle_delivery,
        'Bounce': handle_bounce,
        'SpamComplaint': handle_spam_complaint,
        'Open': handle_open,
        'Click': handle_click,
        'SubscriptionChange': handle_subscription_change,
    }

    handler = handlers.get(record_type)
    if handler:
        handler(event)
    else:
        print(f"Unknown event type: {record_type}")

    return '', 200

def handle_delivery(event):
    print(f"Delivered to {event['Recipient']} at {event['DeliveredAt']}")

def handle_bounce(event):
    bounce_type = event['Type']
    print(f"Bounce ({bounce_type}) for {event['Email']}")
    if bounce_type == 'HardBounce':
        pass  # permanently remove from lists
    if event.get('Inactive'):
        print(f"Recipient {event['Email']} marked inactive by Postmark")

def handle_spam_complaint(event):
    print(f"Spam complaint from {event['Email']}")
    # permanently suppress

def handle_open(event):
    if event.get('FirstOpen'):
        print(f"First open by {event['Recipient']} on {event.get('Platform')}")

def handle_click(event):
    print(f"Click by {event['Recipient']}: {event.get('OriginalLink')}")

def handle_subscription_change(event):
    print(f"Unsubscribe: {event['Recipient']} (stream: {event['MessageStream']})")
    if event.get('SuppressSending'):
        pass  # update your subscription records
```

## Async Processing

For slow operations (database writes, API calls), respond 200 immediately and queue the work:

```javascript
const Queue = require('bull');
const webhookQueue = new Queue('postmark-webhooks', process.env.REDIS_URL);

app.post('/webhooks/postmark', (req, res) => {
  webhookQueue.add(req.body);
  res.sendStatus(200); // respond before processing
});

webhookQueue.process(async (job) => {
  const event = job.data;
  await processWebhookEvent(event);
});
```

## Deduplication

Postmark may deliver webhooks more than once. Use `MessageID` + `RecordType` to deduplicate:

```javascript
// Use Redis or a database in production instead of a Set
const processedEvents = new Set();

app.post('/webhooks/postmark', (req, res) => {
  res.sendStatus(200);

  const event = req.body;
  const key = `${event.RecordType}-${event.MessageID}`;

  if (processedEvents.has(key)) return;
  processedEvents.add(key);

  processEvent(event);
});
```

## Correlating Events with Sent Emails

Include `Metadata` when sending to tie webhook events back to your records:

```javascript
// When sending
await client.sendEmail({
  From: 'notifications@yourdomain.com',
  To: customer.email,
  Subject: 'Your order has shipped',
  TextBody: '...',
  MessageStream: 'outbound',
  Metadata: {
    customer_id: customer.id,
    order_id: order.id
  }
});

// In your webhook handler — Metadata comes back in every event type
function handleDelivery(event) {
  const { customer_id, order_id } = event.Metadata || {};
  if (order_id) {
    // db.orders.update({ id: order_id }, { emailDelivered: true });
  }
}
```
