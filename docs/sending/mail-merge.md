---
title: Mail Merge
sidebar_position: 4
description: Send bulk personalized emails with EmailEngine's mail merge feature and Handlebars templates
---

# Mail Merge

Use the `mailMerge` array in the [message submission API](/docs/api/post-v-1-account-account-submit) call to generate per-recipient copies of the same message, inject template variables, and keep each copy in the mailbox's Sent Mail folder.

## Why It Matters

Bulk-sending receipts, onboarding tips or weekly digests from **your customer's** mailbox means better deliverability and brand consistency - but you don't want 500 addresses exposed in the `To` header. EmailEngine turns one REST call into N fully-formed messages, so every recipient feels like the only one.

## How Mail Merge Works

Instead of calling the submit API multiple times, you:

1. Drop `to`/`cc`/`bcc` from your payload
2. Add a `mailMerge` array with per-recipient data
3. EmailEngine fans out the request into distinct messages
4. Each message gets its own Message-ID for tracking
5. Handlebars templates enable personalization

## Basic Mail Merge

### Broadcasting Same Content

Send the same message to multiple recipients without exposing addresses:

```bash
curl -XPOST "https://emailengine.example.com/v1/account/example/submit" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "subject": "Test message",
    "html": "<p>Each recipient will get the same message</p>",
    "mailMerge": [
      {
        "to": {
          "name": "Ada Lovelace",
          "address": "ada@example.com"
        }
      },
      {
        "to": {
          "name": "Grace Hopper",
          "address": "grace@example.com"
        }
      }
    ]
  }'
```

**Response:**

```json
{
  "mailMerge": [
    {
      "success": true,
      "to": {
        "name": "Ada Lovelace",
        "address": "ada@example.com"
      },
      "messageId": "<91853631-2329-7f13-a4df-da377d873787@example.com>",
      "queueId": "182080c50b63e7e232a",
      "sendAt": "2025-05-14T09:12:23.123Z"
    },
    {
      "success": true,
      "to": {
        "name": "Grace Hopper",
        "address": "grace@example.com"
      },
      "messageId": "<8b47f91c-06f3-b555-ee19-2c99908aff25@example.com>",
      "queueId": "182080c50f283f49252",
      "sendAt": "2025-05-14T09:12:23.123Z"
    }
  ]
}
```

Each recipient:

- Sees only their own address in the `To` field
- Receives a message with a unique Message-ID
- Gets their own queue entry for tracking

### Skip Sent Folder Copies

For bulk sending, you might not want to save 1000 copies to the Sent Mail folder:

```json
{
  "subject": "Newsletter",
  "html": "<p>Weekly digest</p>",
  "copy": false,
  "mailMerge": [{ "to": { "address": "user1@example.com" } }, { "to": { "address": "user2@example.com" } }]
}
```

## Personalization with Handlebars

### Basic Personalization

Inject per-recipient data using Handlebars syntax:

```bash
curl -XPOST "https://emailengine.example.com/v1/account/example/submit" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "subject": "Test message for {{params.nickname}}",
    "html": "<p>Hello {{params.nickname}}, welcome to our service!</p>",
    "mailMerge": [
      {
        "to": {
          "name": "Ada Lovelace",
          "address": "ada@example.com"
        },
        "params": {
          "nickname": "ada"
        }
      },
      {
        "to": {
          "name": "Grace Hopper",
          "address": "grace@example.com"
        },
        "params": {
          "nickname": "grace"
        }
      }
    ]
  }'
```

**Important:** Plain-text fields (`subject`, `text`) are rendered without HTML escaping, so double braces `{{…}}` are always safe there - triple braces are never needed.

For HTML fields (`html`, `previewText`), use double braces `{{…}}` to escape HTML entities, or triple braces `{{{…}}}` if you want to inject raw HTML.

### Built-in Variables

EmailEngine provides built-in variables you can reference:

```handlebars
Hello
{{params.name}}, Your account:
{{account.email}}
Account name:
{{account.name}}
Support:
{{service.url}}
```

Available variables:

- `{{account.email}}` - The sender's email address
- `{{account.name}}` - The sender's display name
- `{{service.url}}` - EmailEngine instance URL
- `{{params.*}}` - Your custom parameters

### Complex Personalization

Include rich personalization data:

```json
{
  "subject": "Your order #{{params.orderNumber}} has shipped",
  "html": "<h1>Hi {{params.firstName}},</h1><p>Your order <strong>{{params.orderNumber}}</strong> has shipped!</p><p>Tracking: <a href=\"{{params.trackingUrl}}\">{{params.trackingNumber}}</a></p><p>Total: ${{params.orderTotal}}</p>",
  "mailMerge": [
    {
      "to": { "address": "ada@example.com" },
      "params": {
        "firstName": "Ada",
        "orderNumber": "12345",
        "orderTotal": "99.99",
        "trackingNumber": "1Z999AA10123456784",
        "trackingUrl": "https://tracking.example.com/1Z999AA10123456784"
      }
    }
  ]
}
```

### Conditional Content

Use Handlebars helpers for conditional content:

