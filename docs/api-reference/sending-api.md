---
title: Sending API
description: API endpoints for sending emails with attachments, templates, and delivery tracking
sidebar_position: 4
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Sending API

The Sending API provides endpoints for sending emails through EmailEngine with support for immediate and scheduled delivery, automatic retries, and queue management.

## Overview

EmailEngine sends emails through the **Submit API** endpoint, which handles both immediate and scheduled delivery with automatic retry logic.

### Key Features

| Feature | Description |
|---------|-------------|
| Immediate delivery | Emails sent right away |
| Scheduled sends | Use `sendAt` parameter for future delivery |
| Automatic retries | Built-in retry logic (configurable attempts) |
| Real-time webhooks | Delivery status notifications |
| Queue management | View, monitor, and cancel queued emails |

## Submit API

The Submit API sends emails and provides real-time delivery status via webhooks. Emails can be sent immediately or scheduled for later using the `sendAt` parameter.

### Endpoint

`POST /v1/account/:account/submit`

### Request Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `to` | array | No | Recipient addresses (can be omitted when derived from `reference` or supplied via `mailMerge`) |
| `subject` | string | No | Email subject |
| `text` | string | No | Plain text content |
| `html` | string | No | HTML content |
| `from` | object | No | Custom from address |
| `cc` | array | No | CC recipients |
| `bcc` | array | No | BCC recipients |
| `replyTo` | object | No | Reply-To address |
| `attachments` | array | No | File attachments |
| `headers` | object | No | Custom headers |
| `reference` | object | No | For replies/forwards |

No field is strictly required by the schema. In practice, provide `text` and/or `html` for the message content - or send a complete MIME message with `raw`, or use a stored `template` instead.

### Address Format

```json
{
  "name": "John Doe",
  "address": "john@example.com"
}
```

Or simplified:
```json
{ "address": "john@example.com" }
```

### Examples

<Tabs groupId="language">
<TabItem value="curl" label="cURL" default>

```bash
curl -X POST "http://localhost:3000/v1/account/user%40example.com/submit" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "to": [{ "name": "Jane Smith", "address": "jane@example.com" }],
    "subject": "Hello from EmailEngine",
    "text": "This is a test email sent via the API.",
    "html": "<p>This is a test email sent via the API.</p>"
  }'
```

</TabItem>
<TabItem value="nodejs" label="Node.js">

```javascript
const account = 'user@example.com';

const res = await fetch(
  `http://localhost:3000/v1/account/${encodeURIComponent(account)}/submit`,
  {
    method: 'POST',
    headers: {
      Authorization: 'Bearer YOUR_ACCESS_TOKEN',
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      to: [{ name: 'Jane Smith', address: 'jane@example.com' }],
      subject: 'Hello from EmailEngine',
      text: 'This is a test email sent via the API.',
      html: '<p>This is a test email sent via the API.</p>'
    })
  }
);

const { messageId, queueId } = await res.json();
console.log(messageId, queueId);
```

</TabItem>
<TabItem value="python" label="Python">

```python
import requests
from urllib.parse import quote

account = 'user@example.com'

res = requests.post(
    f'http://localhost:3000/v1/account/{quote(account)}/submit',
    headers={'Authorization': 'Bearer YOUR_ACCESS_TOKEN'},
    json={
        'to': [{'name': 'Jane Smith', 'address': 'jane@example.com'}],
        'subject': 'Hello from EmailEngine',
        'text': 'This is a test email sent via the API.',
        'html': '<p>This is a test email sent via the API.</p>'
    }
)

