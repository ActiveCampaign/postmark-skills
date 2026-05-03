---
name: postmark-email-best-practices
description: "Use when setting up SPF/DKIM/DMARC authentication, diagnosing deliverability issues, implementing CAN-SPAM/GDPR/CASL-compliant unsubscribe flows, designing transactional email patterns (welcome, password reset, receipts), managing bounce suppressions and list hygiene, or testing email safely without hurting sender reputation — provider-agnostic knowledge with Postmark-specific guidance."
license: MIT
metadata:
  author: postmark
  version: "1.0.0"
---

# Email Best Practices

Actionable guidelines for building reliable, compliant, high-deliverability email systems with Postmark.

## Quick Reference

| Topic | Use When |
|-------|----------|
| **Deliverability** | Setting up SPF/DKIM/DMARC, warming a new domain, diagnosing delivery issues |
| **Compliance** | Building unsubscribe flows, handling GDPR/CAN-SPAM/CASL requirements |
| **Transactional Design** | Designing welcome emails, password resets, receipts, alerts |
| **List Management** | Handling bounces, suppressions, list hygiene |
| **Testing** | Testing safely without hurting sender reputation |
| **Sending Reliability** | Idempotency, retry logic, rate limits |

## Domain Authentication Setup

Every sending domain must have SPF, DKIM, and DMARC configured. Missing records are the most common cause of email landing in spam.

### Step-by-step

1. **Add SPF** — authorize Postmark to send as your domain:
   ```
   v=spf1 include:spf.mtasv.net ~all
   ```
   If you already have an SPF record, merge — only one SPF TXT record per domain.

2. **Add DKIM** — verify your sending domain in Postmark, then add the provided CNAME:
   ```
   pm._domainkey.yourdomain.com  CNAME  pm.mtasv.net
   ```

3. **Add DMARC** — start with monitoring, then escalate:
   ```
   _dmarc.yourdomain.com  TXT  "v=DMARC1; p=none; rua=mailto:dmarc@yourdomain.com"
   ```

4. **Verify** — confirm all three pass using [MXToolbox](https://mxtoolbox.com/SuperTool.aspx) or [Postmark's DMARC Digests](https://dmarc.postmarkapp.com). Only proceed to production sends after SPF, DKIM, and DMARC all pass.

See [references/deliverability.md](references/deliverability.md) for DMARC escalation (none → quarantine → reject), reputation factors, and domain warm-up schedules.

## Transactional vs. Broadcast Email

**Never mix transactional and broadcast email in the same sending stream.** They have different delivery characteristics, compliance requirements, and reputation profiles.

| Type | Examples | Compliance | Unsubscribe Required |
|------|----------|------------|---------------------|
| **Transactional** | Password resets, receipts, alerts, notifications | CAN-SPAM exemption possible | No (but good practice) |
| **Broadcast** | Newsletters, promotions, announcements | CAN-SPAM, GDPR, CASL apply | Yes — legally required |

Postmark enforces this separation with **Message Streams** — use `outbound` for transactional, `broadcast` for marketing.

See [references/compliance.md](references/compliance.md) for CAN-SPAM, GDPR, and CASL requirements.

## Transactional Email Design

Good transactional emails are:
- **Expected** — The recipient triggered this email
- **Timely** — Sent immediately after the triggering event
- **Actionable** — One clear call to action
- **Plain** — Minimal design; content over decoration

Common transactional email types and their essential elements:

| Email Type | Must Include | Avoid |
|-----------|--------------|-------|
| Welcome | Product name, next step CTA, support contact | Marketing upsell on day 1 |
| Password reset | Expiry time, ignore-if-not-you notice, support link | Long copy |
| Receipt / Invoice | Line items, total, billing address, support | Promotional content |
| Shipping notification | Tracking link, estimated delivery, items | Unrelated promotions |
| Security alert | What happened, when, action required, how to secure | Panic-inducing language |

See [references/transactional-design.md](references/transactional-design.md) for design patterns, copy guidelines, and HTML email best practices.

## List Health

Sending to invalid, inactive, or unengaged addresses is the leading cause of deliverability problems.

**Key rules:**
- Remove **hard bounces** immediately and permanently
- Suppress **spam complaints** immediately — never re-add
- Re-permission lists older than 12–18 months before mailing
- Never purchase or rent email lists
- Validate addresses at the point of collection

See [references/list-management.md](references/list-management.md) for suppression strategies, list hygiene schedules, and re-engagement workflows.

## Testing Safely

**Never test with real addresses at consumer providers** (gmail.com, yahoo.com, etc.) — it damages sender reputation.

| Method | How | Use For |
|--------|-----|---------|
| API test token | Use `POSTMARK_API_TEST` as your server token | Validating API calls in CI/development |
| Black hole | Send to `test@blackhole.postmarkapp.com` | Functional testing — appears in activity |
| Sandbox server | Create a dedicated sandbox server in dashboard | Full send pipeline without delivery |
| Bounce testing | `hardbounce@bounce-testing.postmarkapp.com` | Testing bounce webhook handlers |

See [references/testing.md](references/testing.md) for full testing setup and domain warm-up schedules.

## Sending Reliability

Production email systems need idempotency keys, retry logic, and rate limit handling to avoid duplicate sends and silent failures.

```javascript
const crypto = require('crypto');

async function sendEmailIdempotent({ to, templateAlias, templateModel, eventType, eventId }) {
  const key = crypto.createHash('sha256').update(`${eventType}:${eventId}:${to}`).digest('hex');
  if (await db.emailLog.findOne({ idempotencyKey: key })) return; // already sent

  const result = await client.sendEmailWithTemplate({
    From: 'no-reply@yourdomain.com',
    To: to,
    TemplateAlias: templateAlias,
    TemplateModel: templateModel,
    MessageStream: 'outbound'
  });

  await db.emailLog.insert({ idempotencyKey: key, messageId: result.MessageID, sentAt: new Date() });
  return result;
}
```

See [references/sending-reliability.md](references/sending-reliability.md) for retry strategies with exponential backoff, rate limit handling, and queue patterns.

## Notes

- A single spam complaint is more damaging than 1,000 hard bounces — suppress complainers immediately
- Monitor bounce rate (keep below 2%) and spam complaint rate (keep below 0.04%) in the Postmark dashboard
