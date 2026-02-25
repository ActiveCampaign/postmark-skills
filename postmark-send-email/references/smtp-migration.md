# SMTP Migration Guide

How to migrate from SMTP to the Postmark API, or use Postmark's SMTP interface.

## SMTP Settings

| Setting | Transactional | Broadcast |
|---------|--------------|-----------|
| **Host** | `smtp.postmarkapp.com` | `smtp-broadcasts.postmarkapp.com` |
| **Ports** | 25, 2525, or 587 | 25, 2525, or 587 |
| **TLS** | Required (STARTTLS) | Required (STARTTLS) |
| **Username** | Your Server API Token | Your Server API Token |
| **Password** | Your Server API Token | Your Server API Token |

## SMTP Authentication

Postmark supports two SMTP authentication methods:

### 1. API Token (Recommended)

Use your Server API Token as both username and password:

```
Username: your-server-api-token
Password: your-server-api-token
```

### 2. SMTP Tokens

Unique tokens per message stream, generated in the Postmark dashboard under Server → SMTP.

## Custom Headers via SMTP

Control Postmark features by adding custom headers to your SMTP messages:

| Header | Values | Description |
|--------|--------|-------------|
| `X-PM-Message-Stream` | `outbound`, `broadcast` | Message stream selection |
| `X-PM-Tag` | Any string (max 1000 chars) | Tag for filtering and stats |
| `X-PM-Track-Opens` | `true`, `false` | Enable/disable open tracking |
| `X-PM-Track-Links` | `None`, `HtmlAndText`, `HtmlOnly`, `TextOnly` | Link click tracking |
| `X-PM-Metadata-KEY` | Any string | Custom metadata (replace KEY with your key name) |

### Example Headers

```
X-PM-Message-Stream: outbound
X-PM-Tag: password-reset
X-PM-Track-Opens: true
X-PM-Track-Links: HtmlAndText
X-PM-Metadata-customer-id: 12345
X-PM-Metadata-order-id: ORD-67890
```

## Migration Examples

### Node.js (Nodemailer → Postmark API)

**Before (Nodemailer/SMTP):**

```javascript
const nodemailer = require('nodemailer');

const transporter = nodemailer.createTransport({
  host: 'smtp.old-provider.com',
  port: 587,
  auth: { user: 'username', pass: 'password' }
});

await transporter.sendMail({
  from: 'sender@yourdomain.com',
  to: 'recipient@example.com',
  subject: 'Hello',
  text: 'Hello World',
  html: '<p>Hello World</p>'
});
```

**After (Postmark API):**

```javascript
const postmark = require('postmark');
const client = new postmark.ServerClient(process.env.POSTMARK_SERVER_TOKEN);

await client.sendEmail({
  From: 'sender@yourdomain.com',
  To: 'recipient@example.com',
  Subject: 'Hello',
  TextBody: 'Hello World',
  HtmlBody: '<p>Hello World</p>',
  MessageStream: 'outbound'
});
```

### Node.js (Nodemailer → Postmark SMTP)

If you prefer to keep using SMTP (e.g., existing infrastructure):

```javascript
const nodemailer = require('nodemailer');

const transporter = nodemailer.createTransport({
  host: 'smtp.postmarkapp.com',
  port: 587,
  secure: false,
  auth: {
    user: process.env.POSTMARK_SERVER_TOKEN,
    pass: process.env.POSTMARK_SERVER_TOKEN
  }
});

await transporter.sendMail({
  from: 'sender@yourdomain.com',
  to: 'recipient@example.com',
  subject: 'Hello',
  text: 'Hello World',
  html: '<p>Hello World</p>',
  headers: {
    'X-PM-Message-Stream': 'outbound',
    'X-PM-Tag': 'migration-test'
  }
});
```

### Python (smtplib → Postmark API)

**Before (smtplib/SMTP):**

```python
import smtplib
from email.mime.text import MIMEText

msg = MIMEText('Hello World')
msg['Subject'] = 'Hello'
msg['From'] = 'sender@yourdomain.com'
msg['To'] = 'recipient@example.com'

with smtplib.SMTP('smtp.old-provider.com', 587) as server:
    server.starttls()
    server.login('username', 'password')
    server.send_message(msg)
```

**After (Postmark API):**

```python
from postmarker.core import PostmarkClient
import os

postmark = PostmarkClient(server_token=os.environ['POSTMARK_SERVER_TOKEN'])

postmark.emails.send(
    From='sender@yourdomain.com',
    To='recipient@example.com',
    Subject='Hello',
    TextBody='Hello World',
    HtmlBody='<p>Hello World</p>',
    MessageStream='outbound'
)
```

### Django (SMTP Backend → Postmark)

**Before (Django SMTP):**

```python
# settings.py
EMAIL_HOST = 'smtp.old-provider.com'
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_USER = 'username'
EMAIL_HOST_PASSWORD = 'password'
```

**After (Postmark SMTP via Django):**

```python
# settings.py
EMAIL_HOST = 'smtp.postmarkapp.com'
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_USER = os.environ['POSTMARK_SERVER_TOKEN']
EMAIL_HOST_PASSWORD = os.environ['POSTMARK_SERVER_TOKEN']
```

### Rails (ActionMailer → Postmark)

**Before (generic SMTP):**

```ruby
# config/environments/production.rb
config.action_mailer.smtp_settings = {
  address: 'smtp.old-provider.com',
  port: 587,
  user_name: 'username',
  password: 'password',
  authentication: :plain,
  enable_starttls_auto: true
}
```

**After (Postmark SMTP via ActionMailer):**

```ruby
# config/environments/production.rb
config.action_mailer.smtp_settings = {
  address: 'smtp.postmarkapp.com',
  port: 587,
  user_name: ENV['POSTMARK_SERVER_TOKEN'],
  password: ENV['POSTMARK_SERVER_TOKEN'],
  authentication: :plain,
  enable_starttls_auto: true
}
```

Or use the Postmark Rails gem for API-based sending:

```ruby
# Gemfile
gem 'postmark-rails'

# config/application.rb
config.action_mailer.delivery_method = :postmark
config.action_mailer.postmark_settings = {
  api_token: ENV['POSTMARK_SERVER_TOKEN']
}
```

## API vs SMTP

| Feature | API | SMTP |
|---------|-----|------|
| **Performance** | Faster (direct HTTP) | Slower (SMTP handshake) |
| **Batch sending** | Up to 500 per call | One at a time |
| **Error handling** | Structured JSON errors | SMTP error codes |
| **Metadata** | Native support | Via custom headers |
| **Templates** | Native support | Not available |
| **Migration effort** | Requires code changes | Minimal (change host/credentials) |

**Recommendation:** Use the API for new integrations. Use SMTP when migrating existing infrastructure with minimal changes.

## Key Notes

- SMTP and API use the same Server API Token
- Always use TLS (STARTTLS on ports 25, 2525, or 587)
- Use `smtp.postmarkapp.com` for transactional, `smtp-broadcasts.postmarkapp.com` for broadcast
- Set `X-PM-Message-Stream` header via SMTP to control stream selection
- API provides better performance, structured errors, and batch support
- SMTP is ideal for legacy systems or frameworks with built-in SMTP support
