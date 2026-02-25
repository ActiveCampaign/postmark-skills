---
name: postmark-templates
description: Use when creating, managing, or sending with Postmark server-side email templates — Handlebars syntax, layout inheritance, template validation, and cross-server pushing.
license: MIT
metadata:
  author: postmark
  version: "1.0.0"
---

# Email Templates with Postmark

## Overview

Postmark templates are server-side email templates using Handlebars syntax. Templates are rendered on Postmark's servers — no client-side rendering library needed.

| Feature | Description |
|---------|-------------|
| **Syntax** | Handlebars (Mustache-compatible) |
| **Rendering** | Server-side — no React, no client library |
| **Types** | Standard templates and Layout templates |
| **Inheritance** | Standard templates can inherit from a Layout |
| **Validation** | API endpoint to test-render templates before sending |
| **Cross-server** | Push templates between servers (staging → production) |
| **Limit** | 100 templates per server (contact support for more) |

## Quick Start

1. **Create a template** via API or the [Postmark dashboard](https://account.postmarkapp.com)
2. **Define variables** using Handlebars syntax: `{{variable_name}}`
3. **Send with template** using `POST /email/withTemplate` or `POST /email/batchWithTemplates`
4. **Pass data** via `TemplateModel` — Postmark renders and sends

## Template Syntax (Handlebars)

```handlebars
Hello {{name}},                            {{! variable }}
{{{html_content}}}                         {{! unescaped HTML }}
{{#if premium}}Premium member{{/if}}       {{! conditional }}
{{#each items}}{{this.name}}{{/each}}      {{! iteration }}
{{customer.address.city}}                  {{! nested object }}
```

See [references/handlebars-syntax.md](references/handlebars-syntax.md) for the full syntax reference including conditionals, iteration with index, nested objects, and common mistakes.

## Template Types

| Type | `TemplateType` | Sendable | Purpose |
|------|---------------|----------|---------|
| **Standard** | `"Standard"` | Yes | Defines subject, HTML, and text body |
| **Layout** | `"Layout"` | No | Reusable wrapper injected via `{{{@content}}}` |

Standard templates can reference a Layout via `LayoutTemplate: "layout-alias"`. The Standard template's body replaces `{{{@content}}}` in the Layout at send time.

See [references/layout-templates.md](references/layout-templates.md) for layout creation, assignment, and examples.

## API Endpoints

### Template CRUD

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/templates` | `POST` | Create a new template |
| `/templates` | `GET` | List all templates (`?count=100&offset=0&templateType=Standard`) |
| `/templates/{idOrAlias}` | `GET` | Get a single template |
| `/templates/{idOrAlias}` | `PUT` | Update a template |
| `/templates/{idOrAlias}` | `DELETE` | Delete a template |
| `/templates/validate` | `POST` | Validate template rendering |
| `/templates/push` | `PUT` | Push templates to another server |

### Sending with Templates

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/email/withTemplate` | `POST` | Send single email with template |
| `/email/batchWithTemplates` | `POST` | Send batch with templates (up to 500) |

## Send with Template

Always use `TemplateAlias` over `TemplateId` — aliases survive re-creation and work across environments:

```javascript
const postmark = require('postmark');
const client = new postmark.ServerClient(process.env.POSTMARK_SERVER_TOKEN);

await client.sendEmailWithTemplate({
  From: 'hello@yourdomain.com',
  To: 'customer@example.com',
  TemplateAlias: 'welcome-email',
  TemplateModel: {
    name: 'Jane Doe',
    product_name: 'Acme App'
  },
  MessageStream: 'outbound'
});
```

## Validate a Template

Test-render without sending — useful in CI/CD before deploying template changes:

```javascript
const validation = await client.validateTemplate({
  Subject: 'Welcome {{name}}',
  HtmlBody: '<h1>Hello {{name}}</h1>',
  TextBody: 'Hello {{name}}',
  TestRenderModel: { name: 'Test User' }
});

console.log('Valid:', validation.AllContentIsValid);
console.log('Rendered:', validation.Subject.RenderedContent);
```

See [references/template-api.md](references/template-api.md) for full CRUD operations (create, list, update, delete), batch sending, validate, and push between servers.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using `{{html}}` for HTML content | Use triple braces `{{{html}}}` for unescaped HTML |
| Forgetting `{{{@content}}}` in Layout | Layout templates must include `{{{@content}}}` placeholder |
| Deleting a Layout with dependents | Remove layout association from Standard templates first |
| Using Template ID across environments | Use `TemplateAlias` — it survives re-creation and works across servers |
| Not validating before deploy | Use `/templates/validate` to test-render before sending |
| Sending a Layout directly | Layouts are wrappers — you can only send Standard templates |
| Missing TemplateModel fields | Handlebars renders missing variables as empty strings — validate your data |
| Exceeding 100 templates | Contact Postmark support to increase the per-server limit |

## Notes

- Templates use Handlebars (Mustache-compatible) syntax — no React or client-side rendering needed
- Template aliases are strings; template IDs are integers — prefer aliases for portability
- `TemplateType` is either `Standard` (sendable) or `Layout` (wrapper)
- Layout inheritance: Standard template body replaces `{{{@content}}}` in the Layout
- Push templates between servers using the Account Token (not Server Token)
- Maximum 100 templates per server by default
- Template validation (`/templates/validate`) lets you test-render without sending
- Both `TemplateId` and `TemplateAlias` work for sending — use one, not both
