---
title: Email Templates
sidebar_position: 6
description: Create and manage reusable email templates with Handlebars for consistent messaging
---

# Email Templates

Email templating allows you to prepare and reuse email content efficiently. Create templates once, then reference them when sending to ensure consistent branding and messaging across all your emails.

## Why Use Templates

Templates provide several benefits:

- **Consistency**: Ensure brand consistency across all emails
- **Efficiency**: Define content once, reuse many times
- **Maintainability**: Update content in one place
- **Personalization**: Use Handlebars for dynamic content
- **Separation of concerns**: Keep email content separate from application logic

## Managing Templates

You can manage templates in two ways:

1. **Templates API**: Programmatically create, update, and delete templates
2. **Admin Interface**: Visual interface at **Templates** in the side menu

![Email Templates List](/img/screenshots/15-templates-with-data.png)
*Email templates list in the admin interface*

![Template Editor](/img/screenshots/16-template-editor.png)
*Template editor showing Handlebars syntax and fields*

## Creating Templates

### Via API

Create a template using the [create template API](/docs/api/post-v-1-templates-template):

```bash
curl -XPOST "https://emailengine.example.com/v1/templates/template" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "example",
    "name": "Welcome Email",
    "description": "Welcome new users to the platform",
    "content": {
      "subject": "Welcome to {{params.companyName}}!",
      "text": "Hello {{params.firstName}},\n\nWelcome to {{params.companyName}}!",
      "html": "<h1>Hello {{params.firstName}}</h1><p>Welcome to <strong>{{params.companyName}}</strong>!</p>"
    }
  }'
```

**Response:**

```json
{
  "created": true,
  "account": "example",
  "id": "AAABgUIbuG0AAAAE"
}
```

Save the `id` value to reference this template when sending.

### Via Admin Interface

1. Navigate to **Email templates** in the EmailEngine UI
2. Click **Create Template**
3. Fill in template details:
   - **Name**: Template identifier (required)
   - **Description**: What this template is for (optional)
   - **Subject**: Email subject with optional Handlebars
   - **Text**: Plain text version with optional Handlebars
   - **HTML**: HTML version with optional Handlebars
4. Click **Save**

The template page offers a "Send test email" action so you can check the rendered output in a real mailbox.

## Using Templates

### Basic Usage

When sending emails using the Submission API, set the `template` property instead of `subject`, `html`, or `text`:

```bash
curl -XPOST "https://emailengine.example.com/v1/account/example/submit" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "to": [
      {
        "name": "Recipient Name",
        "address": "recipient@example.com"
      }
    ],
    "template": "AAABgUIbuG0AAAAE",
    "render": {
      "params": {
        "firstName": "Alice",
        "companyName": "Acme Corp"
      }
    }
  }'
```

EmailEngine loads the template and renders it with your provided params.

### With Other Properties

You can include any other valid submission properties:

```bash
curl -XPOST "https://emailengine.example.com/v1/account/example/submit" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "to": [{ "address": "recipient@example.com" }],
    "template": "AAABgUIbuG0AAAAE",
    "render": {
      "params": {
        "firstName": "Alice",
        "companyName": "Acme Corp"
      }
    },
    "replyTo": { "address": "support@example.com" },
    "attachments": [
      {
        "filename": "welcome.pdf",
        "content": "base64-encoded-content"
      }
    ]
  }'
```

## Template Syntax

