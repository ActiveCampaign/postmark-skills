# Error Handling

Complete reference for handling Postmark API errors.

## HTTP Status Codes

| Code | Meaning | Retryable | Action |
|------|---------|-----------|--------|
| 200 | Success | — | Continue |
| 401 | Unauthorized | No | Check `X-Postmark-Server-Token` value |
| 406 | Inactive recipient | No | Recipient is suppressed — check suppression list |
| 409 | JSON required | No | Set `Content-Type: application/json` and `Accept: application/json` |
| 410 | Too many batch messages | No | Reduce batch to 500 or fewer messages |
| 413 | Payload too large | No | Reduce payload size (10 MB single, 50 MB batch) |
| 422 | Validation error | No | Fix request parameters (invalid From, missing required fields, etc.) |
| 429 | Rate limited | Yes | Retry with exponential backoff |
| 500 | Server error | Yes | Retry with exponential backoff |

## Postmark Error Codes

Beyond HTTP status, Postmark returns specific `ErrorCode` values in the response body:

| ErrorCode | Meaning |
|-----------|---------|
| 0 | No error (success) |
| 10 | Bad or missing API token |
| 300 | Invalid email request |
| 406 | Inactive recipient (suppressed due to bounce/complaint) |
| 409 | JSON body required |
| 410 | Too many batch messages (max 500) |

## Retry Strategy

Only retry **429** (rate limited) and **500** (server error). Never retry 401, 406, 422, or other client errors.

### Node.js

```javascript
async function sendWithRetry(client, email, maxRetries = 3) {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await client.sendEmail(email);
    } catch (error) {
      const statusCode = error.statusCode || error.code;
      const isRetryable = statusCode === 429 || statusCode === 500;

      if (!isRetryable || attempt === maxRetries) {
        throw error;
      }

      const delay = Math.pow(2, attempt) * 1000; // 1s, 2s, 4s
      console.log(`Attempt ${attempt + 1} failed (${statusCode}), retrying in ${delay}ms...`);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}

// Usage
try {
  const result = await sendWithRetry(client, {
    From: 'sender@yourdomain.com',
    To: 'recipient@example.com',
    Subject: 'Hello',
    TextBody: 'Hello!',
    MessageStream: 'outbound'
  });
  console.log('Sent:', result.MessageID);
} catch (error) {
  console.error('Failed after retries:', error.message);
}
```

### Python

```python
import time

def send_with_retry(postmark, email, max_retries=3):
    for attempt in range(max_retries + 1):
        try:
            return postmark.emails.send(**email)
        except Exception as e:
            status_code = getattr(e, 'status_code', None)
            is_retryable = status_code in (429, 500)

            if not is_retryable or attempt == max_retries:
                raise

            delay = (2 ** attempt)
            print(f"Attempt {attempt + 1} failed ({status_code}), retrying in {delay}s...")
            time.sleep(delay)

# Usage
try:
    result = send_with_retry(postmark, {
        'From': 'sender@yourdomain.com',
        'To': 'recipient@example.com',
        'Subject': 'Hello',
        'TextBody': 'Hello!',
        'MessageStream': 'outbound'
    })
    print('Sent:', result['MessageID'])
except Exception as e:
    print('Failed after retries:', str(e))
```

## Handling Batch Partial Failures

Batch responses return individual results. Some emails may succeed while others fail:

```javascript
const results = await client.sendEmailBatch(emails);

const succeeded = [];
const permanentFailures = [];
const retryableFailures = [];

results.forEach((result, index) => {
  if (result.ErrorCode === 0) {
    succeeded.push({ index, messageId: result.MessageID });
  } else if (result.ErrorCode === 429 || result.ErrorCode >= 500) {
    retryableFailures.push({ index, error: result.Message });
  } else {
    permanentFailures.push({ index, error: result.Message, code: result.ErrorCode });
  }
});

console.log(`Sent: ${succeeded.length}`);
console.log(`Permanent failures: ${permanentFailures.length}`);
console.log(`Retryable failures: ${retryableFailures.length}`);

// Retry only retryable failures
if (retryableFailures.length > 0) {
  const retryEmails = retryableFailures.map(f => emails[f.index]);
  await new Promise(r => setTimeout(r, 2000));
  const retryResults = await client.sendEmailBatch(retryEmails);
  // Process retry results...
}
```

## Handling Inactive Recipients (406)

When a recipient is suppressed (due to previous bounce or spam complaint):

```javascript
const postmark = require('postmark');

try {
  await client.sendEmail(emailData);
} catch (error) {
  if (error.statusCode === 406) {
    // Recipient is inactive — check suppression list
    console.log('Inactive recipient:', emailData.To);

    // Optionally check the suppression list
    const suppressions = await client.getSuppressions('outbound');
    const suppressed = suppressions.Suppressions.find(
      s => s.EmailAddress === emailData.To
    );

    if (suppressed) {
      console.log('Suppression reason:', suppressed.SuppressionReason);
      // SuppressionReason: "HardBounce" | "SpamComplaint" | "ManualSuppression"
    }
  }
}
```

## Error Object Structure (Node.js SDK)

The `postmark` SDK throws typed errors:

```javascript
const { Errors } = require('postmark');

try {
  await client.sendEmail(emailData);
} catch (error) {
  if (error instanceof Errors.InactiveRecipientsError) {
    // 406 — Inactive recipient
    console.log('Inactive recipients:', error.recipients);
  } else if (error instanceof Errors.InvalidAPIKeyError) {
    // 401 — Bad API token
    console.log('Invalid API key');
  } else if (error instanceof Errors.RateLimitExceededError) {
    // 429 — Rate limited
    console.log('Rate limited, retry after delay');
  } else if (error instanceof Errors.ApiInputError) {
    // 422 — Validation error
    console.log('Invalid input:', error.message);
  } else if (error instanceof Errors.InternalServerError) {
    // 500 — Server error
    console.log('Server error, retry');
  }
}
```

## Key Guidelines

- **Never retry** 401, 406, 409, 410, 413, or 422 errors — fix the request
- **Always retry** 429 and 500 with exponential backoff (1s, 2s, 4s)
- **Max retries:** 3–5 for most use cases
- **Batch responses** can have mixed results — check each entry
- **406 (Inactive recipient)** means the address was suppressed — check suppression list for reason
- **Postmark handles rate limiting** automatically for optimal deliverability — 429 errors are rare