result = res.json()
print(result['messageId'], result['queueId'])
```

</TabItem>
</Tabs>

The account ID goes in the URL path, so remember to URL-encode it. An address that contains `@` or `+` breaks the route otherwise.

**Response:**

```json
{
  "response": "Queued for delivery",
  "messageId": "<a2184d08-a470-fec6-a493-fa211a3756e9@example.com>",
  "sendAt": "2025-01-15T10:30:00.000Z",
  "queueId": "d41f0423195f271f"
}
```

Keep the `messageId`. It is the value that comes back in [`messageSent`](/docs/webhooks/messagesent), [`messageDeliveryError`](/docs/webhooks/messagedeliveryerror), and [`messageBounce`](/docs/webhooks/messagebounce) events, so it is how you match a delivery outcome to the message you sent. Use `queueId` to inspect or cancel the message while it is still queued.

#### Attachments

Attachments are inlined into the request as base64:

```json
{
  "to": [{ "address": "client@example.com" }],
  "subject": "Invoice #12345",
  "html": "<h1>Invoice</h1><p>Please find your invoice attached.</p>",
  "attachments": [
    {
      "filename": "invoice.pdf",
      "content": "JVBERi0xLjQKJcfsj6IK...",
      "contentType": "application/pdf"
    }
  ]
}
```

#### Replies and forwards

Set `reference` instead of building the threading headers yourself. EmailEngine looks up the referenced message, then adds the `In-Reply-To` and `References` headers so the reply threads correctly in the recipient's client:

```json
{
  "to": [{ "address": "original-sender@example.com" }],
  "subject": "Re: Original Subject",
  "text": "This is my reply.",
  "reference": {
    "message": "AAAABAABNc",
    "action": "reply"
  }
}
```

Forwarding uses the same field with `"action": "forward"`, and `forwardAttachments: true` carries the original attachments along:

```json
{
  "to": [{ "address": "colleague@example.com" }],
  "reference": {
    "message": "AAAABAABNc",
    "action": "forward",
    "forwardAttachments": true
  }
}
```

`action` is one of `reply`, `reply-all`, or `forward`, and defaults to `reply`. Useful companions on `reference`:

| Field | Effect |
|-------|--------|
| `inline` | Quote the original message under your text, the way a mail client does |
| `ignoreMissing` | Send anyway if the referenced message is gone, instead of failing |
| `messageId` | Refuse to send unless the referenced message carries this `Message-ID` |

Omit `subject` and EmailEngine derives it from the referenced message, prefixing `Re:` for a reply or `Fwd:` for a forward unless the subject already starts with one. Omit `to` and the recipients are derived as well.

See [Replies and Forwards](/docs/sending/replies-forwards) for the full behavior.

### Webhooks

The Submit API triggers these webhook events:

**messageSent** - Message successfully sent
```json
{
  "event": "messageSent",
  "account": "user@example.com",
  "data": {
    "queueId": "queue_456def",
    "messageId": "<abc123@example.com>",
    "response": "250 2.0.0 OK"
  }
}
```

**messageDeliveryError** - Sending failed
```json
{
  "event": "messageDeliveryError",
  "account": "user@example.com",
  "data": {
    "queueId": "queue_456def",
    "error": "Mailbox not found",
    "smtpResponse": "550 5.1.1 User unknown",
    "smtpResponseCode": 550
  }
}
```

### Use Cases

- **Interactive email**: User clicks "send" button
- **Real-time notifications**: Password resets, confirmations
- **Transactional emails**: Order receipts, shipping notifications
- **One-off messages**: Personal emails, manual sends

[Detailed API reference →](/docs/api/post-v-1-account-account-submit)

---

## Scheduling Emails

To schedule emails for future delivery, use the Submit API with the `sendAt` parameter.

### Schedule Email Example

Add `sendAt` with an ISO 8601 timestamp. The message is held in the outbox until then:

```bash
curl -X POST "http://localhost:3000/v1/account/user%40example.com/submit" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "to": [{ "address": "client@example.com" }],
    "subject": "Meeting Reminder",
    "text": "Reminder: Meeting tomorrow at 10 AM.",
    "sendAt": "2025-01-20T09:00:00.000Z"
  }'
```

**Response:**
```json
{
  "response": "Queued for delivery",
  "messageId": "<a2184d08-a470-fec6-a493-fa211a3756e9@example.com>",
  "sendAt": "2025-01-20T09:00:00.000Z",
  "queueId": "d41f0423195f271f"
}
```

A scheduled message can be [cancelled](#cancel-outbox-message) with its `queueId` at any point before it is sent.

---

## Outbox Management

The outbox contains queued messages waiting to be sent. Use these endpoints to view and manage queued messages.

### List Outbox

**Endpoint:** `GET /v1/outbox`

[Detailed API reference →](/docs/api/get-v-1-outbox)

Lists all queued messages across all accounts.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `page` | number | Page number (0-indexed) |
| `pageSize` | number | Items per page (default 20) |

**Example:**

```bash
curl "http://localhost:3000/v1/outbox?pageSize=50" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

The response is paginated, with `total`, `page`, `pages`, and a `messages` array of entries in the shape shown below.

### Get Outbox Message

**Endpoint:** `GET /v1/outbox/:queueId`

[Detailed API reference →](/docs/api/get-v-1-outbox-queueid)

**Example:**

```bash
curl "http://localhost:3000/v1/outbox/d41f0423195f271f" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

**Response:**
```json
{
  "queueId": "d41f0423195f271f",
  "account": "user123",
  "messageId": "<a2184d08-a470-fec6-a493-fa211a3756e9@example.com>",
  "envelope": {
    "from": "sender@example.com",
    "to": ["client@example.com"]
  },
  "subject": "Newsletter January 2025",
  "created": "2025-01-15T10:00:00.000Z",
  "scheduled": "2025-01-20T09:00:00.000Z",
  "nextAttempt": "2025-01-20T09:00:05.000Z",
  "attemptsMade": 0,
  "attempts": 10,
  "progress": {
    "status": "queued"
  }
}
```

`attemptsMade` against `attempts` tells you how much of the retry budget is left, and `nextAttempt` when the next one is due. A message that has exhausted its attempts reports `progress.status` as `error` and carries the failure details.

### Cancel Outbox Message

**Endpoint:** `DELETE /v1/outbox/:queueId`

[Detailed API reference →](/docs/api/delete-v-1-outbox-queueid)

**Example:**

```bash
curl -X DELETE "http://localhost:3000/v1/outbox/d41f0423195f271f" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

The response reports whether the entry was removed:

```json
{ "deleted": true }
```

