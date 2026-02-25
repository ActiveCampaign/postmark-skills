# Layout Templates

Layout templates provide a reusable wrapper (CSS, header, footer) that Standard templates inherit from. This enforces consistent branding without duplicating boilerplate across every template.

## How It Works

```
Layout Template
├── <html><head>styles</head><body>
├── <header>Logo, navigation</header>
├── {{{@content}}}   ← Standard template body injected here
├── <footer>Links, unsubscribe</footer>
└── </body></html>
```

The Standard template only defines the content section — the Layout wraps it automatically when sending.

## Template Types

### Standard Templates

Regular sendable templates that define subject, HTML body, and text body:

```json
{
  "Name": "Order Confirmation",
  "Alias": "order-confirmation",
  "Subject": "Order {{order_id}} confirmed",
  "HtmlBody": "<h1>Order Confirmed</h1><p>Hi {{name}}, your order {{order_id}} is confirmed.</p>",
  "TextBody": "Order Confirmed\nHi {{name}}, your order {{order_id}} is confirmed.",
  "TemplateType": "Standard"
}
```

### Layout Templates

Reusable wrappers that inject Standard template content via `{{{@content}}}`:

```json
{
  "Name": "Base Layout",
  "Alias": "base-layout",
  "HtmlBody": "<html><body><header>...</header><main>{{{@content}}}</main><footer>...</footer></body></html>",
  "TextBody": "{{{@content}}}\n\n---\n(c) 2025 Your Company",
  "TemplateType": "Layout"
}
```

**Layouts do not have a Subject** — the subject is defined on the Standard template.

## Creating a Layout

```javascript
const postmark = require('postmark');
const client = new postmark.ServerClient(process.env.POSTMARK_SERVER_TOKEN);

const layout = await client.createTemplate({
  Name: 'Company Layout',
  Alias: 'company-layout',
  HtmlBody: `
    <!DOCTYPE html>
    <html>
    <head>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <style>
        body { font-family: -apple-system, BlinkMacSystemFont, sans-serif; margin: 0; padding: 0; background: #f4f4f4; }
        .wrapper { max-width: 600px; margin: 0 auto; background: #ffffff; }
        .header { background: #0a2540; padding: 24px; text-align: center; }
        .header img { height: 32px; }
        .content { padding: 32px 40px; }
        .footer { padding: 20px 40px; text-align: center; color: #666; font-size: 12px; }
        .footer a { color: #666; }
      </style>
    </head>
    <body>
      <div class="wrapper">
        <div class="header">
          <img src="https://yourdomain.com/logo.png" alt="Your Company">
        </div>
        <div class="content">
          {{{@content}}}
        </div>
        <div class="footer">
          <p>&copy; {{year}} Your Company, Inc.</p>
          {{#if unsubscribe_url}}
            <p><a href="{{unsubscribe_url}}">Unsubscribe</a></p>
          {{/if}}
        </div>
      </div>
    </body>
    </html>
  `,
  TextBody: `{{{@content}}}

---
(c) {{year}} Your Company, Inc.
{{#if unsubscribe_url}}Unsubscribe: {{unsubscribe_url}}{{/if}}`,
  TemplateType: 'Layout'
});

console.log('Layout ID:', layout.TemplateId);
```

## Assigning a Layout to a Standard Template

Set `LayoutTemplate` to the layout's alias (or ID):

```javascript
const template = await client.createTemplate({
  Name: 'Order Confirmation',
  Alias: 'order-confirmation',
  LayoutTemplate: 'company-layout',
  Subject: 'Your order {{order_id}} is confirmed',
  HtmlBody: `
    <h1>Order Confirmed!</h1>
    <p>Hi {{name}}, your order <strong>{{order_id}}</strong> is confirmed.</p>
    <table>
      {{#each items}}
      <tr>
        <td>{{this.name}}</td>
        <td>{{this.price}}</td>
      </tr>
      {{/each}}
    </table>
    <p>Total: {{order_total}}</p>
  `,
  TextBody: `Order Confirmed!\n\nHi {{name}}, your order {{order_id}} is confirmed.\n\nTotal: {{order_total}}`,
  TemplateType: 'Standard'
});
```

The Standard template's `HtmlBody` replaces `{{{@content}}}` in the Layout automatically.

## Variables Flow Through Both

Variables in `TemplateModel` are accessible in both the Layout and the Standard template:

```javascript
await client.sendEmailWithTemplate({
  From: 'orders@yourdomain.com',
  To: 'customer@example.com',
  TemplateAlias: 'order-confirmation',
  TemplateModel: {
    // Used by Standard template
    name: 'Jane Doe',
    order_id: 'ORD-12345',
    items: [{ name: 'Widget', price: '$19.99' }],
    order_total: '$19.99',
    // Used by Layout template
    year: '2025',
    unsubscribe_url: null  // null = condition is false, footer link hidden
  },
  MessageStream: 'outbound'
});
```

## Changing or Removing a Layout

```javascript
// Assign a different layout
await client.editTemplate('order-confirmation', {
  LayoutTemplate: 'new-layout'
});

// Remove a layout (set to null)
await client.editTemplate('order-confirmation', {
  LayoutTemplate: null
});
```

## Rules and Constraints

| Rule | Detail |
|------|--------|
| `{{{@content}}}` required | Layout must include this placeholder — triple braces |
| No Subject on Layout | Subject is defined only on Standard templates |
| Layouts are not sendable | You can only send Standard templates, not Layouts directly |
| Delete order | Cannot delete a Layout that has dependent Standard templates |
| No nested layouts | Layouts cannot inherit from other Layouts |
| Variables shared | TemplateModel variables are available to both Layout and Standard |
