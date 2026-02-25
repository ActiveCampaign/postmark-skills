# Sending Reliability

Production email systems must handle failures gracefully, avoid duplicate sends, and stay within rate limits. Email that silently fails or duplicates triggers causes poor user experience and support burden.

## Idempotency

Email is not inherently idempotent — if your send fails mid-process and you retry, you may send the same email twice. Build idempotency into your system before this happens.

### Idempotency Key Pattern

Generate a deterministic key from the email context before sending. Check it before sending and record it after:

```javascript
const crypto = require('crypto');

async function sendEmailIdempotent({ to, subject, templateAlias, templateModel, eventType, eventId }) {
  // Generate a deterministic key from the event context
  const idempotencyKey = crypto
    .createHash('sha256')
    .update(`${eventType}:${eventId}:${to}`)
    .digest('hex');

  // Check if this email was already sent
  const existing = await db.emailLog.findOne({ idempotencyKey });
  if (existing) {
    console.log(`Skipping duplicate send for key ${idempotencyKey}`);
    return existing;
  }

  const result = await client.sendEmailWithTemplate({
    From: 'no-reply@yourdomain.com',
    To: to,
    TemplateAlias: templateAlias,
    TemplateModel: templateModel,
    MessageStream: 'outbound'
  });

  // Record after successful send
  await db.emailLog.insert({
    idempotencyKey,
    messageId: result.MessageID,
    to,
    subject,
    sentAt: new Date()
  });

  return result;
}

// Usage
await sendEmailIdempotent({
  to: 'user@example.com',
  subject: 'Order Confirmed',
  templateAlias: 'order-confirmation',
  templateModel: { order_id: 'ORD-12345', total: '$49.00' },
  eventType: 'order.confirmed',
  eventId: 'ORD-12345'
});
```

### What to Use as the Idempotency Basis

| Event Type | Key Components |
|-----------|----------------|
| Order confirmation | `order.confirmed:{orderId}:{email}` |
| Password reset | `auth.reset:{userId}:{tokenId}` |
| Welcome email | `onboarding.welcome:{userId}` |
| Shipping notification | `shipment.shipped:{shipmentId}:{email}` |
| Invoice | `billing.invoice:{invoiceId}:{email}` |

---

## Retry Logic

Postmark occasionally returns transient errors (rate limit, temporary server error). Build retries with exponential backoff to handle these without hammering the API.

### Exponential Backoff with Jitter

```javascript
async function sendWithRetry(emailParams, options = {}) {
  const { maxAttempts = 3, baseDelayMs = 1000 } = options;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await client.sendEmail(emailParams);
    } catch (error) {
      const isRetryable = isRetryableError(error);

      if (!isRetryable || attempt === maxAttempts) {
        throw error;
      }

      // Exponential backoff with jitter: base * 2^attempt + random 0–1s
      const delay = baseDelayMs * Math.pow(2, attempt) + Math.random() * 1000;
      console.warn(`Send attempt ${attempt} failed (${error.code}). Retrying in ${Math.round(delay)}ms`);
      await sleep(delay);
    }
  }
}

function isRetryableError(error) {
  // Postmark error codes: https://postmarkapp.com/developer/api/overview#error-codes
  const retryableCodes = [
    429,  // Rate limit exceeded
    500,  // Internal server error
    503,  // Service unavailable
  ];
  return retryableCodes.includes(error.statusCode);
}

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// Non-retryable errors — fix the request, don't retry
// 401: Invalid API token
// 422: Invalid request (bad email address, invalid template, etc.)
// 406: Recipient is suppressed
// 300: Inactive recipient
```

---

## Rate Limits

Postmark rate limits vary by plan. Exceeding limits returns HTTP 429.

| Plan | Default Rate Limit |
|------|--------------------|
| Developer | 25 emails/second |
| Starter | 50 emails/second |
| Pro | 100 emails/second |
| Custom | Contact Postmark |

### Handling Rate Limits

For bulk sending, pace your requests to stay under the limit:

```javascript
const pLimit = require('p-limit'); // npm install p-limit

async function sendBulkEmail(emailList) {
  // Limit concurrent sends to 25/second
  const limit = pLimit(25);

  const results = await Promise.allSettled(
    emailList.map(email =>
      limit(() => sendWithRetry({
        From: 'sender@yourdomain.com',
        To: email.address,
        TemplateAlias: email.template,
        TemplateModel: email.model,
        MessageStream: 'outbound'
      }))
    )
  );

  const failures = results.filter(r => r.status === 'rejected');
  if (failures.length > 0) {
    console.error(`${failures.length} sends failed:`, failures.map(f => f.reason));
  }

  return results;
}
```

For high-volume sends (thousands per batch), use Postmark's **Batch Send API** — one API call, up to 500 emails:

```javascript
// More efficient than individual calls for batch scenarios
const batch = emailList.slice(0, 500).map(email => ({
  From: 'sender@yourdomain.com',
  To: email.address,
  TemplateAlias: 'newsletter',
  TemplateModel: email.model,
  MessageStream: 'broadcast'
}));

const results = await client.sendEmailBatch(batch);
// results is an array of per-email results
```

---

## Error Codes and Handling

| HTTP Status | Postmark Code | Meaning | Action |
|-------------|--------------|---------|--------|
| 401 | 10 | Invalid API token | Fix token in config |
| 406 | 406 | Inactive recipient (suppressed) | Remove from list — do not retry |
| 422 | 405 | Not allowed to send | Check message stream type |
| 422 | 407 | Invalid template alias | Fix template name |
| 422 | 408 | Template model mismatch | Fix template model |
| 429 | — | Rate limit exceeded | Back off and retry |
| 500 | — | Server error | Retry with backoff |

Full error code reference: https://postmarkapp.com/developer/api/overview#error-codes

---

## Email Queue Pattern

For high-reliability requirements, don't send email synchronously from your request handlers. Use a queue:

```javascript
// Instead of sending in the request handler:
app.post('/checkout', async (req, res) => {
  const order = await processOrder(req.body);

  // Enqueue — don't await delivery
  await emailQueue.add('order-confirmation', {
    orderId: order.id,
    email: order.customerEmail,
    total: order.total
  });

  res.json({ orderId: order.id, status: 'confirmed' });
});

// Separate worker process:
emailQueue.process('order-confirmation', async (job) => {
  const { orderId, email, total } = job.data;

  return sendEmailIdempotent({
    to: email,
    eventType: 'order.confirmed',
    eventId: orderId,
    templateAlias: 'order-confirmation',
    templateModel: { order_id: orderId, total }
  });
});
```

**Benefits:**
- Request handlers return immediately
- Failed sends can be retried without re-running the full order flow
- Idempotency keys prevent duplicates on retry
- Send volume can be smoothed to avoid rate limit spikes

---

## Monitoring

Log and monitor these signals in production:

| Signal | Alert Threshold | Meaning |
|--------|-----------------|---------|
| Send error rate | > 1% | API issues or bad addresses |
| Bounce rate | > 2% | List quality or domain issue |
| Spam complaint rate | > 0.04% | Content or targeting problem |
| Queue depth | Growing > 5 min | Worker is falling behind |
| P95 send latency | > 5s | API or network issue |
