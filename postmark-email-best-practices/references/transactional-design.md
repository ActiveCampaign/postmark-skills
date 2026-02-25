# Transactional Email Design

Transactional emails are among the most-read emails in any inbox. They're expected, triggered by user action, and opened at rates that marketing email rarely achieves. Design them accordingly.

## Core Principles

| Principle | What It Means |
|-----------|--------------|
| **One purpose** | Each email does one thing and has one primary CTA |
| **Send immediately** | Trigger within seconds of the event — delays erode trust |
| **Content over design** | The information is why they're reading, not aesthetics |
| **Plain over decorated** | Minimal design renders better across all email clients |
| **Mobile first** | Over 50% of transactional email is opened on mobile |

---

## Common Email Types

### Welcome Email

Sent immediately after signup. Sets expectations for the product.

**Must include:**
- Confirmation that signup succeeded
- Product name and value proposition (1–2 sentences max)
- Single "Get Started" CTA pointing to onboarding
- Support contact (reply address or link)

**Avoid:** Marketing upsell, multiple CTAs, long feature lists on day 1.

Subject line pattern: `Welcome to [Product]` or `Your [Product] account is ready`

---

### Password Reset

**Must include:**
- Clear "Reset your password" button
- Expiry time (15–60 minutes is standard)
- "If you didn't request this, you can safely ignore this email" notice
- Support link for account security concerns

**Must not:**
- Include or reference the current password
- Allow the link to be used more than once
- Expire after more than 24 hours

Subject line: `Reset your password` — direct, no product name needed

---

### Email / Account Verification

**Must include:**
- Confirmation link or numeric code
- How long the link or code is valid
- Instructions if they didn't initiate this

Subject line: `Confirm your email address` or `Verify your [Product] account`

---

### Order Confirmation / Receipt

**Must include:**
- Order or transaction ID
- Line items with quantities and prices
- Subtotal, tax, shipping, and total
- Billing and shipping addresses
- Estimated delivery date (if applicable)
- Link to order status page
- Customer support contact

**Avoid:** Promotional upsell in the primary content area.

Subject line: `Your order #12345 is confirmed` or `Receipt for $49.99`

---

### Shipping Notification

**Must include:**
- Tracking link (make it prominent — this is why they opened the email)
- Carrier name
- Estimated delivery date
- Items being shipped
- Link to full order

Subject line: `Your order is on its way` — put tracking number in preheader text

---

### Security Alert

Sent for new device login, password change, unusual activity, etc.

**Must include:**
- Exactly what happened (device type, location, approximate time)
- A reassurance: "If this was you, no action is needed"
- A clear action if it wasn't them (link to secure account / change password)

**Must not:**
- Include marketing content of any kind
- Use alarmist language unnecessarily
- Send for every routine login

Subject line: `New sign-in to your [Product] account` or `Your password was changed`

---

## HTML Email Best Practices

### Structure

Use table-based layouts — CSS grid and flexbox have poor support in email clients:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Email Subject</title>
</head>
<body style="margin: 0; padding: 0; background-color: #f4f4f4;">
  <table role="presentation" width="100%" cellpadding="0" cellspacing="0">
    <tr>
      <td align="center" style="padding: 20px 0;">
        <table role="presentation" width="600" cellpadding="0" cellspacing="0"
               style="background: #ffffff; border-radius: 4px;">
          <!-- content here -->
        </table>
      </td>
    </tr>
  </table>
</body>
</html>
```

### CSS Rules for Email

| Rule | Why |
|------|-----|
| Inline all styles | Many clients strip `<style>` tags |
| Avoid `position`, `float`, `flexbox`, `grid` | Poor or no support in email clients |
| Use `role="presentation"` on layout tables | Improves accessibility for screen readers |
| Max width 600px | Renders well on desktop and mobile |
| Body font size ≥ 14px, mobile ≥ 16px | Readable without zooming |
| Line height 1.5–1.6 | Improves readability |
| Use web-safe fonts with fallbacks | Custom fonts not universally supported |

### CTA Buttons

Avoid image-based buttons — they disappear when images are blocked. Use HTML/CSS:

```html
<a href="{{action_url}}"
   style="display: inline-block; background-color: #0a2540; color: #ffffff;
          padding: 12px 24px; border-radius: 4px; text-decoration: none;
          font-size: 16px; font-weight: bold; font-family: sans-serif;">
  Get Started
</a>
```

### Always Include Plain Text

Every HTML email needs a plain text alternative:
- Used by screen readers and accessibility tools
- Preferred by some corporate email servers
- Improves deliverability (HTML-only emails trigger some spam filters)

```javascript
await client.sendEmail({
  From: 'noreply@yourdomain.com',
  To: 'customer@example.com',
  Subject: 'Your order is confirmed',
  HtmlBody: '<!-- full HTML -->',
  TextBody: 'Your order ORD-12345 is confirmed.\n\nView your order: https://...\n\nNeed help? support@yourdomain.com',
  MessageStream: 'outbound'
});
```

---

## Subject Lines and Preheader

### Subject Line Guidelines

| Do | Avoid |
|----|-------|
| Be specific: `Your order #12345 is confirmed` | Vague: `Update from Your Company` |
| Keep under 50 characters for mobile | Over 70 characters (truncated on most clients) |
| Match the email body | Misleading or clickbait phrasing |

### Preheader Text

The preheader is the short preview text shown after the subject line in the inbox. It acts as a second subject line. Set it explicitly — otherwise email clients pull the first visible text, which may be "View in browser" or a navigation link.

```html
<!-- Add immediately after <body> tag -->
<div style="display: none; max-height: 0; overflow: hidden; mso-hide: all;">
  Track your shipment: estimated delivery February 15 · View order status
</div>
```

---

## Sender Name and From Address

| Best Practice | Why |
|--------------|-----|
| Use a recognizable sender name | `Acme Support` not `noreply@acme.com` |
| Match sender to email type | `security@` for alerts, `orders@` for receipts |
| Avoid `noreply@` for customer-facing email | Signals you don't want a reply; use `support@` instead |
| Keep sender name consistent | Changing it confuses recipients and hurts open rates |
| Use a verified domain | Required by Postmark; also improves deliverability |
