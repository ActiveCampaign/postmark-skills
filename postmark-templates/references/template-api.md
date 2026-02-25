# Template API Reference

## Create a Template

**Endpoint:** `POST /templates`

### Node.js

```javascript
const postmark = require('postmark');
const client = new postmark.ServerClient(process.env.POSTMARK_SERVER_TOKEN);

const template = await client.createTemplate({
  Name: 'Welcome Email',
  Alias: 'welcome-email',
  Subject: 'Welcome to {{product_name}}, {{name}}!',
  HtmlBody: `
    <h1>Welcome, {{name}}!</h1>
    <p>Thanks for joining {{product_name}}.</p>
    {{#if trial}}
      <p>Your trial ends on {{trial_end_date}}.</p>
    {{/if}}
    <a href="{{action_url}}">Get Started</a>
  `,
  TextBody: 'Welcome, {{name}}!\n\nThanks for joining {{product_name}}.\n\nGet started: {{action_url}}',
  TemplateType: 'Standard'
});

console.log('Template ID:', template.TemplateId);
```

### cURL

```bash
curl "https://api.postmarkapp.com/templates" \
  -X POST \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -H "X-Postmark-Server-Token: $POSTMARK_SERVER_TOKEN" \
  -d '{
    "Name": "Welcome Email",
    "Alias": "welcome-email",
    "Subject": "Welcome to {{product_name}}, {{name}}!",
    "HtmlBody": "<h1>Welcome, {{name}}!</h1><p>Thanks for joining {{product_name}}.</p>",
    "TextBody": "Welcome, {{name}}!\n\nThanks for joining {{product_name}}.",
    "TemplateType": "Standard"
  }'
```

### Response

```json
{
  "TemplateId": 12345,
  "Name": "Welcome Email",
  "Alias": "welcome-email",
  "Active": true,
  "TemplateType": "Standard",
  "LayoutTemplate": null
}
```

---

## List Templates

```javascript
const templates = await client.getTemplates({
  count: 100,
  offset: 0,
  templateType: 'Standard' // 'Standard', 'Layout', or 'All'
});

templates.Templates.forEach(t => {
  console.log(`${t.Name} (${t.Alias || t.TemplateId}) — ${t.Active ? 'Active' : 'Inactive'}`);
});
```

---

## Get a Template

```javascript
const template = await client.getTemplate('welcome-email');
// or by ID:
const template = await client.getTemplate(12345);
```

---

## Update a Template

```javascript
await client.editTemplate('welcome-email', {
  Subject: 'Welcome to {{product_name}}!',
  HtmlBody: '<h1>Updated content for {{name}}</h1>'
});

// By ID:
await client.editTemplate(12345, {
  Name: 'Welcome Email v2',
  Subject: 'Welcome aboard, {{name}}!'
});
```

Only fields you include are updated — omitted fields are unchanged.

---

## Delete a Template

```javascript
await client.deleteTemplate('welcome-email');
// or by ID:
await client.deleteTemplate(12345);
```

**Note:** You cannot delete a Layout template that has dependent Standard templates. Remove the `LayoutTemplate` association from all dependents first.

---

## Validate a Template

Test-render without sending. Use in CI/CD to verify templates before deploying.

**Endpoint:** `POST /templates/validate`

### Node.js

```javascript
const validation = await client.validateTemplate({
  Subject: 'Welcome {{name}}',
  HtmlBody: '<h1>Hello {{name}}</h1>{{#if premium}}<p>Premium member</p>{{/if}}',
  TextBody: 'Hello {{name}}',
  TestRenderModel: {
    name: 'Test User',
    premium: true
  }
});

if (validation.AllContentIsValid) {
  console.log('Rendered subject:', validation.Subject.RenderedContent);
  console.log('Rendered HTML:', validation.HtmlBody.RenderedContent);
} else {
  console.error('Errors:', validation.HtmlBody.ValidationErrors);
}
```

### cURL

```bash
curl "https://api.postmarkapp.com/templates/validate" \
  -X POST \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -H "X-Postmark-Server-Token: $POSTMARK_SERVER_TOKEN" \
  -d '{
    "Subject": "Welcome {{name}}",
    "HtmlBody": "<h1>Hello {{name}}</h1>",
    "TextBody": "Hello {{name}}",
    "TestRenderModel": { "name": "Test User" }
  }'
```

The response includes `SuggestedTemplateModel` — a list of all variables detected in the template. Useful for documentation.

---

## Push Templates Between Servers

Sync templates from staging to production. Requires an **Account Token**, not a Server Token.

**Endpoint:** `PUT /templates/push`

```bash
# Preview first (PerformChanges: false)
curl "https://api.postmarkapp.com/templates/push" \
  -X PUT \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -H "X-Postmark-Account-Token: $POSTMARK_ACCOUNT_TOKEN" \
  -d '{
    "SourceServerID": 12345,
    "DestinationServerID": 67890,
    "PerformChanges": false
  }'

# Apply (PerformChanges: true)
curl "https://api.postmarkapp.com/templates/push" \
  -X PUT \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -H "X-Postmark-Account-Token: $POSTMARK_ACCOUNT_TOKEN" \
  -d '{
    "SourceServerID": 12345,
    "DestinationServerID": 67890,
    "PerformChanges": true
  }'
```

Templates are matched by Alias — templates without an Alias cannot be pushed.

---

## Send with a Template

### Single Email (alias recommended)

```javascript
const result = await client.sendEmailWithTemplate({
  From: 'hello@yourdomain.com',
  To: 'customer@example.com',
  TemplateAlias: 'welcome-email',
  TemplateModel: {
    name: 'Jane Doe',
    product_name: 'Acme App',
    trial: true,
    trial_end_date: 'February 15, 2025',
    action_url: 'https://app.yourdomain.com/start'
  },
  MessageStream: 'outbound'
});
```

### Batch with Templates (up to 500)

```javascript
const results = await client.sendEmailBatchWithTemplates([
  {
    From: 'hello@yourdomain.com',
    To: 'user1@example.com',
    TemplateAlias: 'welcome-email',
    TemplateModel: { name: 'User 1', product_name: 'Acme App' },
    MessageStream: 'outbound'
  },
  {
    From: 'hello@yourdomain.com',
    To: 'user2@example.com',
    TemplateAlias: 'welcome-email',
    TemplateModel: { name: 'User 2', product_name: 'Acme App' },
    MessageStream: 'outbound'
  }
]);

results.forEach((result, i) => {
  if (result.ErrorCode === 0) {
    console.log(`Email ${i + 1} sent: ${result.MessageID}`);
  } else {
    console.error(`Email ${i + 1} failed: ${result.Message}`);
  }
});
```
