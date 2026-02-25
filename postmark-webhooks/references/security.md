# Webhook Security

Verify that webhook requests are genuinely from Postmark before processing them.

## Option 1: HTTP Basic Authentication (Recommended)

Set credentials when creating the webhook — Postmark includes them in every request via the `Authorization` header.

### Set credentials on the webhook

```javascript
const webhook = await client.createWebhook({
  Url: 'https://yourdomain.com/webhooks/postmark',
  MessageStream: 'outbound',
  HttpAuth: {
    Username: 'postmark-webhook',
    Password: process.env.WEBHOOK_SECRET
  },
  Triggers: { /* ... */ }
});
```

### Validate in your endpoint (Node.js)

```javascript
app.post('/webhooks/postmark', (req, res) => {
  const authHeader = req.headers['authorization'];
  if (!authHeader) return res.sendStatus(401);

  const [scheme, encoded] = authHeader.split(' ');
  if (scheme !== 'Basic') return res.sendStatus(401);

  const decoded = Buffer.from(encoded, 'base64').toString('utf-8');
  const [username, password] = decoded.split(':');

  if (username !== 'postmark-webhook' || password !== process.env.WEBHOOK_SECRET) {
    return res.sendStatus(401);
  }

  res.sendStatus(200);
  // process event...
});
```

### Validate in your endpoint (Python)

```python
import base64, os
from flask import Flask, request

app = Flask(__name__)

@app.route('/webhooks/postmark', methods=['POST'])
def handle_webhook():
    auth = request.headers.get('Authorization', '')
    if not auth.startswith('Basic '):
        return '', 401

    decoded = base64.b64decode(auth[6:]).decode('utf-8')
    username, _, password = decoded.partition(':')

    if username != 'postmark-webhook' or password != os.environ['WEBHOOK_SECRET']:
        return '', 401

    return '', 200
```

---

## Option 2: Custom HTTP Headers (Shared Secret)

```javascript
// Set the header when creating the webhook
const webhook = await client.createWebhook({
  Url: 'https://yourdomain.com/webhooks/postmark',
  HttpHeaders: [
    { Name: 'X-Webhook-Secret', Value: process.env.WEBHOOK_SECRET }
  ],
  Triggers: { /* ... */ }
});

// Validate in your endpoint
app.post('/webhooks/postmark', (req, res) => {
  const secret = req.headers['x-webhook-secret'];
  if (!secret || secret !== process.env.WEBHOOK_SECRET) {
    return res.sendStatus(401);
  }
  res.sendStatus(200);
});
```

---

## Option 3: IP Allowlisting

Restrict your endpoint to Postmark's IP ranges at the network level (firewall, load balancer ACLs). Check [Postmark's documentation](https://postmarkapp.com/developer/webhooks/webhooks-overview) for the current IP list — it can change, so don't hardcode it.

---

## Security Best Practices

| Practice | Why |
|----------|-----|
| Always use HTTPS | Prevents credentials being intercepted in transit |
| Return 401 for auth failures, not 403 | A 403 stops all Postmark retries permanently |
| Use constant-time comparison | Prevents timing attacks |
| Rotate secrets periodically | Limits exposure if a secret is compromised |

### Constant-time comparison (Node.js)

```javascript
const crypto = require('crypto');

function safeCompare(a, b) {
  const bufA = Buffer.from(String(a));
  const bufB = Buffer.from(String(b));
  if (bufA.length !== bufB.length) return false;
  return crypto.timingSafeEqual(bufA, bufB);
}

app.post('/webhooks/postmark', (req, res) => {
  const incoming = req.headers['x-webhook-secret'] || '';
  if (!safeCompare(incoming, process.env.WEBHOOK_SECRET)) {
    return res.sendStatus(401);
  }
  res.sendStatus(200);
});
```
