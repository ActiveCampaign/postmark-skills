---
name: postmark-email-best-practices
description: Use when asking about email deliverability, compliance (CAN-SPAM, GDPR, CASL), transactional email design patterns, list management, testing safely, or general email best practices — provider-agnostic knowledge with Postmark-specific guidance.
license: MIT
metadata:
  author: postmark
  version: "1.0.0"
---

# Email Best Practices

Postmark has delivered billions of transactional emails over 15+ years. This skill distills that expertise into actionable guidelines for building reliable, compliant, high-deliverability email systems.

## Quick Reference

| Topic | Use When |
|-------|----------|
| **Deliverability** | Setting up SPF/DKIM/DMARC, warming a new domain, diagnosing delivery issues |
| **Compliance** | Building unsubscribe flows, handling GDPR/CAN-SPAM/CASL requirements |
| **Transactional Design** | Designing welcome emails, password resets, receipts, alerts |
| **List Management** | Handling bounces, suppressions, list hygiene |
| **Testing** | Testing safely without hurting sender reputation |
| **Sending Reliability** | Idempotency, retry logic, rate limits |

## Deliverability Fundamentals

The three authentication records every sending domain must have:

| Record | Purpose | Priority |
|--------|---------|----------|
| **SPF** | Authorizes servers to send as your domain | Required |
| **DKIM** | Cryptographically signs emails to prove authenticity | Required |
| **DMARC** | Policy for handling SPF/DKIM failures | Required |

With Postmark, DKIM is configured automatically when you verify a sender domain. SPF and DMARC must be set up in your DNS.

See [references/deliverability.md](references/deliverability.md) for DNS setup, reputation factors, and domain warm-up guidance.

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

See [references/sending-reliability.md](references/sending-reliability.md) for idempotency patterns, retry strategies, and rate limit handling.

## Notes

- Postmark is purpose-built for transactional email — use it for triggered 1:1 emails, not bulk marketing
- Deliverability is not just about authentication — it's about sending wanted email to engaged recipients
- A single spam complaint from a real user is more damaging than 1,000 hard bounces
- Monitor your bounce rate (keep below 2%) and spam complaint rate (keep below 0.04%)
