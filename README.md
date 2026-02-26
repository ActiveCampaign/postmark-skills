```
 ____           _                        _       ____  _    _ _ _
|  _ \ ___  ___| |_ _ __ ___   __ _ _ __| | __  / ___|| | _(_) | |___
| |_) / _ \/ __| __| '_ ` _ \ / _` | '__| |/ /  \___ \| |/ / | | / __|
|  __/ (_) \__ \ |_| | | | | | (_| | |  |   <    ___) |   <| | | \__ \
|_|   \___/|___/\__|_| |_| |_|\__,_|_|  |_|\_\  |____/|_|\_\_|_|_|___/
```

# Postmark Skills

[Agent Skills](https://github.com/anthropics/skills) for the [Postmark](https://postmarkapp.com) email platform. Teach AI coding agents how to send transactional email, process inbound messages, manage templates, and configure webhooks using the Postmark API.

## Available Skills

| Skill | Description |
|-------|-------------|
| [`postmark-send-email`](./postmark-send-email/) | Send emails via Postmark API — single, batch, bulk, and template-based. Supports transactional and broadcast message streams. |
| [`postmark-inbound`](./postmark-inbound/) | Process incoming emails with Postmark inbound webhooks. Build reply-by-email, email-to-ticket, and document extraction workflows. |
| [`postmark-templates`](./postmark-templates/) | Create and manage server-side email templates with Handlebars syntax. Supports layouts, template validation, and cross-server pushing. |
| [`postmark-webhooks`](./postmark-webhooks/) | Configure webhooks for delivery, bounce, open, click, spam complaint, and subscription change events. |
| [`postmark-email-best-practices`](./postmark-email-best-practices/) | Deliverability, compliance (CAN-SPAM, GDPR, CASL), transactional email design, list management, testing safely, and sending reliability. |

## Installation

```bash
npx skills add ActiveCampaign/postmark-skills
```

Install a specific skill:

```bash
npx skills add ActiveCampaign/postmark-skills --skill postmark-send-email
```

## Usage

Once installed, your AI coding agent will automatically use these skills when relevant. Example prompts:

- "Send a welcome email using Postmark"
- "Set up inbound email processing for a support ticket system"
- "Create a Postmark template for order confirmation emails"
- "Add bounce and delivery webhooks to track email status"
- "Send a batch of 200 notification emails through Postmark"
- "How do I set up SPF, DKIM, and DMARC for Postmark?"
- "What are the CAN-SPAM and GDPR requirements for my email list?"
- "Help me design a password reset email"

## Postmark & Community SDKs

- **Node.js / TypeScript** — [`postmark`](https://www.npmjs.com/package/postmark)
- **Python** — [`postmarker`](https://pypi.org/project/postmarker/)*
- **Ruby** — [`postmark`](https://rubygems.org/gems/postmark)
- **PHP** — [`wildbit/postmark-php`](https://packagist.org/packages/wildbit/postmark-php)
- **.NET** — [`Postmark`](https://www.nuget.org/packages/Postmark)

<small>* Disclaimer: Third-party SDKs are provided "as is" without warranty of any kind. Postmark disclaims all liability for any damages or issues arising from their use.</small>

## Prerequisites

1. A [Postmark account](https://account.postmarkapp.com/sign_up)
2. A verified sender domain or sender signature
3. Your Server API Token (from [Postmark Servers](https://account.postmarkapp.com/servers))
4. Set your token as an environment variable: `POSTMARK_SERVER_TOKEN`

## Why Postmark?

- **Message Streams** — Separate transactional and broadcast email for better deliverability
- **500 emails per batch** — 5x more than alternatives
- **Batch attachments** — Attach files in batch sends
- **Server-side templates** — Handlebars templates with layout inheritance, no client-side rendering needed
- **Full inbound processing** — Parse and route incoming emails with webhook delivery
- **15+ years deliverability expertise** — Trusted by 100,000+ developers

## License

MIT
