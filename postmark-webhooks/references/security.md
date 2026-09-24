# Webhook Security

Verify that webhook requests are genuinely from Postmark before processing them.

## Postmark Does Not Sign Webhooks

There is no HMAC signature, no signing secret, and no signature header on a Postmark webhook request. **Do not write or trust signature-verification logic** — there is nothing to verify against.

Protect your endpoint with these mechanisms instead:

| Mechanism | Covered in |
|-----------|-----------|
| HTTP Basic Auth (credentials in the webhook URL) | Option 1 below |
| IP allowlisting | Option 3 below |
| Payload-shape validation | Option 4 below |

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

## Option 4: Payload-Shape Validation

Reject anything that does not look like a Postmark event before you act on it. This costs nothing and catches malformed or spoofed bodies that pass transport-level checks.

```javascript
const VALID_RECORD_TYPES = new Set([
  'Delivery', 'Bounce', 'SpamComplaint', 'Open', 'Click', 'SubscriptionChange'
]);

function isValidPostmarkEvent(body) {
  if (!body || typeof body !== 'object') return false;
  if (!VALID_RECORD_TYPES.has(body.RecordType)) return false;
  if (typeof body.MessageID !== 'string') return false;
  return true;
}

app.post('/webhooks/postmark', (req, res) => {
  if (!isValidPostmarkEvent(req.body)) return res.sendStatus(400);

  res.sendStatus(200);
  // process event...
});
```

```python
VALID_RECORD_TYPES = {
    'Delivery', 'Bounce', 'SpamComplaint', 'Open', 'Click', 'SubscriptionChange'
}

def is_valid_postmark_event(body):
    if not isinstance(body, dict):
        return False
    if body.get('RecordType') not in VALID_RECORD_TYPES:
        return False
    if not isinstance(body.get('MessageID'), str):
        return False
    return True
```

---

## Security Best Practices

| Practice | Why |
|----------|-----|
| Always use HTTPS | Prevents credentials being intercepted in transit |
| Don't rely on signature verification | Postmark does not sign webhooks — use Basic Auth, IP allowlisting, and payload-shape validation |
| Return 401 for auth failures | 401 is the semantically correct response. Note that for outbound webhooks every 4xx except 408 and 429 is permanent — a 401 or 403 drops the event on the first attempt with no retries |
| Fix auth misconfiguration immediately | Because auth failures are never retried, a bad credential silently loses events. Watch `FailureCount` and `SuccessRate` via `GET /webhooks/{Id}/statistics` |
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
