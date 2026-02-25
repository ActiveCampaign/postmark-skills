# Testing Email Safely

Testing email against real consumer addresses (gmail.com, yahoo.com, outlook.com) trains inbox providers to associate your domain with test traffic, which looks like spam behavior. Use Postmark's dedicated testing tools instead.

## Postmark Testing Options

### API Test Token

Postmark provides a special server token, `POSTMARK_API_TEST`, that accepts the same API calls as a real token but never delivers email. Responses indicate success, and no email is sent.

Use this for unit tests and CI/CD pipelines where you only need to verify the API call is correctly formed:

```javascript
const postmark = require('postmark');
const client = new postmark.ServerClient('POSTMARK_API_TEST');

// This call succeeds but sends nothing
const result = await client.sendEmail({
  From: 'sender@yourdomain.com',
  To: 'recipient@example.com',
  Subject: 'Test',
  TextBody: 'Test body'
});
// result.Message === 'Test job accepted'
```

**Limitations:**
- No email sent, no activity log entry
- Template rendering is not validated
- Bounce/open/click webhooks are not triggered

---

### Black Hole Address

Send to `test@blackhole.postmarkapp.com` — this address accepts email and records it in your Postmark activity log, but does not forward to a real inbox. Useful for functional testing where you want to see the email in your Postmark activity feed.

```javascript
await client.sendEmail({
  From: 'sender@yourdomain.com',
  To: 'test@blackhole.postmarkapp.com',
  Subject: 'Functional test',
  HtmlBody: '<p>Test email</p>',
  MessageStream: 'outbound'
});
```

Appears in your Postmark activity log with a "delivered" status.

---

### Bounce Testing Addresses

Postmark provides special addresses for triggering specific bounce types, allowing you to test your bounce webhook handlers without using real addresses.

| Address | Bounce Type Triggered |
|---------|----------------------|
| `hardbounce@bounce-testing.postmarkapp.com` | Hard bounce |
| `softbounce@bounce-testing.postmarkapp.com` | Soft bounce |
| `subscribe@bounce-testing.postmarkapp.com` | Subscription change (opt-in) |
| `unsubscribe@bounce-testing.postmarkapp.com` | Subscription change (opt-out) |

```javascript
// Test bounce handler
await client.sendEmail({
  From: 'sender@yourdomain.com',
  To: 'hardbounce@bounce-testing.postmarkapp.com',
  Subject: 'Bounce test',
  TextBody: 'Testing bounce handler',
  MessageStream: 'outbound'
});
// Your bounce webhook fires with a HardBounce event shortly after
```

---

### Sandbox Server

Create a dedicated server in your Postmark account for testing. The sandbox server accepts real API calls and processes the full send pipeline (templates, layouts, attachments, headers) but does not deliver email.

**Use case:** End-to-end testing of your sending logic, template rendering, and email content without reputation risk.

Setup in Postmark dashboard:
1. Account → Servers → Create Server
2. Name it "Sandbox" or "Testing"
3. Use this server's API token in your test environment

---

## CI/CD Pipeline Setup

Use environment variables to switch between test and production tokens:

```javascript
// email.js
const postmark = require('postmark');

const POSTMARK_TOKEN = process.env.NODE_ENV === 'test'
  ? 'POSTMARK_API_TEST'
  : process.env.POSTMARK_SERVER_TOKEN;

const client = new postmark.ServerClient(POSTMARK_TOKEN);
```

```yaml
# .github/workflows/test.yml
env:
  POSTMARK_SERVER_TOKEN: POSTMARK_API_TEST
  NODE_ENV: test
```

---

## Inbox Rendering Testing

Test how your HTML email renders across email clients before sending. Use these tools (external services):

| Tool | What It Tests |
|------|--------------|
| [Litmus](https://litmus.com) | Rendering across 90+ email clients and devices |
| [Email on Acid](https://www.emailonacid.com) | Rendering plus spam filter testing |
| [Mailtrap](https://mailtrap.io) | SMTP-based inbox for development teams |

---

## Domain Warm-up Testing

When warming up a new domain, monitor these metrics daily:

| Metric | Check | Action if Off |
|--------|-------|---------------|
| Bounce rate | Below 2% | Pause; review list quality |
| Spam complaint rate | Below 0.04% | Pause; review content and list |
| Delivery rate | Above 95% | Investigate SMTP response codes for blocks |
| Inbox placement | Most to inbox | Use inbox testing service to diagnose |

### Warm-up Schedule Reference

| Day | Max per Day | Max per Hour |
|-----|-------------|--------------|
| 1 | 150 | — |
| 2 | 250 | — |
| 3 | 400 | — |
| 4 | 700 | 50 |
| 5 | 1,000 | 75 |
| 6 | 1,500 | 100 |
| 7 | 2,000 | 150 |

After day 7: increase by no more than 2x per week until target volume is reached.

**During warm-up, always start with your best audience:**
1. Recent signups (last 30 days) — highest engagement
2. Actively purchasing customers
3. Users who have opened email in the last 90 days
4. Older actives (90–180 days)
5. Inactive users last

---

## Testing Checklist

Before any production send:

- [ ] Test API call with `POSTMARK_API_TEST` token passes
- [ ] Template renders correctly with actual data (use black hole or sandbox)
- [ ] Plain text version included and readable
- [ ] All links resolve to correct destinations
- [ ] Unsubscribe link works and suppresses the address
- [ ] HTML renders correctly in target email clients (use Litmus / Email on Acid)
- [ ] Subject line is under 50 characters
- [ ] Preheader text is set
- [ ] From address uses a verified Postmark sending domain
- [ ] Message Stream is correct (outbound vs. broadcast)
- [ ] Bounce and spam complaint webhooks are handling test events correctly