**Note:** Only messages still in the `queued` state can be cancelled. Once a submit worker has picked a message up, it is past the point of cancellation, so `deleted` comes back `false`.

### Outbox Message States

| Status | Description |
|--------|-------------|
| `queued` | Accepted and waiting for a delivery attempt |
| `processing` | A submit worker has picked the message up |
| `smtp-starting` | Opening the SMTP connection to the destination |
| `smtp-completed` | The SMTP transaction finished and the message was handed over |
| `submitted` | Delivered to the receiving server. Later bounces arrive as webhooks |
| `error` | Delivery failed. `progress.error` says why, and whether it will be retried |

`submitted` means the receiving server accepted the message, not that it reached the recipient's inbox. A message can still bounce afterwards, which arrives as a [`messageBounce`](/docs/webhooks/messagebounce) webhook rather than a change to this status.

[Detailed API reference →](/docs/api/get-v-1-outbox)

## Message Format

### Recipients

**To, CC, BCC:**
```json
{
  "to": [
    { "name": "John Doe", "address": "john@example.com" },
    { "address": "jane@example.com" }
  ],
  "cc": [
    { "address": "manager@example.com" }
  ],
  "bcc": [
    { "address": "archive@example.com" }
  ]
}
```

### Custom Headers

```json
{
  "headers": {
    "X-Campaign-ID": "newsletter-2025-01",
    "X-Priority": "1"
  }
}
```

### Content (Text & HTML)

```json
{
  "subject": "Welcome!",
  "text": "Plain text version",
  "html": "<h1>HTML version</h1><p>With formatting</p>"
}
```

Best practice: Always provide both `text` and `html` for maximum compatibility.

### Attachments

**Base64 Encoded:**
```json
{
  "attachments": [
    {
      "filename": "document.pdf",
      "content": "JVBERi0xLjQKJeLjz9MKMy...",
      "encoding": "base64",
      "contentType": "application/pdf"
    }
  ]
}
```

**Referencing an Existing Attachment:**

Attachment content must always be provided as a base64-encoded `content` string - fetching attachments from a URL is not supported. To attach a file that already exists in the mailbox (for example when forwarding), set the attachment's `reference` field to an existing attachment ID instead of providing `content`.

**Inline Images:**
```json
{
  "html": "<img src=\"cid:logo\">",
  "attachments": [
    {
      "filename": "logo.png",
      "content": "iVBORw0KGgoAAAANSU...",
      "encoding": "base64",
      "contentType": "image/png",
      "cid": "logo"
    }
  ]
}
```

### References (Replies & Forwards)

**Reply to Message:**
```json
{
  "reference": {
    "message": "AAAABAABNc",
    "action": "reply"
  }
}
```

**Reply All:**
```json
{
  "reference": {
    "message": "AAAABAABNc",
    "action": "reply-all"
  }
}
```

**Forward with Attachments:**
```json
{
  "reference": {
    "message": "AAAABAABNc",
    "action": "forward",
    "forwardAttachments": true
  }
}
```

## Sending from a Stored Template

Rather than assembling subject and body on every call, store the content once as a [template](/docs/sending/templates) and reference it by ID. EmailEngine renders it, so the same template is applied identically no matter which service submits the message:

```json
{
  "to": [{ "name": "John", "address": "john@example.com" }],
  "template": "welcome-email",
  "render": {
    "params": { "name": "John", "activationUrl": "https://example.com/activate/abc" }
  }
}
```

The template supplies the subject, HTML, and text; `render.params` fills its placeholders. Anything you set explicitly on the request overrides what the template provides.

## Sending to Many Recipients

Do not loop over recipients calling submit once each. Pass `mailMerge` instead, and EmailEngine generates one personalized message per entry from a single request, each with its own queue entry, tracking, and webhooks:

```json
{
  "template": "newsletter",
  "mailMerge": [
    {
      "to": { "name": "User One", "address": "user1@example.com" },
      "params": { "name": "User One" }
    },
    {
      "to": { "name": "User Two", "address": "user2@example.com" },
      "params": { "name": "User Two" }
    }
  ]
}
```

Using `mailMerge` disables the root-level `to`, `cc`, `bcc`, `messageId`, `envelope`, and `render` keys, since each entry carries its own. The response returns a `mailMerge` array with one result per recipient instead of a single `messageId`.

See [Mail Merge](/docs/sending/mail-merge) for personalization, batching, and how failures for one recipient are reported.

:::warning Sending limits still apply
Mail merge submits through the account's own mail server, which enforces its own rate and volume limits. Gmail and Microsoft 365 in particular will start deferring or rejecting mail well before a bulk campaign finishes. For genuine bulk mail, send through a [gateway](/docs/sending/basic-sending) built for it.
:::

## See Also

- [Basic sending](/docs/sending/basic-sending) - The same endpoint as a walkthrough
- [Replies and forwards](/docs/sending/replies-forwards) - The reference block in practice
- [Outbox queue](/docs/sending/outbox-queue) - What happens after the call returns
- [messageSent and messageFailed](/docs/webhooks/messagesent) - How delivery is reported
