# Webhook Payload Examples

All webhook payloads are delivered as HTTP POST with `Content-Type: application/json`. Use `RecordType` to identify the event.

## Delivery

Fired when the recipient's mail server accepts the message.

```json
{
  "RecordType": "Delivery",
  "ServerID": 23,
  "MessageStream": "outbound",
  "MessageID": "883953f4-6105-42a2-a16a-77a8eac79483",
  "Recipient": "john@example.com",
  "Tag": "welcome-email",
  "DeliveredAt": "2025-04-05T16:33:54.9070259Z",
  "Details": "Test delivery webhook details",
  "Metadata": {
    "customer_id": "12345"
  }
}
```

`DeliveredAt` is when the recipient's server accepted the message — not when it was read.

---

## Bounce

Fired when an email is rejected by the recipient's server.

```json
{
  "RecordType": "Bounce",
  "ID": 42,
  "Type": "HardBounce",
  "TypeCode": 1,
  "Name": "Hard bounce",
  "ServerID": 23,
  "MessageStream": "outbound",
  "MessageID": "883953f4-6105-42a2-a16a-77a8eac79483",
  "Tag": "welcome-email",
  "Description": "The server was unable to deliver your message (ex: unknown user, mailbox not found).",
  "Details": "smtp;550 5.1.1 The email account that you tried to reach does not exist.",
  "Email": "john@example.com",
  "From": "sender@yourdomain.com",
  "BouncedAt": "2025-04-05T16:33:54.9070259Z",
  "DumpAvailable": true,
  "Inactive": true,
  "CanActivate": true,
  "Subject": "Welcome to our service",
  "Metadata": {
    "customer_id": "12345"
  }
}
```

**Key fields:**
- `Inactive: true` — Postmark has deactivated this address; future sends are blocked
- `CanActivate: true` — Address can be reactivated if the bounce was temporary
- `DumpAvailable: true` — Full SMTP conversation available via the Bounces API

### Bounce Types

| Type | TypeCode | Action |
|------|----------|--------|
| `HardBounce` | 1 | Permanent — remove address from all lists immediately |
| `SoftBounce` | 4096 | Temporary — Postmark retries automatically; monitor |
| `Transient` | 2 | Temporary — retry may succeed |
| `SpamNotification` | 512 | Marked as spam by recipient's server |
| `Blocked` | 16 | Blocked by recipient server |
| `DMARCPolicy` | 100000 | Rejected due to sender's DMARC policy |

---

## Spam Complaint

Fired when a recipient marks your email as spam via a feedback loop.

```json
{
  "RecordType": "SpamComplaint",
  "ID": 42,
  "Type": "SpamComplaint",
  "TypeCode": 512,
  "ServerID": 23,
  "MessageStream": "outbound",
  "MessageID": "883953f4-6105-42a2-a16a-77a8eac79483",
  "Tag": "welcome-email",
  "Email": "john@example.com",
  "From": "sender@yourdomain.com",
  "BouncedAt": "2025-04-05T16:33:54.9070259Z",
  "Subject": "Welcome to our service",
  "Metadata": {
    "customer_id": "12345"
  }
}
```

**Action:** Immediately and permanently suppress this recipient. Spam complaints severely damage sender reputation — do not retry or re-add to lists.

---

## Open

Fired when a recipient opens the email. Requires `TrackOpens: true` on the sent message.

```json
{
  "RecordType": "Open",
  "FirstOpen": true,
  "ServerID": 23,
  "MessageStream": "outbound",
  "MessageID": "883953f4-6105-42a2-a16a-77a8eac79483",
  "Recipient": "john@example.com",
  "Tag": "welcome-email",
  "ReceivedAt": "2025-04-05T16:33:54.9070259Z",
  "ReadSeconds": 5,
  "Platform": "WebMail",
  "Client": {
    "Name": "Gmail",
    "Company": "Google",
    "Family": "Gmail"
  },
  "OS": {
    "Name": "Windows 10",
    "Company": "Microsoft",
    "Family": "Windows"
  },
  "UserAgent": "Mozilla/5.0 ...",
  "Geo": {
    "CountryISOCode": "US",
    "Country": "United States",
    "RegionISOCode": "CA",
    "Region": "California",
    "City": "San Francisco",
    "Zip": "94107",
    "Coords": "37.7749,-122.4194",
    "IP": "203.0.113.1"
  },
  "Metadata": {
    "customer_id": "12345"
  }
}
```

**Key fields:**
- `FirstOpen: true` — First time this recipient opened this message
- `ReadSeconds` — Estimated read time in seconds
- `Platform` — `Desktop`, `Mobile`, `WebMail`, or `Unknown`

**Note:** Apple Mail Privacy Protection (MPP) inflates open rates. Use click tracking as a more reliable engagement signal.

---

## Click

Fired when a recipient clicks a tracked link. Requires `TrackLinks` to be set on the sent message.

```json
{
  "RecordType": "Click",
  "ClickLocation": "HTML",
  "ServerID": 23,
  "MessageStream": "outbound",
  "MessageID": "883953f4-6105-42a2-a16a-77a8eac79483",
  "Recipient": "john@example.com",
  "Tag": "welcome-email",
  "OriginalLink": "https://yourdomain.com/pricing",
  "ReceivedAt": "2025-04-05T16:33:54.9070259Z",
  "Platform": "Desktop",
  "Client": {
    "Name": "Chrome",
    "Company": "Google",
    "Family": "Chrome"
  },
  "OS": {
    "Name": "macOS 14",
    "Company": "Apple",
    "Family": "macOS"
  },
  "UserAgent": "Mozilla/5.0 ...",
  "Geo": {
    "CountryISOCode": "US",
    "Country": "United States",
    "RegionISOCode": "CA",
    "Region": "California",
    "City": "San Francisco",
    "Zip": "94107",
    "Coords": "37.7749,-122.4194",
    "IP": "203.0.113.1"
  },
  "Metadata": {
    "customer_id": "12345"
  }
}
```

**Key fields:**
- `OriginalLink` — The actual destination URL (before Postmark's tracking redirect)
- `ClickLocation` — `HTML` or `Text`

---

## Subscription Change

Fired when a recipient unsubscribes via the Postmark-managed unsubscribe mechanism (broadcast stream).

```json
{
  "RecordType": "SubscriptionChange",
  "ServerID": 23,
  "MessageStream": "broadcast",
  "MessageID": "883953f4-6105-42a2-a16a-77a8eac79483",
  "Recipient": "john@example.com",
  "Tag": "newsletter",
  "ChangedAt": "2025-04-05T16:33:54.9070259Z",
  "Origin": "Recipient",
  "SuppressSending": true,
  "SuppressionReason": "ManualSuppression",
  "Metadata": {
    "customer_id": "12345"
  }
}
```

**Key fields:**
- `SuppressSending: true` — Postmark has suppressed this address; future sends are blocked
- `Origin` — `Recipient` (self-unsubscribed) or `Customer` (suppressed via API/dashboard)
- `SuppressionReason` — `ManualSuppression`, `HardBounce`, `SpamComplaint`

**Action:** Sync the unsubscribe to your own system immediately. Postmark suppresses automatically, but your database should reflect it too.