```handlebars
<p>Hello {{params.firstName}},</p>

{{#if params.isPremium}}
  <p>As a premium member, you get 20% off!</p>
{{else}}
  <p>Upgrade to premium for exclusive discounts.</p>
{{/if}}

{{#each params.items}}
  <li>{{this.name}} - ${{this.price}}</li>
{{/each}}
```

With data:

```json
{
  "mailMerge": [
    {
      "to": { "address": "ada@example.com" },
      "params": {
        "firstName": "Ada",
        "isPremium": true,
        "items": [
          { "name": "Product A", "price": "29.99" },
          { "name": "Product B", "price": "39.99" }
        ]
      }
    }
  ]
}
```

## Using with Templates

Combine mail merge with [stored templates](./templates.md) for maximum efficiency.

### Create a Template

First, create a template via API or UI:

```bash
curl -XPOST "https://emailengine.example.com/v1/templates/template" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "account": null,
    "name": "Welcome Email",
    "description": "Welcome new users",
    "content": {
      "subject": "Welcome {{params.nickname}}!",
      "html": "<h1>Hello {{params.nickname}}</h1><p>Welcome to our service!</p>",
      "text": "Hello {{params.nickname}}\n\nWelcome to our service!"
    }
  }'
```

**Response:**

```json
{
  "created": true,
  "account": null,
  "id": "AAABgggrm00AAAABZWtpcmk"
}
```

### Use Template with Mail Merge

Reference the template ID in your mail merge:

```bash
curl -XPOST "https://emailengine.example.com/v1/account/example/submit" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "template": "AAABgggrm00AAAABZWtpcmk",
    "mailMerge": [
      {
        "to": {
          "name": "Ada Lovelace",
          "address": "ada@example.com"
        },
        "params": {
          "nickname": "ada"
        }
      },
      {
        "to": {
          "name": "Grace Hopper",
          "address": "grace@example.com"
        },
        "params": {
          "nickname": "grace"
        }
      }
    ]
  }'
```

EmailEngine:

1. Loads the template
2. Applies each recipient's `params` to the template
3. Generates individual messages
4. Queues each for delivery

## Advanced Features

### Per-Recipient Custom Message-ID

Override the Message-ID for specific recipients:

```json
{
  "subject": "Team Update",
  "html": "<p>Update for {{params.teamName}}</p>",
  "mailMerge": [
    {
      "to": { "address": "alice@example.com" },
      "messageId": "<custom-id-alice@example.com>",
      "params": {
        "teamName": "Engineering"
      }
    },
    {
      "to": { "address": "bob@example.com" },
      "params": {
        "teamName": "Sales"
      }
    }
  ]
}
```

**Note:** Each mailMerge entry accepts a single recipient address. For sending to multiple recipients, use separate merge entries.

### Scheduled Mail Merge

Schedule the entire merge for future sending:

```json
{
  "subject": "Newsletter",
  "html": "<p>Weekly update</p>",
  "sendAt": "2025-12-25T09:00:00.000Z",
  "mailMerge": [{ "to": { "address": "user1@example.com" } }, { "to": { "address": "user2@example.com" } }]
}
```

All messages will be queued and sent at the specified time.

## Rate Limiting and Throttling

### Provider Limits

Be aware of provider sending limits:

- **Gmail**: ~500 recipients/day for free accounts, ~2000 for Google Workspace
- **Outlook**: ~300 recipients/day for personal, ~10,000 for business
- **Yahoo**: ~500 recipients/day
- **Custom SMTP**: Check with your provider

### Implement Throttling

For large merges, consider:

1. **Batch Processing**: Split large merges into smaller batches

```javascript
const recipients = [...]; // Large list
const batchSize = 100;

for (let i = 0; i < recipients.length; i += batchSize) {
  const batch = recipients.slice(i, i + batchSize);

  await fetch('/v1/account/example/submit', {
    method: 'POST',
    headers: { 'Authorization': 'Bearer <token>' },
    body: JSON.stringify({
      subject: 'Newsletter',
      html: '<p>Content</p>',
      mailMerge: batch.map(r => ({
        to: { address: r.email },
        params: { name: r.name }
      }))
    })
  });

  // Wait between batches
  await new Promise(resolve => setTimeout(resolve, 60000));
}
```

2. **Scheduled Delivery**: Spread messages over time using `sendAt`

3. **Monitor Queue**: Watch the outbox queue to avoid overload

### Check Queue Status

Monitor the queue to ensure you're not overwhelming the system:

```bash
curl "https://emailengine.example.com/v1/outbox" \
  -H "Authorization: Bearer <token>"
```

Look for:

- Number of waiting jobs
- Number of delayed jobs
- Any failed jobs

## Tracking Delivery

### Per-Message Tracking

Each mail merge entry gets a unique Message-ID and queue ID:

```json
{
  "mailMerge": [
    {
      "success": true,
      "to": { "address": "ada@example.com" },
      "messageId": "<unique-id-1@example.com>",
      "queueId": "abc123"
    },
    {
      "success": true,
      "to": { "address": "grace@example.com" },
      "messageId": "<unique-id-2@example.com>",
      "queueId": "def456"
    }
  ]
}
```

Store these IDs in your database to track delivery status.

