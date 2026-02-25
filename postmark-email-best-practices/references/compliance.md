# Email Compliance

Email compliance is not optional. Violations of CAN-SPAM, GDPR, or CASL can result in significant fines and lasting damage to sender reputation.

## Transactional vs. Marketing: Why It Matters for Compliance

| Type | Examples | Compliance Burden |
|------|----------|-------------------|
| **Transactional** | Password resets, order confirmations, alerts | Lower — CAN-SPAM exemption possible |
| **Marketing / Broadcast** | Newsletters, promotions, announcements | Full — CAN-SPAM, GDPR, CASL all apply |

**Never mix types in the same Postmark Message Stream.** Use `outbound` for transactional, `broadcast` for marketing. Mixing damages deliverability and creates compliance risk.

---

## CAN-SPAM (United States)

Applies to commercial email sent to US recipients.

### Requirements

| Requirement | Detail |
|-------------|--------|
| No deceptive headers | From, To, Reply-To must accurately identify the sender |
| No deceptive subject lines | Subject must reflect the email's actual content |
| Physical postal address | Include a valid mailing address (P.O. Box acceptable) |
| Unsubscribe mechanism | Clear, working opt-out method in every commercial email |
| Honor opt-outs promptly | Process unsubscribes within 10 business days |
| No opt-out fees | Unsubscribing must be free and require only one action |

### Transactional Exemption

CAN-SPAM exempts **transactional or relationship messages** — order confirmations, password resets, account notifications — from most requirements, including the unsubscribe mandate.

To qualify, the email must be **primarily transactional**. Adding promotional content to a transactional email can void the exemption. When in doubt, include a postal address and an optional unsubscribe link anyway.

---

## GDPR (European Union)

Applies to processing personal data (including email addresses) of individuals in the EU/EEA, regardless of where your company is based.

### Key Principles

| Principle | Email Application |
|-----------|------------------|
| **Lawful basis** | You need a legal basis to email someone (consent, contract, legitimate interest) |
| **Consent for marketing** | Marketing requires explicit, freely-given, informed consent |
| **Purpose limitation** | Can't use an address collected for support to send marketing |
| **Right to erasure** | Must delete all personal data upon request |
| **Data minimization** | Collect only email data you actually need |
| **Transparency** | Inform users how their email address will be used at collection time |

### Valid GDPR Consent for Marketing

Consent must be:
- **Freely given** — not bundled with terms of service or required for account creation
- **Specific** — for the exact type of emails you'll send
- **Informed** — users know what they're agreeing to
- **Unambiguous** — explicit opt-in action (checked checkbox), never pre-ticked

**Record everything:** Timestamp, source, and wording of consent for every subscriber.

### Transactional Email Under GDPR

Sending transactional email (order confirmation, password reset) is generally justified under **performance of a contract** — explicit consent is not required. However:
- Only send emails necessary to deliver the service the user signed up for
- Do not include promotional content that would require consent

### Right to Erasure

When a user requests deletion, remove their email address (and all associated data) from your system — including from any Postmark suppressions you're maintaining.

---

## CASL (Canada)

Canada's Anti-Spam Legislation is stricter than CAN-SPAM. It applies to Commercial Electronic Messages (CEMs) sent to Canadian recipients.

### Consent Types

| Type | When It Applies | Expires |
|------|-----------------|---------|
| **Express consent** | User explicitly opted in (checked a box, confirmed via email) | Never (until withdrawn) |
| **Implied — existing business relationship** | Purchase, donation, or service inquiry within 2 years | 2 years from last transaction |
| **Implied — conspicuous publication** | Email address published publicly without opt-out | Ongoing, for role-related messages only |

### Every CEM Must Include

1. Identification of the sender (name, and whose behalf if different)
2. Mailing address plus phone, email, or web address
3. A clear, easy unsubscribe mechanism

### Key Differences from CAN-SPAM

| Aspect | CAN-SPAM | CASL |
|--------|----------|------|
| Opt-in required? | No (opt-out model) | Yes (opt-in model) |
| Implied consent | Broad | Narrow — 2-year expiry |
| Unsubscribe deadline | 10 business days | 10 business days |
| Max penalty | $16,000 per violation | $10M CAD per violation |

---

## Unsubscribe Implementation

### What Every Marketing Email Must Include

```html
<!-- In every broadcast email footer -->
<p style="font-size: 12px; color: #666; text-align: center;">
  You're receiving this because you subscribed to updates from Your Company.<br>
  <a href="{{unsubscribe_url}}">Unsubscribe</a> &middot;
  <a href="{{preferences_url}}">Email preferences</a> &middot;
  123 Main St, San Francisco, CA 94107
</p>
```

### One-Click Unsubscribe (List-Unsubscribe Header)

Gmail and Yahoo require bulk senders to support RFC 8058 one-click unsubscribe. Add these headers to broadcast sends:

```javascript
await client.sendEmail({
  // ...
  Headers: [
    {
      Name: 'List-Unsubscribe',
      Value: '<mailto:unsub@yourdomain.com?subject=unsubscribe>, <https://yourdomain.com/unsubscribe?token={{token}}>'
    },
    {
      Name: 'List-Unsubscribe-Post',
      Value: 'List-Unsubscribe=One-Click'
    }
  ],
  MessageStream: 'broadcast'
});
```

### Processing Unsubscribes

When a user unsubscribes (via your link or Postmark's `SubscriptionChange` webhook):

1. **Suppress immediately** — do not batch-process
2. **Honor across all lists** — one unsubscribe removes from all marketing email
3. **Record it** — timestamp and source for compliance documentation
4. **Never re-add** without fresh explicit consent
5. **Confirm** — a single confirmation email is acceptable

```javascript
// Handle Postmark's SubscriptionChange webhook
function handleSubscriptionChange(event) {
  if (event.SuppressSending) {
    db.suppressions.upsert({
      email: event.Recipient,
      suppressedAt: event.ChangedAt,
      reason: event.SuppressionReason,
      stream: event.MessageStream
    });
  }
}
```

---

## Compliance Checklist

- [ ] SPF, DKIM, and DMARC configured for every sending domain
- [ ] Transactional and marketing in separate Postmark Message Streams
- [ ] Physical postal address in every marketing email
- [ ] Working unsubscribe link in every marketing email
- [ ] List-Unsubscribe headers on broadcast sends
- [ ] Consent recorded with timestamp and wording for all marketing subscribers
- [ ] Unsubscribes honored within 10 business days (aim for immediate)
- [ ] GDPR data deletion process documented and operational
- [ ] Suppression list maintained and synced with Postmark
