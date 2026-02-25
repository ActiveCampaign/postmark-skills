# List Management

Sending to a healthy list is the single most impactful thing you can do for deliverability. Invalid, inactive, and unengaged addresses are the root cause of most deliverability problems.

## Bounce Types and Actions

| Bounce Type | Meaning | Required Action |
|-------------|---------|-----------------|
| **Hard bounce** | Permanent failure — address doesn't exist or domain is gone | Suppress immediately, permanently |
| **Soft bounce** | Temporary failure — mailbox full, server down | Monitor; suppress after 3–5 consecutive soft bounces |
| **Transient** | Temporary delay — not a bounce yet | Postmark retries automatically; no action needed |
| **Spam complaint** | Recipient marked as spam | Suppress immediately, permanently — never re-add |

### Handling Bounces via Webhook

```javascript
// Postmark Bounce webhook handler
function handleBounce(event) {
  if (event.Type === 'HardBounce' || event.Type === 'SpamComplaint') {
    // Suppress immediately
    db.suppressions.upsert({
      email: event.Email,
      reason: event.Type,
      suppressedAt: event.BouncedAt,
      permanent: true
    });
    // Remove from all lists/audiences
    db.subscribers.update(
      { email: event.Email },
      { status: 'suppressed', suppressedAt: new Date() }
    );
  } else if (event.Type === 'SoftBounce') {
    // Increment soft bounce counter; suppress after threshold
    const softBounces = db.softBounces.increment(event.Email);
    if (softBounces >= 3) {
      db.suppressions.upsert({ email: event.Email, reason: 'SoftBounce' });
    }
  }
}
```

---

## Suppression Management

Postmark maintains suppression lists per Message Stream. When an address is suppressed, Postmark blocks delivery and returns a `406` status — you are not charged for suppressed sends.

### Postmark Suppression API

```javascript
const postmark = require('postmark');
const client = new postmark.ServerClient(process.env.POSTMARK_SERVER_TOKEN);

// Get all suppressions for the broadcast stream
const suppressions = await client.getSuppressions('broadcast');
// { Suppressions: [{ EmailAddress, SuppressionReason, CreatedAt, ... }] }

// Create a suppression manually
await client.createSuppressions('broadcast', {
  Suppressions: [
    { EmailAddress: 'user@example.com' }
  ]
});

// Delete a suppression (re-activate) — only if user gave fresh consent
await client.deleteSuppressions('broadcast', {
  Suppressions: [
    { EmailAddress: 'user@example.com' }
  ]
});
```

### Your Database vs. Postmark's Suppression List

Maintain your own suppression list. Don't rely solely on Postmark's:
- Enables checking before initiating a send
- Survives if you change email providers
- Required for GDPR compliance (right to erasure tracking)

```javascript
async function canSendEmail(emailAddress, streamType = 'outbound') {
  // Check your own suppression list first
  const suppressed = await db.suppressions.findOne({ email: emailAddress });
  if (suppressed) return false;

  // For broadcast, also verify they're actively subscribed
  if (streamType === 'broadcast') {
    const subscriber = await db.subscribers.findOne({ email: emailAddress });
    if (!subscriber || subscriber.status !== 'subscribed') return false;
  }

  return true;
}
```

---

## List Hygiene Schedule

| Frequency | Action |
|-----------|--------|
| **Immediate** | Suppress hard bounces and spam complaints |
| **After each send** | Review bounce and complaint rates |
| **Monthly** | Remove soft bounce accumulations past threshold |
| **Quarterly** | Identify and segment unengaged subscribers (no opens/clicks in 90 days) |
| **Every 12–18 months** | Run re-engagement campaign; remove non-responders |
| **Annually** | Audit consent records for GDPR/CASL compliance |

---

## Re-engagement Campaigns

Before removing inactive subscribers, attempt re-engagement. This recovers some subscribers and ensures you're only removing genuinely disinterested recipients.

### Recommended Pattern

1. **Segment**: Subscribers with no opens or clicks in 90–180 days
2. **Send re-permission email**: Ask if they want to stay subscribed
3. **Wait 7 days**: No response = unsubscribe
4. **Send final notice**: "This is your last email from us unless you opt in"
5. **Remove non-responders**: Suppress anyone who didn't click to stay

**Subject lines that work:**
- "Are you still interested in [Product]? Let us know."
- "We haven't heard from you — should we stop emailing?"
- "One more from us, then we'll leave you alone"

---

## Address Collection Best Practices

The quality of your list starts at the point of collection.

### Double Opt-in for Marketing

Double opt-in sends a confirmation email before adding to the marketing list. It reduces invalid addresses, creates explicit consent records for GDPR/CASL, and produces more engaged lists (confirmed intent).

```javascript
// Step 1: User submits form
async function handleSignupForm(email) {
  const token = crypto.randomUUID();
  await db.pendingSubscriptions.insert({
    email,
    token,
    expiresAt: Date.now() + 24 * 60 * 60 * 1000 // 24 hours
  });

  await client.sendEmailWithTemplate({
    From: 'hello@yourdomain.com',
    To: email,
    TemplateAlias: 'confirm-subscription',
    TemplateModel: {
      confirm_url: `https://yourdomain.com/confirm?token=${token}`
    },
    MessageStream: 'outbound' // confirmation is transactional
  });
}

// Step 2: User clicks confirmation link
async function handleConfirmation(token) {
  const pending = await db.pendingSubscriptions.findOne({ token });
  if (!pending || pending.expiresAt < Date.now()) return false;

  await db.subscribers.insert({
    email: pending.email,
    status: 'subscribed',
    consentTimestamp: new Date(),
    consentSource: 'double-opt-in'
  });
  await db.pendingSubscriptions.delete({ token });
  return true;
}
```

### What to Collect at Signup

| Collect | Don't Collect |
|---------|---------------|
| Email address | More fields than necessary (GDPR data minimization) |
| Consent timestamp | Pre-filled or pre-checked marketing consent checkbox |
| Consent source (form URL, context) | Email address without a stated purpose |
| Preference (email types they want) | |

---

## Handling List Imports

When importing an existing list, validate it first:

1. **Remove obvious invalids**: No `@` symbol, no TLD, disposable domains
2. **Check age**: Lists older than 12 months have significant decay
3. **Verify you have legal basis**: Explicit consent or recent relationship
4. **Never import purchased lists**: Always results in high bounce/complaint rates

For large imports of older lists, warm up the list in segments:
- Start with most recently active (last 90 days)
- Add 6-month and 12-month cohorts after confirming good initial metrics
- Never send to the full list on day 1