### Webhook Events

Each message triggers its own webhooks:

**messageSent** (per recipient):

```json
{
  "event": "messageSent",
  "data": {
    "messageId": "<unique-id-1@example.com>",
    "queueId": "abc123",
    "envelope": {
      "from": "sender@example.com",
      "to": ["ada@example.com"]
    }
  }
}
```

**messageDeliveryError** (on every failed delivery attempt, including the final one):

```json
{
  "event": "messageDeliveryError",
  "data": {
    "queueId": "abc123",
    "messageId": "<unique-id-1@example.com>",
    "error": "Connection timeout",
    "job": {
      "attemptsMade": 1,
      "attempts": 10,
      "nextAttempt": "2025-05-14T15:07:45.465Z"
    }
  }
}
```

**messageFailed** (if all retries exhausted):

```json
{
  "event": "messageFailed",
  "data": {
    "queueId": "abc123",
    "messageId": "<unique-id-1@example.com>",
    "error": "Max retries exceeded"
  }
}
```

### Track Delivery Status

Build a tracking system:

```javascript
// Store merge results
const mergeSendResult = await sendMailMerge(...);
const deliveryTracking = mergeSendResult.mailMerge.map(entry => ({
  recipient: entry.to.address,
  messageId: entry.messageId,
  queueId: entry.queueId,
  status: 'queued',
  timestamp: new Date()
}));

// Save to database
await db.deliveryTracking.insertMany(deliveryTracking);

// Webhook handler
app.post('/webhook', async (req, res) => {
  const { event, data } = req.body;

  if (event === 'messageSent') {
    await db.deliveryTracking.updateOne(
      { messageId: data.messageId },
      { $set: { status: 'sent', sentAt: data.date } }
    );
  } else if (event === 'messageFailed') {
    await db.deliveryTracking.updateOne(
      { messageId: data.messageId },
      { $set: { status: 'failed', error: data.error } }
    );
  }

  res.sendStatus(200);
});
```

## Common Pitfalls

### Template Escaping

**Problem:** Parameter values containing HTML markup are escaped in HTML content. Double braces in `html` HTML-escape the value, so a param like `<b>Ada</b>` renders as literal text instead of bold.

```json
{
  "html": "Hello {{params.signature}}"
}
```

**Solution:** Use triple braces in HTML content only when you intend to inject raw HTML:

```json
{
  "html": "Hello {{{params.signature}}}"
}
```

Plain-text fields (`subject`, `text`) are never HTML-escaped, so double braces are always correct there - escaped subjects like `&lt;Welcome&gt;` cannot occur.

### Queue Timeouts

**Problem:** Each generated message gets its own queue entry; if your merge size is huge, watch `/v1/outbox` for items that exceed EmailEngine's processing window.

**Solution:**

- Break large merges into smaller batches (100-500 per batch)
- Monitor queue depth
- Scale EmailEngine vertically (increase CPU/RAM and worker configuration)

### Unwanted Sent Copies

**Problem:** Mailbox gets filled with thousands of sent copies.

**Solution:** Set `"copy": false` in your payload:

```json
{
  "copy": false,
  "mailMerge": [{ "to": { "address": "ada@example.com" } }]
}
```

### Missing Parameters

**Problem:** Template references `{{params.name}}` but some recipients don't have that parameter.

```json
{
  "mailMerge": [
    {
      "to": { "address": "ada@example.com" },
      "params": { "name": "Ada" }
    },
    {
      "to": { "address": "bob@example.com" },
      "params": {}
    }
  ]
}
```

**Result:** Second email shows empty value or "undefined"

**Solution:** Always provide all required params, or use Handlebars conditionals:

```handlebars
Hello {{#if params.name}}{{params.name}}{{else}}there{{/if}}
```

### Rate Limit Exceeded

**Problem:** Sending too many messages too quickly.

**Solution:**

- Implement batching with delays
- Use `sendAt` to schedule over time
- Check provider limits
- Monitor error webhooks

## Performance Optimization

### Use Templates

Pre-create templates instead of including HTML in every API call:

- Reduces payload size
- Faster processing
- Easier to update content

### Optimize Params

Keep param objects lean:

- Only include necessary data
- Avoid large nested objects
- Don't duplicate account-level data

### Monitor Performance

Track metrics:

- Merge request processing time
- Queue depth
- Delivery success rate
- Error rate by recipient domain

## Testing Mail Merge

### Test with Small Batch

Always test with a small batch first:

```json
{
  "subject": "Test merge",
  "html": "<p>Hello {{params.name}}</p>",
  "mailMerge": [
    {
      "to": { "address": "test1@ethereal.email" },
      "params": { "name": "Test User 1" }
    },
    {
      "to": { "address": "test2@ethereal.email" },
      "params": { "name": "Test User 2" }
    }
  ]
}
```

### Verify Personalization

Check that each recipient gets personalized content:

1. Send to test addresses
2. Check each inbox
3. Verify variables were replaced correctly
4. Confirm no HTML escaping issues

### Test Error Handling

Test with invalid data to ensure graceful failure:

- Invalid email addresses
- Missing required params
- Oversized attachments