Templates use [Handlebars](https://handlebarsjs.com/) for dynamic content. EmailEngine extends Handlebars with additional helpers compatible with [SendGrid's dynamic templates](https://docs.sendgrid.com/for-developers/sending-email/using-handlebars), making migration between platforms straightforward.

### Variables

Insert variables using double or triple braces:

```handlebars
Hello {{params.firstName}} {{params.lastName}}!

Subject: Welcome {{params.name}}
```

- **Double braces** `{{...}}`: HTML-escape the value in HTML content (`html`, `previewText`, `markdown`); plain-text fields (`subject`, `text`) are rendered without escaping, so double braces are always safe there
- **Triple braces** `{{{...}}}`: No escaping - only needed in HTML content when you want to inject raw HTML

### Built-in Variables

EmailEngine provides built-in variables:

```handlebars
From: {{account.name}} <{{account.email}}>
Support: {{service.url}}
Custom params: {{params.anyKey}}
```

Available variables:

- `{{account.email}}` - Sender's email address
- `{{account.name}}` - Sender's display name
- `{{service.url}}` - EmailEngine instance URL
- `{{params.*}}` - Any custom parameters you provide

### Conditionals

Use `if`/`else` for conditional content:

```handlebars
<p>Hello {{params.firstName}},</p>

{{#if params.isPremium}}
  <p>Welcome to our premium tier! You have access to all features.</p>
{{else}}
  <p>Welcome! Upgrade to premium for additional features.</p>
{{/if}}
```

### Loops

Iterate over arrays with `each`:

```handlebars
<h2>Your order:</h2>
<ul>
{{#each params.items}}
  <li>{{this.name}} - ${{this.price}}</li>
{{/each}}
</ul>

<p>Total: ${{params.total}}</p>
```

With data:

```json
{
  "params": {
    "items": [
      { "name": "Product A", "price": "29.99" },
      { "name": "Product B", "price": "39.99" }
    ],
    "total": "69.98"
  }
}
```

### Basic Helpers

Common built-in Handlebars helpers:

```handlebars
{{!-- Comments --}}
{{! This is a comment and won't appear in output }}

{{!-- Unless (opposite of if) --}}
{{#unless params.isSubscribed}}
  <p>Subscribe to our newsletter!</p>
{{/unless}}

{{!-- With (change context) --}}
{{#with params.user}}
  <p>{{firstName}} {{lastName}}</p>
  <p>{{email}}</p>
{{/with}}
```

### SendGrid-Compatible Helpers

EmailEngine provides additional helpers that are compatible with SendGrid's dynamic templates. These helpers enable advanced templating capabilities for comparisons, date formatting, and default values.

#### Comparison Helpers

##### equals

Check if two values are equal. Uses loose equality (`==`) for automatic type coercion.

```handlebars
{{#equals params.status "active"}}
  <p>Your account is active.</p>
{{else}}
  <p>Your account is inactive.</p>
{{/equals}}
```

```handlebars
{{#equals params.customerCode params.winningCode}}
  <p>Congratulations! You have a winning code.</p>
{{/equals}}
```

##### notEquals

Check if two values are not equal. Uses loose inequality (`!=`) for automatic type coercion.

```handlebars
{{#notEquals params.role "admin"}}
  <p>You don't have admin privileges.</p>
{{/notEquals}}
```

##### greaterThan

Check if the first numeric value is greater than the second.

```handlebars
{{#greaterThan params.score 90}}
  <p>Excellent score! You're in the top tier.</p>
{{else}}
  <p>Keep working to improve your score.</p>
{{/greaterThan}}
```

```handlebars
{{#greaterThan params.cartTotal 100}}
  <p>You qualify for free shipping!</p>
{{/greaterThan}}
```

##### lessThan

Check if the first numeric value is less than the second.

```handlebars
{{#lessThan params.daysRemaining 7}}
  <p>Your subscription expires soon. Renew now!</p>
{{/lessThan}}
```

```handlebars
{{#lessThan params.inventory 10}}
  <p>Low stock - only {{params.inventory}} items left!</p>
{{/lessThan}}
```

#### Logical Helpers

##### and

Renders content only when all conditions are true. Accepts multiple arguments.

```handlebars
{{#and params.isVerified params.hasSubscription}}
  <p>Welcome back, verified subscriber!</p>
{{else}}
  <p>Please verify your email or subscribe to continue.</p>
{{/and}}
```

```handlebars
{{#and params.inStock params.hasDiscount params.isPremiumMember}}
  <p>Exclusive deal available for you!</p>
{{/and}}
```

##### or

Renders content when at least one condition is true. Accepts multiple arguments.

```handlebars
{{#or params.isAdmin params.isModerator}}
  <p>You have moderation privileges.</p>
{{/or}}
```

```handlebars
{{#or params.hasCoupon params.isPremium params.isFirstOrder}}
  <p>You're eligible for a discount!</p>
{{/or}}
```

#### Value Helpers

##### insert

Insert a value with an optional default if the value is missing or empty.

```handlebars
<p>Hello {{insert params.firstName "default=Customer"}}!</p>
```

If `params.firstName` is empty or undefined, displays "Customer" instead.

```handlebars
<p>Your membership level: {{insert params.tier "default=Standard"}}</p>
```

##### length

Get the length of an array. Useful in conditionals to check if an array has items.

```handlebars
{{#if (length params.items)}}
  <p>You have {{length params.items}} items in your cart.</p>
{{else}}
  <p>Your cart is empty.</p>
{{/if}}
```

```handlebars
{{#greaterThan (length params.orders) 10}}
  <p>Thank you for being a loyal customer with over 10 orders!</p>
{{/greaterThan}}
```

#### Date Formatting

##### formatDate

Format dates using [Moment.js format tokens](https://momentjs.com/docs/#/displaying/format/). Accepts an optional timezone offset.

**Syntax:** `{{formatDate timestamp format [timezoneOffset]}}`

**Common format tokens:**

| Token | Output | Example |
|-------|--------|---------|
| `YYYY` | 4-digit year | 2025 |
| `YY` | 2-digit year | 25 |
| `MMMM` | Full month name | January |
| `MMM` | Short month name | Jan |
| `MM` | Month number (padded) | 01 |
| `DD` | Day of month (padded) | 05 |
| `D` | Day of month | 5 |
| `dddd` | Full weekday name | Monday |
| `ddd` | Short weekday name | Mon |
| `HH` | Hour (24h, padded) | 14 |
| `hh` | Hour (12h, padded) | 02 |
| `h` | Hour (12h) | 2 |
| `mm` | Minutes (padded) | 05 |
| `ss` | Seconds (padded) | 09 |
| `A` | AM/PM | PM |
| `a` | am/pm | pm |
| `ZZ` | Timezone offset | +0000 |

**Examples:**

```handlebars
<p>Order date: {{formatDate params.orderDate "MMMM DD, YYYY"}}</p>
<!-- Output: Order date: January 15, 2025 -->
```

```handlebars
<p>Event time: {{formatDate params.eventTime "dddd, MMMM D, YYYY [at] h:mm A"}}</p>
<!-- Output: Event time: Monday, January 15, 2025 at 2:30 PM -->
```

```handlebars
<p>Delivery: {{formatDate params.deliveryDate "MMM D, YYYY" "-0500"}}</p>
<!-- Output with EST timezone offset: Delivery: Jan 15, 2025 -->
```

```handlebars
<p>Created: {{formatDate params.timestamp "YYYY-MM-DD HH:mm:ss"}}</p>
<!-- Output: Created: 2025-01-15 14:30:00 -->
```

#### Iteration Helpers

##### each with Special Variables

When iterating over arrays, you have access to special variables:

```handlebars
<ol>
{{#each params.steps}}
  <li>Step {{@index}}: {{this}}</li>
{{/each}}
</ol>
```

| Variable | Description |
|----------|-------------|
| `{{@index}}` | Zero-based index of the current item |
| `{{@first}}` | True if this is the first item |
| `{{@last}}` | True if this is the last item |
| `{{this}}` | The current item value |

```handlebars
<ul>
{{#each params.items}}
  <li{{#if @first}} class="first"{{/if}}{{#if @last}} class="last"{{/if}}>
    {{this.name}}
  </li>
{{/each}}
</ul>
```

#### Root Context Access

Access top-level variables from within nested blocks using `@root`:

```handlebars
{{#each params.orders}}
  <div class="order">
    <p>Order #{{this.id}}</p>
    <p>Customer: {{@root.params.customerName}}</p>
    <p>Support: {{@root.service.url}}/support</p>
  </div>
{{/each}}
```

### Helpers Quick Reference

| Helper | Purpose | Example |
|--------|---------|---------|
| `{{#if}}` | Conditional rendering | `{{#if params.active}}...{{/if}}` |
| `{{#unless}}` | Inverse conditional | `{{#unless params.disabled}}...{{/unless}}` |
| `{{#each}}` | Iterate over arrays | `{{#each params.items}}...{{/each}}` |
| `{{#with}}` | Change context | `{{#with params.user}}...{{/with}}` |
| `{{#equals}}` | Equality check | `{{#equals a b}}...{{/equals}}` |
| `{{#notEquals}}` | Inequality check | `{{#notEquals a b}}...{{/notEquals}}` |
| `{{#greaterThan}}` | Numeric greater than | `{{#greaterThan a b}}...{{/greaterThan}}` |
| `{{#lessThan}}` | Numeric less than | `{{#lessThan a b}}...{{/lessThan}}` |
| `{{#and}}` | All conditions true | `{{#and a b c}}...{{/and}}` |
| `{{#or}}` | Any condition true | `{{#or a b c}}...{{/or}}` |
| `{{insert}}` | Value with default | `{{insert var "default=fallback"}}` |
| `{{length}}` | Array length | `{{length params.items}}` |
| `{{formatDate}}` | Format dates | `{{formatDate date "MMM D, YYYY"}}` |

## Template Examples

### Welcome Email

```handlebars
Subject: Welcome to {{params.companyName}}, {{params.firstName}}!

HTML:
<html>
<head>
  <style>
    body { font-family: Arial, sans-serif; }
    .header { background: #4CAF50; color: white; padding: 20px; }
    .content { padding: 20px; }
  </style>
</head>
<body>
  <div class="header">
    <h1>Welcome to {{params.companyName}}!</h1>
  </div>
  <div class="content">
    <p>Hello {{params.firstName}},</p>
    <p>Thank you for joining {{params.companyName}}. We're excited to have you on board!</p>

    {{#if params.nextSteps}}
    <h2>Next Steps:</h2>
    <ul>
    {{#each params.nextSteps}}
      <li>{{this}}</li>
    {{/each}}
    </ul>
    {{/if}}

    <p>If you have any questions, reply to this email or visit our <a href="{{service.url}}/help">help center</a>.</p>

    <p>Best regards,<br>
    The {{params.companyName}} Team</p>
  </div>
</body>
</html>

Text:
Welcome to {{params.companyName}}!

Hello {{params.firstName}},

Thank you for joining {{params.companyName}}. We're excited to have you on board!

{{#if params.nextSteps}}
Next Steps:
{{#each params.nextSteps}}
- {{this}}
{{/each}}
{{/if}}

If you have any questions, reply to this email or visit our help center at {{service.url}}/help.

Best regards,
The {{params.companyName}} Team
```

### Order Confirmation

```handlebars
Subject: Order #{{params.orderNumber}} Confirmed

HTML:
<h1>Order Confirmation</h1>
<p>Hello {{params.customerName}},</p>
<p>Your order <strong>#{{params.orderNumber}}</strong> has been confirmed!</p>

<h2>Order Details:</h2>
<table style="width: 100%; border-collapse: collapse;">
  <thead>
    <tr style="background: #f5f5f5;">
      <th style="padding: 10px; text-align: left;">Item</th>
      <th style="padding: 10px; text-align: right;">Qty</th>
      <th style="padding: 10px; text-align: right;">Price</th>
    </tr>
  </thead>
  <tbody>
  {{#each params.items}}
    <tr style="border-bottom: 1px solid #ddd;">
      <td style="padding: 10px;">{{this.name}}</td>
      <td style="padding: 10px; text-align: right;">{{this.quantity}}</td>
      <td style="padding: 10px; text-align: right;">${{this.price}}</td>
    </tr>
  {{/each}}
  </tbody>
  <tfoot>
    <tr style="font-weight: bold; background: #f5f5f5;">
      <td colspan="2" style="padding: 10px; text-align: right;">Total:</td>
      <td style="padding: 10px; text-align: right;">${{params.total}}</td>
    </tr>
  </tfoot>
</table>

<p>Estimated delivery: {{params.estimatedDelivery}}</p>

<p>Track your order: <a href="{{params.trackingUrl}}">{{params.trackingNumber}}</a></p>
```

### Password Reset

```handlebars
Subject: Reset Your Password

HTML:
<h1>Password Reset Request</h1>
<p>Hello {{params.firstName}},</p>
<p>We received a request to reset your password for your {{account.name}} account.</p>

<p>Click the button below to reset your password:</p>

<p style="text-align: center; margin: 30px 0;">
  <a href="{{params.resetUrl}}" style="background: #4CAF50; color: white; padding: 15px 30px; text-decoration: none; border-radius: 5px; display: inline-block;">
    Reset Password
  </a>
</p>

<p>Or copy and paste this link into your browser:</p>
<p style="word-break: break-all; color: #666;">{{params.resetUrl}}</p>

<p><strong>This link will expire in {{params.expiryHours}} hours.</strong></p>

<p>If you didn't request this password reset, you can safely ignore this email.</p>

<p>Best regards,<br>
{{account.name}}</p>
```

## Template Management

### List All Templates

Retrieve all templates using the [list templates API](/docs/api/get-v-1-templates):

```bash
curl "https://emailengine.example.com/v1/templates?account=example" \
  -H "Authorization: Bearer <token>"
```

**Response:**

```json
{
  "account": "example",
  "total": 2,
  "page": 0,
  "pages": 1,
  "templates": [
    {
      "id": "AAABgUIbuG0AAAAE",
      "name": "Welcome Email",
      "description": "Welcome new users",
      "format": "html",
      "created": "2025-05-14T10:00:00.000Z",
      "updated": "2025-05-14T12:00:00.000Z"
    },
    {
      "id": "AAABgUIbuG0AAAAF",
      "name": "Order Confirmation",
      "description": "Confirm orders",
      "format": "html",
      "created": "2025-05-14T11:00:00.000Z",
      "updated": "2025-05-14T11:00:00.000Z"
    }
  ]
}
```

### Get Template Details

Use the [get template API](/docs/api/get-v-1-templates-template-template):

```bash
curl "https://emailengine.example.com/v1/templates/template/AAABgUIbuG0AAAAE" \
  -H "Authorization: Bearer <token>"
```

**Response:**

```json
{
  "account": "example",
  "id": "AAABgUIbuG0AAAAE",
  "name": "Welcome Email",
  "description": "Welcome new users to the platform",
  "format": "html",
  "created": "2025-05-14T10:00:00.000Z",
  "updated": "2025-05-14T12:00:00.000Z",
  "content": {
    "subject": "Welcome to {{params.companyName}}!",
    "text": "Hello {{params.firstName}}...",
    "html": "<h1>Hello {{params.firstName}}</h1>..."
  }
}
```

### Update Template

Use the [update template API](/docs/api/put-v-1-templates-template-template):

```bash
curl -XPUT "https://emailengine.example.com/v1/templates/template/AAABgUIbuG0AAAAE" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "content": {
      "subject": "Welcome to {{params.companyName}}, {{params.firstName}}!",
      "text": "Hello {{params.firstName}}...",
      "html": "<h1>Updated content</h1>"
    }
  }'
```

Top-level fields (`name`, `description`, `format`) are merged - include only the ones you want to change. The `content` object is different: when provided, it replaces the stored content entirely, so resubmit all content fields (`subject`, `text`, `html`, `previewText`) - any field you omit is removed from the template.

### Delete Template

Use the [delete template API](/docs/api/delete-v-1-templates-template-template):

```bash
curl -XDELETE "https://emailengine.example.com/v1/templates/template/AAABgUIbuG0AAAAE" \
  -H "Authorization: Bearer <token>"
```

## Using Templates with Mail Merge

Templates work great with mail merge for bulk personalized sending:

```bash
curl -XPOST "https://emailengine.example.com/v1/account/example/submit" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "template": "AAABgUIbuG0AAAAE",
    "mailMerge": [
      {
        "to": { "name": "Alice", "address": "alice@example.com" },
        "params": {
          "firstName": "Alice",
          "companyName": "Acme Corp",
          "isPremium": true
        }
      },
      {
        "to": { "name": "Bob", "address": "bob@example.com" },
        "params": {
          "firstName": "Bob",
          "companyName": "Acme Corp",
          "isPremium": false
        }
      }
    ]
  }'
```

Each recipient gets a personalized email based on their params.

