# Handlebars Template Syntax

Postmark templates use Handlebars (Mustache-compatible) syntax. Templates are rendered server-side — no client-side library needed.

## Variables

```handlebars
Hello {{name}},

Your order {{order_id}} has been confirmed.
Thank you for shopping with {{company_name}}.
```

Variables render to an empty string if missing from `TemplateModel` — they do not throw errors.

## Unescaped HTML

Use triple braces to render raw HTML without escaping:

```handlebars
{{{html_content}}}
```

- Use `{{variable}}` for user-supplied content (HTML-escaped for security)
- Use `{{{variable}}}` only for trusted HTML you control

## Conditionals

```handlebars
{{#if premium_member}}
  <p>Thank you for being a premium member!</p>
{{else}}
  <p>Upgrade to premium for exclusive benefits.</p>
{{/if}}
```

Falsy values (`false`, `0`, `""`, `null`, `undefined`, `[]`) trigger the `else` branch.

### Nested Conditionals

```handlebars
{{#if order_shipped}}
  {{#if tracking_number}}
    <p>Track your order: {{tracking_number}}</p>
  {{else}}
    <p>Your order is on its way!</p>
  {{/if}}
{{else}}
  <p>Your order is being prepared.</p>
{{/if}}
```

## Iteration

```handlebars
<table>
  {{#each items}}
  <tr>
    <td>{{this.name}}</td>
    <td>{{this.quantity}}</td>
    <td>{{this.price}}</td>
  </tr>
  {{/each}}
</table>
```

Inside `{{#each}}`, use `{{this.field}}` or just `{{field}}` to access item properties.

### Index and First/Last

```handlebars
{{#each items}}
  <p>{{@index}}. {{this.name}}</p>
{{/each}}
```

`@index` is 0-based. `@first` and `@last` are also available as boolean helpers.

## Nested Objects

```handlebars
{{customer.name}}
{{customer.address.city}}
{{customer.address.zip}}
```

## Layout Placeholder

In Layout templates, use this special placeholder where the Standard template body is injected:

```handlebars
{{{@content}}}
```

Must use triple braces — the Standard template's rendered body is HTML and should not be escaped.

## Full TemplateModel Example

Template:

```handlebars
Hello {{name}},

{{#if premium_member}}
  Thank you for being a premium member!
{{/if}}

Your order {{order_id}} contains:
{{#each items}}
  - {{this.name}} x{{this.quantity}} — {{this.price}}
{{/each}}

Shipping to: {{customer.address.city}}, {{customer.address.state}}

{{{custom_message}}}
```

Corresponding `TemplateModel`:

```json
{
  "name": "Jane Doe",
  "premium_member": true,
  "order_id": "ORD-12345",
  "items": [
    { "name": "Widget", "quantity": 2, "price": "$19.99" },
    { "name": "Gadget", "quantity": 1, "price": "$29.99" }
  ],
  "customer": {
    "address": {
      "city": "San Francisco",
      "state": "CA"
    }
  },
  "custom_message": "<strong>Special offer inside!</strong>"
}
```

## Common Syntax Mistakes

| Mistake | Fix |
|---------|-----|
| `{{html_body}}` renders escaped HTML | Use `{{{html_body}}}` for unescaped HTML |
| `{{#each}}` items without `this.` | Use `{{this.field}}` inside each blocks |
| Missing variable shows as blank | Handlebars renders missing vars as empty string — validate your model |
| `{{@content}}` in Layout | Must use triple braces: `{{{@content}}}` |
