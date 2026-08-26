---
title: Sending API
description: API endpoints for sending emails with attachments, templates, and delivery tracking
sidebar_position: 5
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Sending API

The Sending API provides endpoints for sending emails through EmailEngine with support for immediate and scheduled delivery, automatic retries, and queue management.

## Overview

EmailEngine sends emails through the **Submit API** endpoint. A message is accepted into the outbox queue and delivered asynchronously, so a `200` response means queued, not sent. Follow the outcome through the [outbox endpoints](#outbox-management) or the delivery webhooks.

| Feature | Description |
|---------|-------------|
| Immediate delivery | Queued messages are picked up by a submit worker right away |
| Scheduled sends | Use `sendAt` for future delivery |
| Automatic retries | Failed attempts are retried with exponential backoff, up to `deliveryAttempts` |
| Delivery webhooks | `messageSent`, `messageDeliveryError`, `messageFailed`, `messageBounce` |
| Queue management | List, inspect, and cancel queued messages |

## Submit API

### Endpoint

`POST /v1/account/{account}/submit`

[Detailed API reference](/docs/api/post-v-1-account-account-submit)

The account ID goes in the URL path, so URL-encode it. An address that contains `@` or `+` breaks the route otherwise.

### Request Parameters

No field is required by the schema. In practice, provide `text` and/or `html` for the message content, or send a complete MIME message with `raw`, or load the content from a stored `template`.

**Content and addressing**

| Field | Type | Description |
|-------|------|-------------|
| `to`, `cc`, `bcc` | array | Recipients, as [address objects](#address-format). `to` can be omitted when it is derived from `reference` or supplied per entry in `mailMerge` |
| `from` | object | Sender address. Defaults to the account's configured address |
| `replyTo` | object or array | Reply-To address, or a list of them |
| `subject` | string | Subject line. Derived from the referenced message when omitted with `reference` |
| `text` | string | Plain text body |
| `html` | string | HTML body. An HTML-only submission gets no generated plain text part |
| `previewText` | string | Preview text shown by mail clients after the subject line |
| `attachments` | array | [Attachments](#attachments), base64 inlined or referenced by ID |
| `headers` | object | Custom headers as key-value pairs |
| `messageId` | string | Message-ID to use instead of a generated one |
| `envelope` | object | SMTP envelope (`from`, `to`) when it should differ from the headers |
| `raw` | string | A complete base64-encoded RFC 822 message. Fields sent alongside it override the matching headers inside it |
| `reference` | object | [Reply to or forward](#replies-and-forwards) a stored message. Its `forwardAttachments` flag is only accepted with `action: "forward"` |
| `template` | string | ID of a [stored template](#sending-from-a-stored-template) to load the content from |
| `render` | object or `false` | Template rendering: `format` (`html` or `markdown`) and `params`. Set to `false` to send the template as-is |
| `mailMerge` | array | [One message per recipient](#sending-to-many-recipients) from a single request |
| `listId` | string | List ID for mail merge, in hostname format. Lists are registered ad hoc. Only accepted together with `mailMerge` |

**Delivery**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `sendAt` | date-time | now | Hold the message in the outbox until this time |
| `deliveryAttempts` | integer | the `deliveryAttempts` setting, `10` if unset | How many delivery attempts to make before the message is considered failed |
| `gateway` | string | none | ID of an [SMTP gateway](/docs/api/post-v-1-gateway) to route the message through instead of the account's own SMTP server |
| `copy` | boolean or null | account default | Whether to upload the sent message to the Sent Mail folder. SMTP deliveries only: Gmail API and MS Graph accounts file sent messages themselves |
| `sentMailPath` | string | the account's Sent Mail folder | Folder to upload the sent copy to |
| `trackOpens`, `trackClicks` | boolean | the `trackOpens` and `trackClicks` settings, falling back to `trackSentMessages` | Add open and click tracking to this message. `trackingEnabled`, absent from the spec but still accepted, sets both at once and is what the SMTP interface's `X-EE-Tracking-Enabled` header maps to |
| `baseUrl` | string | `serviceUrl` | Base URL for tracking links. Must point at your EmailEngine instance |
| `dsn` | object | none | Delivery status notification request (`id`, `return`, `notify`, `recipient`) |
| `locale`, `tz` | string | none | Locale and timezone for template rendering |
| `proxy` | string | none | Proxy URL for the SMTP connection. A token with restricted permissions cannot set this |
| `localAddress` | string | none | Local IP address to bind the SMTP connection to |
| `dryRun` | boolean | `false` | Build the message but do not send it. The response carries the RFC 822 message as base64 in `preview`, without tracking |

**Request headers and query parameters**

| Name | In | Description |
|------|----|-------------|
| `Idempotency-Key` | header | Replays the stored result for a repeated request with the same key instead of queueing the message again. The response then carries an `idempotency` object with `status` `HIT` or `MISS`. Since v2.52.0 |
| `X-EE-Timeout` | header | Request timeout in milliseconds, up to `7200000`, overriding `EENGINE_TIMEOUT` |
| `useStructuredFormat` | query | MS Graph accounts only: send as structured JSON instead of raw MIME. Structured JSON is the only mode that honors a `from` address other than the mailbox's own, which a shared mailbox needs, but it does not preserve calendar invites and other special MIME parts; raw MIME preserves them and ignores `from` |
| `documentStore` | query | Deprecated. Load the referenced message from the Document Store, which is removed from releases starting 1 October 2026 |

### Address Format

```json
{
  "name": "John Doe",
  "address": "john@example.com"
}
```

Or without a display name:

```json
{ "address": "john@example.com" }
```

### Examples

<Tabs groupId="language">
<TabItem value="curl" label="cURL" default>

```bash
curl -X POST "https://emailengine.example.com/v1/account/user%40example.com/submit" \
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
  `https://emailengine.example.com/v1/account/${encodeURIComponent(account)}/submit`,
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
    f'https://emailengine.example.com/v1/account/{quote(account)}/submit',
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

### Response

```json
{
  "response": "Queued for delivery",
  "messageId": "<a2184d08-a470-fec6-a493-fa211a3756e9@example.com>",
  "sendAt": "2025-01-15T10:30:00.000Z",
  "queueId": "d41f0423195f271f"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `response` | string | `Queued for delivery`, or `Dry run` when `dryRun` was set |
| `messageId` | string | Message-ID header of the queued message. Not present for mail merge |
| `queueId` | string | Outbox queue entry ID. Not present for mail merge or for a dry run |
| `sendAt` | date-time | When the message is scheduled to be sent. Not present for a dry run |
| `reference` | object | Present when `reference` was used: `message`, `success`, and `error` if the referenced message could not be processed. For a Gmail API account `threadId` carries the thread the message was attached to |
| `preview` | string | The built RFC 822 message as base64, when `dryRun` was set. Not returned for mail merge |
| `idempotency` | object | Present when an `Idempotency-Key` header was sent: `key` and `status` (`HIT` or `MISS`) |
| `mailMerge` | array | One entry per recipient for [mail merge](#sending-to-many-recipients) submissions |

Keep the `messageId`. It is the value that comes back in [`messageSent`](/docs/webhooks/messagesent), [`messageDeliveryError`](/docs/webhooks/messagedeliveryerror), [`messageFailed`](/docs/webhooks/messagefailed), and [`messageBounce`](/docs/webhooks/messagebounce) events, so it is how you match a delivery outcome to the message you sent. Use `queueId` to inspect or cancel the message while it is still queued.

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
      "content": "JVBERi0xLjQKJcfsj6IK",
      "contentType": "application/pdf"
    }
  ]
}
```

Fetching attachments from a URL is not supported. To attach a file that already exists in the mailbox, for example when forwarding, set the attachment's `reference` field to an existing attachment ID and leave `content` out; the two are exclusive.

#### Replies and forwards

Set `reference` instead of building the threading headers yourself. EmailEngine looks up the referenced message, adds the `In-Reply-To` and `References` headers so the reply threads correctly in the recipient's client, and flags the original as answered or forwarded once the new message is sent:

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

`action` is one of `reply`, `reply-all`, or `forward`, and defaults to `reply`. Other fields on `reference`:

| Field | Effect |
|-------|--------|
| `inline` | Quote the original message under your text, the way a mail client does |
| `ignoreMissing` | Send anyway if the referenced message is gone, instead of failing |
| `messageId` | Refuse to send unless the referenced message carries this `Message-ID` |
| `threadId` | Gmail API accounts only: attach the outgoing message to this Gmail thread. IMAP and MS Graph derive threading from the headers and ignore it |

Omit `subject` and EmailEngine derives it from the referenced message, prefixing `Re:` for a reply or `Fwd:` for a forward unless the subject already starts with one. Omit `to` and the recipients are derived as well.

See [Replies and Forwards](/docs/sending/replies-forwards) for the full behavior.

### Webhooks

The Submit API reports delivery through these events. Each has a reference page with the full payload:

| Event | When |
|-------|------|
| [`messageSent`](/docs/webhooks/messagesent) | The receiving server accepted the message |
| [`messageDeliveryError`](/docs/webhooks/messagedeliveryerror) | An attempt failed and will be retried |
| [`messageFailed`](/docs/webhooks/messagefailed) | Every attempt failed and the message was dropped from the queue |
| [`messageBounce`](/docs/webhooks/messagebounce) | A bounce for the message arrived in the mailbox later |

A `messageSent` payload carries the `queueId` and `messageId` from the submit response, so the two can be matched:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user@example.com",
  "date": "2025-01-15T10:30:05.000Z",
  "event": "messageSent",
  "data": {
    "messageId": "<a2184d08-a470-fec6-a493-fa211a3756e9@example.com>",
    "response": "250 2.0.0 Ok: queued as 9441D8220E",
    "queueId": "d41f0423195f271f",
    "envelope": {
      "from": "user@example.com",
      "to": ["jane@example.com"]
    }
  }
}
```

---

## Scheduling Emails

Add `sendAt` with an ISO 8601 timestamp. The message is held in the outbox until then:

```bash
curl -X POST "https://emailengine.example.com/v1/account/user%40example.com/submit" \
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

A scheduled message can be [cancelled](#cancel-outbox-message) with its `queueId` at any point before it is sent. In a mail merge, each entry can carry its own `sendAt`, which overrides the message-level value.

---

## Outbox Management

The outbox contains queued messages waiting to be sent, including scheduled sends and entries that have failed and are waiting for a retry. Use these endpoints to view and manage them.

### List Outbox

**Endpoint:** `GET /v1/outbox`

[Detailed API reference](/docs/api/get-v-1-outbox)

Lists queued messages across all accounts.

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `page` | number | `0` | Page number, zero-indexed |
| `pageSize` | number | `20` | Entries per page, up to `1000` |

```bash
curl "https://emailengine.example.com/v1/outbox?pageSize=50" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

The response is paginated, with `total`, `page`, `pages`, and a `messages` array of entries in the shape shown below.

### Get Outbox Message

**Endpoint:** `GET /v1/outbox/{queueId}`

[Detailed API reference](/docs/api/get-v-1-outbox-queueid)

```bash
curl "https://emailengine.example.com/v1/outbox/d41f0423195f271f" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

**Response:**

```json
{
  "queueId": "d41f0423195f271f",
  "account": "user123",
  "source": "api",
  "messageId": "<a2184d08-a470-fec6-a493-fa211a3756e9@example.com>",
  "envelope": {
    "from": "sender@example.com",
    "to": ["client@example.com"]
  },
  "subject": "Newsletter January 2025",
  "gateway": null,
  "useStructuredFormat": false,
  "created": "2025-01-15T10:00:00.000Z",
  "scheduled": "2025-01-20T09:00:00.000Z",
  "nextAttempt": "2025-01-20T09:00:00.000Z",
  "attemptsMade": 0,
  "attempts": 10,
  "progress": {
    "status": "queued"
  }
}
```

`attemptsMade` against `attempts` tells you how much of the retry budget is left, and `nextAttempt` when the next one is due; it is `false` once every attempt has been used. `source` says how the message entered the queue: `api`, `smtp` (the [SMTP interface](/docs/sending/smtp-interface)), `ui`, or `test`. `proxy`, `localAddress`, and `idempotencyKey` appear when they were set on the submission. A message whose last attempt failed reports `progress.status` as `error` and carries the failure in `progress.error` (`message`, `code`, `statusCode`).

### Cancel Outbox Message

**Endpoint:** `DELETE /v1/outbox/{queueId}`

[Detailed API reference](/docs/api/delete-v-1-outbox-queueid)

```bash
curl -X DELETE "https://emailengine.example.com/v1/outbox/d41f0423195f271f" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

The response reports whether the entry was removed:

```json
{ "deleted": true }
```

Any entry that is not being delivered at that moment can be removed: one waiting for its scheduled time, one waiting for a retry, and a failed entry that the outbox retains after its last attempt. An entry a submit worker is delivering right now cannot be removed, and neither can a `queueId` that does not exist; both come back as `deleted: false` rather than an error.

### Outbox Message States

`progress.status` is one of:

| Status | Description |
|--------|-------------|
| `queued` | Accepted and waiting for a delivery attempt |
| `processing` | A submit worker has picked the message up |
| `smtp-starting` | Opening the SMTP connection to the destination |
| `smtp-completed` | The SMTP transaction finished and the message was handed over |
| `submitted` | Delivered to the receiving server. Later bounces arrive as webhooks |
| `error` | Delivery failed. `progress.error` says why; the entry is retried while attempts remain |

`submitted` means the receiving server accepted the message, not that it reached the recipient's inbox. A message can still bounce afterwards, which arrives as a [`messageBounce`](/docs/webhooks/messagebounce) webhook rather than a change to this status.

## Message Format

### Recipients

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

### Content

```json
{
  "subject": "Welcome!",
  "text": "Plain text version",
  "html": "<h1>HTML version</h1><p>With formatting</p>"
}
```

Provide both `text` and `html`. EmailEngine does not generate a plain text part from HTML, and some clients and filters expect one.

### Attachments

**Base64 encoded:**

```json
{
  "attachments": [
    {
      "filename": "document.pdf",
      "content": "JVBERi0xLjQKJeLjz9MKMy",
      "encoding": "base64",
      "contentType": "application/pdf"
    }
  ]
}
```

**Inline images**, referenced from the HTML by `cid`:

```json
{
  "html": "<img src=\"cid:logo\">",
  "attachments": [
    {
      "filename": "logo.png",
      "content": "iVBORw0KGgoAAAANSU",
      "encoding": "base64",
      "contentType": "image/png",
      "cid": "logo"
    }
  ]
}
```

**Referencing an existing attachment**, by the ID from the referenced message's attachment list:

```json
{
  "attachments": [
    {
      "filename": "contract.pdf",
      "reference": "AAAAAQAACnAcde"
    }
  ]
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

Using `mailMerge` rejects the root-level `to`, `cc`, `bcc`, `envelope`, `raw`, and `render` keys, since each entry carries its own, and `listId` is only accepted when `mailMerge` is present. The response returns a `mailMerge` array with one entry per recipient instead of a single `messageId`: each entry has `success`, `to`, `messageId`, `queueId`, and `sendAt`, or `error`, `code`, and `statusCode` when queueing that recipient failed, or `skipped` when the recipient was left out, for example after unsubscribing from the `listId`.

See [Mail Merge](/docs/sending/mail-merge) for personalization, batching, and how failures for one recipient are reported.

:::warning Sending limits still apply
Mail merge submits through the account's own mail server, which enforces its own rate and volume limits. Gmail and Microsoft 365 in particular will start deferring or rejecting mail well before a bulk campaign finishes. For genuine bulk mail, route the messages through an [SMTP gateway](/docs/api/post-v-1-gateway) built for it, using the `gateway` field.
:::

## See Also

- [Basic sending](/docs/sending/basic-sending) - The same endpoint as a walkthrough
- [Replies and forwards](/docs/sending/replies-forwards) - The reference block in practice
- [Outbox queue](/docs/sending/outbox-queue) - What happens after the call returns
- [SMTP interface](/docs/sending/smtp-interface) - The same queue fed over SMTP, with `X-EE-*` headers standing in for the JSON fields
- [messageSent](/docs/webhooks/messagesent) and [messageFailed](/docs/webhooks/messagefailed) - How delivery is reported
