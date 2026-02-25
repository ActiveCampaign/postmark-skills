# Bounce Management and Suppression

## Delivery Statistics

Get a summary of bounce and delivery activity for your server:

```javascript
const postmark = require('postmark');
const client = new postmark.ServerClient(process.env.POSTMARK_SERVER_TOKEN);

const stats = await client.getDeliveryStatistics();
console.log('Inactive emails:', stats.InactiveMails);
stats.Bounces.forEach(b => console.log(`${b.Name}: ${b.Count}`));
```

---

## Bounces API

### List Bounces

```javascript
const bounces = await client.getBounces({
  count: 100,
  offset: 0,
  type: 'HardBounce',     // optional filter
  messageStream: 'outbound'
});

bounces.Bounces.forEach(b => {
  console.log(`${b.Email} — ${b.Type} (inactive: ${b.Inactive})`);
});
```

### Get a Single Bounce

```javascript
const bounce = await client.getBounce(bounceId);
console.log(bounce.Description);
console.log(bounce.Details); // raw SMTP response
```

### Reactivate a Bounced Recipient

Use only when a soft bounce was genuinely temporary and the issue is resolved:

```javascript
const result = await client.activateBounce(bounceId);
console.log(result.Message); // "OK"
```

**Do not reactivate hard bounces.** The address does not exist — reactivating results in another bounce.

---

## Suppression Management

Suppressions prevent emails from being sent to specific recipients, managed per message stream.

### List Suppressions

```javascript
const result = await client.getSuppressions('outbound');

result.Suppressions.forEach(s => {
  console.log(`${s.EmailAddress} — ${s.SuppressionReason} (${s.Origin})`);
});
```

### Manually Suppress a Recipient

```javascript
await client.createSuppressions('outbound', {
  Suppressions: [{ EmailAddress: 'user@example.com' }]
});
```

### Remove a Suppression

Only remove when a recipient has explicitly re-opted in:

```javascript
await client.deleteSuppressions('outbound', {
  Suppressions: [{ EmailAddress: 'user@example.com' }]
});
```

**Never remove suppressions for spam complaints or hard bounces.** Doing so damages sender reputation.

---

## Suppression Reasons

| Reason | Origin | Can Remove? |
|--------|--------|-------------|
| `HardBounce` | Automatic | Only if confirmed deliverable |
| `SpamComplaint` | Automatic | No — never re-add |
| `ManualSuppression` | API or dashboard | Yes, with fresh consent |
| `Unsubscribe` | Recipient action | Only with explicit re-opt-in |

---

## Bounce Handling Strategy

| Bounce Type | Action |
|-------------|--------|
| `HardBounce` | Permanently remove — address does not exist |
| `SoftBounce` | Log and monitor — Postmark retries; don't remove yet |
| `SpamComplaint` | Immediately suppress — never send again |
| `Blocked` | Log and investigate — may indicate content/reputation issue |
| `DMARCPolicy` | Fix SPF/DKIM/DMARC configuration |

## Bounce Rate Thresholds

| Metric | Warning | Critical |
|--------|---------|----------|
| Bounce rate | > 2% | > 4% |
| Spam complaint rate | > 0.04% | > 0.08% |

High bounce rates are typically caused by sending to old or purchased lists, not removing hard bounces promptly, or domain warm-up that is too aggressive.
