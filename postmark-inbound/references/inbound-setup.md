# Inbound Email Setup

## Option 1: MX Record (Recommended)

Point your domain's MX records to Postmark to route all email for the domain through Postmark:

```
MX  inbound.postmarkapp.com  priority 10
```

### DNS Configuration Steps

1. Log in to your DNS provider (Route 53, Cloudflare, Namecheap, etc.)
2. Remove or lower the priority of any existing MX records
3. Add: `MX inbound.postmarkapp.com 10`
4. Allow up to 48 hours for DNS propagation

## Option 2: Email Forwarding

Forward a specific address to your Postmark inbound address. Your server's inbound address is shown in the Postmark dashboard under **Server → Inbound**.

The forwarding address looks like: `[hash]@inbound.postmarkapp.com`

### When to Use Forwarding

- You already have an email provider and don't want to change MX records
- You only want to process a single address (e.g., `support@yourdomain.com`)
- You want a quick setup without DNS changes

### Caveats with Forwarding

- Some email providers modify headers during forwarding, which can affect sender identification
- DMARC policies on the sender's domain may cause issues with forwarded email
- `StrippedTextReply` may not work as reliably with forwarded email

## Constraints

| Constraint | Detail |
|-----------|--------|
| **Inbound Streams** | One per server |
| **Domain** | One per Inbound Stream |
| **Webhook URL** | One per Inbound Message Stream |
| **Outbound compatibility** | Same server can handle both inbound and outbound |

If you need to process email for multiple domains, create separate Postmark servers — one per domain.

## Setting the Webhook URL

In your Postmark server dashboard:

1. Go to **Server → Settings → Inbound**
2. Set the **Inbound Webhook URL** to your endpoint (must be HTTPS)
3. Postmark will POST parsed email data to this URL for every incoming email

## Webhook Retries

If your endpoint returns a non-200 status code, Postmark retries delivery:

| Retry | Interval After Previous Attempt |
|-------|-------------------------------|
| 1 | 1 minute |
| 2 | 5 minutes |
| 3 | 10 minutes |
| 4 | 10 minutes |
| 5 | 10 minutes |
| 6 | 15 minutes |
| 7 | 30 minutes |
| 8 | 1 hour |
| 9 | 2 hours |
| 10 | 6 hours |

A **403 response** immediately stops all retries — Postmark interprets this as intentional rejection.

After all retries are exhausted, the message is marked as "Failed" in your activity page. Manually retry via the API:

```bash
curl "https://api.postmarkapp.com/messages/inbound/{messageid}/retry" \
  -X PUT \
  -H "Accept: application/json" \
  -H "X-Postmark-Server-Token: $POSTMARK_SERVER_TOKEN"
```

## Testing Inbound

Use the Postmark dashboard to send a test inbound message:

1. Go to **Server → Inbound → Send Test**
2. Enter a sample email payload
3. Postmark will POST to your configured webhook URL

Alternatively, send an actual email to your inbound address and watch your webhook endpoint receive it.
