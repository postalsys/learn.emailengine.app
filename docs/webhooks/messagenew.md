---
title: "messageNew"
sidebar_position: 3
description: "Webhook event triggered when a new email is detected in a mailbox folder"
---

# messageNew

The `messageNew` webhook event is triggered when EmailEngine detects a new email in a monitored mailbox folder. This is the most commonly used webhook event, and the one whose payload is shaped by the most settings.

## When This Event is Triggered

The `messageNew` event fires when:

- A new email arrives in a synced mailbox (INBOX, Sent, or any other monitored folder)
- An existing email is moved or copied into a monitored folder from another folder
- On Gmail API and MS Graph accounts, a change notification reports a message EmailEngine has not seen before

The event is triggered after EmailEngine has fetched and parsed the message metadata from the mail server. On IMAP accounts a message that the server does not return on the first fetch is retried three times before a [`messageMissing`](/docs/webhooks/messagemissing) event is sent instead.

Nothing is sent for messages dated before the account's `notifyFrom` (IMAP), for accounts whose `webhookEvents` allowlist does not include `messageNew`, or, with `inboxNewOnly` set, for messages outside the Inbox. [Webhook routes](/docs/webhooks/webhook-routing) apply their own filters instead of the last two.

:::note The initial sync does not replay the mailbox
A newly connected IMAP account only reports messages received after `notifyFrom`, which defaults to the moment the account was registered. The rest of the mailbox is indexed silently, and is reachable through the API without ever producing an event. Set `notifyFrom` to an earlier date, when [registering the account](/docs/api/post-v-1-account) or on a [flush](/docs/accounts/imap-indexers#changing-indexer-for-existing-account), to have older messages reported as well. Gmail API and MS Graph accounts ignore the field and never replay history.
:::

By default the event is sent for every monitored folder. Set [`inboxNewOnly`](/docs/webhooks/overview#inbox-only-webhooks-inboxnewonly) to limit the default webhook to messages in the Inbox.

When the new message is itself a bounce or an ARF complaint, this event is sent first, with `isBounce` or `isComplaint` set, and a [`messageBounce`](/docs/webhooks/messagebounce) or [`messageComplaint`](/docs/webhooks/messagecomplaint) event follows it.

## Common Use Cases

- **Support ticket creation** - Create tickets from incoming support emails
- **Lead capture** - Process inquiry emails and add contacts to your CRM
- **Order processing** - Parse order confirmation emails
- **AI analysis** - Feed incoming emails to language models for classification or summarization
- **Email archival** - Store emails in external databases or document management systems
- **Notification forwarding** - Send alerts via Slack, SMS, or other channels

## Payload Schema

### Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `serviceUrl` | string or null | Yes | The configured EmailEngine service URL, `null` if not set |
| `account` | string | Yes | Account ID that received the message |
| `date` | string | Yes | ISO 8601 timestamp when the webhook was generated |
| `path` | string | Yes | Folder the message was found in. IMAP and MS Graph accounts report the folder path (for example `INBOX`). Gmail API accounts always report `\All`, because every message lives in All Mail; use `data.labels` or `data.messageSpecialUse` for its folder |
| `specialUse` | string | No | Special use flag of the folder, for example `\Inbox`, `\Sent` or `\Trash`. Gmail API accounts report `\All` |
| `event` | string | Yes | Always `messageNew` |
| `data` | object | Yes | Message data object (see below) |

The unique event identifier is sent as the HTTP header `X-EE-Wh-Event-Id`, not in the JSON payload.

### Message Data Fields (`data` object)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | EmailEngine message ID. Base64url-packed folder and UID for IMAP, the provider's message ID for Gmail API and MS Graph |
| `uid` | number | IMAP only | IMAP UID of the message within the folder |
| `path` | string | No | Folder path. IMAP accounts only. Gmail API and MS Graph accounts report the folder in the top-level `path` instead |
| `emailId` | string | No | Globally unique message ID: IMAP `EMAILID` where the server supports `OBJECTID`, the Gmail message ID, or the Graph message ID |
| `threadId` | string | No | Thread identifier, when the server or provider supplies one |
| `date` | string | Yes | Message date from headers (ISO 8601) |
| `flags` | array | No | IMAP flags (for example `["\\Seen", "\\Flagged"]`). `\Recent` is never included |
| `labels` | array | No | Gmail labels, on Gmail API accounts and on Gmail over IMAP. System labels use their IMAP special-use names (`\Inbox`, `\Sent`, `\Trash`, `\Drafts`, `\Junk`); other labels are reported by Gmail label ID on API accounts and by name over IMAP |
| `unseen` | boolean | No | `true` if the message has not been read. Absent when read |
| `flagged` | boolean | No | `true` if the message is flagged or starred. Absent otherwise |
| `answered` | boolean | No | `true` if the message has been replied to. Absent otherwise |
| `draft` | boolean | No | `true` if the message is a draft. Absent otherwise |
| `size` | number | No | Message size in bytes |
| `subject` | string | No | Decoded subject line |
| `from` | object | No | Sender address object |
| `replyTo` | array | No | Reply-To addresses |
| `sender` | object | No | Sender header (if different from From) |
| `to` | array | No | Recipient addresses |
| `cc` | array | No | CC recipient addresses |
| `bcc` | array | No | BCC recipient addresses (rarely available) |
| `messageId` | string | No | Message-ID header value |
| `inReplyTo` | string | No | In-Reply-To header for threading |
| `attachments` | array | No | List of attachment objects |
| `headers` | object | No | Selected email headers. Only present when `notifyHeaders` is configured (see below) |
| `text` | object | No | Text content object (see below) |
| `preview` | string | No | Short body preview supplied by the provider. Gmail API and MS Graph accounts only |
| `bounces` | array | No | Bounces previously recorded against this message's Message-ID. IMAP accounts only (see below) |
| `deliveryReport` | object | No | Parsed delivery status notification, set when the message is a "delivered" or "delayed" DSN (see below) |
| `isAutoReply` | boolean | No | `true` when the message looks like an automatic reply. The subject decides it when it begins with `Auto reply`, `Automatic reply`, `Automatic response`, `Out of Office`, `Out of the Office`, `OOF:` or `OOO:`, or with `Auto:` on a message that also has an `In-Reply-To` header. Otherwise an `Auto-Submitted: auto-replied`, a `Precedence: auto-reply`, or any `X-Auto-Response-Suppress`, `X-Autoresponder`, `X-Autorespond` or `X-Autoreply` header decides it |
| `isBounce` | boolean | No | `true` when the message was recognized as a bounce. A [`messageBounce`](/docs/webhooks/messagebounce) event follows this one |
| `isComplaint` | boolean | No | `true` when the message was recognized as an ARF complaint. A [`messageComplaint`](/docs/webhooks/messagecomplaint) event follows this one |
| `relatedMessageId` | string | No | Message-ID of the original message a bounce or complaint refers to. Set together with `isBounce` or `isComplaint` |
| `seemsLikeNew` | boolean | Yes | `true` if EmailEngine has not seen this message on the account before, so it is probably new rather than moved or copied. The check is an approximate one, over `emailId` where the server supplies one and the Message-ID otherwise. Always `false` for messages in the Sent folder. Always `true` on Gmail API accounts, where moves are reported as label changes instead |
| `category` | string | No | Gmail inbox tab. Gmail API accounts report `primary`, `social`, `promotions`, `updates` or `forums`, derived from the message labels. Gmail over IMAP resolves it with a server-side search when `resolveGmailCategories` is enabled, and can additionally report `reservations` and `purchases` |
| `messageSpecialUse` | string | No | Special use of the folder the message belongs to, for example `\Inbox`, `\Sent` or `\Junk`. Derived from the folder or, on Gmail, from the labels |
| `missingRetries` | number | No | IMAP only. How many extra fetch attempts were needed before the server returned the message |
| `missingDelay` | number | No | IMAP only. Milliseconds spent waiting between those attempts |
| `calendarEvents` | array | No | Parsed calendar event data. Only when `notifyCalendarEvents` is enabled |
| `summary` | object | No | AI-generated summary. Only when `generateEmailSummary` is enabled |
| `riskAssessment` | object | No | AI-generated risk assessment. Only when `generateEmailSummary` is enabled |
| `embeddings` | object | No | AI-generated embeddings. Only when `openAiGenerateEmbeddings` is enabled |

### Delivery Report Structure

When an incoming message is a delivery status notification ([RFC 3464](https://www.rfc-editor.org/rfc/rfc3464)) reporting a successful delivery or a delay, EmailEngine parses its `message/delivery-status` part into `data.deliveryReport`. Failures are not reported here - those raise a separate [`messageBounce`](/docs/webhooks/messagebounce) event instead. Only messages in the Inbox are checked.

Every field of the report is passed through with its name camelCased, so the exact set of keys depends on what the reporting server sent. A value that starts with an address type, such as `rfc822; user@example.com`, is split into a `label` and a `value`, and `Arrival-Date` is normalized to an ISO 8601 timestamp:

```json
{
  "deliveryReport": {
    "reportingMta": { "label": "dns", "value": "mx.example.com" },
    "arrivalDate": "2026-08-17T09:12:44.000Z",
    "finalRecipient": { "label": "rfc822", "value": "user@example.com" },
    "action": "delayed",
    "status": "4.4.1",
    "diagnosticCode": { "label": "smtp", "value": "451 4.4.1 Connection timed out" }
  }
}
```

A notification that reports on several recipients is described one recipient at a time, so `action`, `status`, and `diagnosticCode` always belong together rather than being mixed across recipients. This shape was introduced in v2.78.0; earlier versions reported only a fixed subset of the fields.

### Bounce List Structure

On IMAP accounts, when EmailEngine has previously processed a bounce that referred to this message's Message-ID, `data.bounces` lists what it recorded. Gmail API and MS Graph accounts do not carry this field. Each entry contains:

| Field | Type | Description |
|-------|------|-------------|
| `message` | string | EmailEngine message ID of the bounce notification |
| `recipient` | string | Address that bounced |
| `action` | string | Bounce action, typically `failed` |
| `response` | object | `message` and `status` from the bounce, when known |
| `date` | string | When the bounce was recorded (ISO 8601) |

### Address Object Structure

The `from`, `sender`, `replyTo`, `to`, `cc`, and `bcc` fields contain address objects:

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Display name (may be empty) |
| `address` | string | Email address |

### Attachment Object Structure

Each attachment in the `attachments` array contains:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Attachment ID for downloading via the API |
| `contentType` | string | MIME type (for example `application/pdf`) |
| `encodedSize` | number | Size in bytes as transferred (encoded) |
| `filename` | string | Filename (if provided) |
| `contentId` | string | Content-ID for inline attachments |
| `embedded` | boolean | `true` if part of a `multipart/related` group |
| `inline` | boolean | `true` if the part is marked for inline display |
| `method` | string | Calendar method (for `text/calendar` parts) |
| `content` | string | Base64-encoded content. Only when `notifyAttachments` is enabled, and only for attachments within `notifyAttachmentSize` |

### Text Object Structure

The `text` object describes the message body. Without `notifyText` it carries only `id` and `encodedSize`:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Text part ID for fetching the full content through the API |
| `encodedSize` | object | Object with `plain` and `html` size values in bytes |
| `plain` | string | Plain text body. Requires `notifyText` |
| `html` | string | HTML body. Requires `notifyText`. Holds the [web-safe](/docs/receiving/web-safe-html) version if `notifyWebSafeHtml` is enabled |
| `webSafe` | boolean | `true` if the `html` value was processed for web display |
| `hasMore` | boolean | `true` if `plain` or `html` was truncated to `notifyTextSize` |

### Calendar Event Object Structure

If `notifyCalendarEvents` is enabled and the message contains `text/calendar` or `application/ics` attachments, each distinct event UID appears once in `calendarEvents`. Empty values are omitted:

| Field | Type | Description |
|-------|------|-------------|
| `eventId` | string | Calendar event UID |
| `attachment` | string | ID of the attachment the event was parsed from |
| `method` | string | iCalendar method (`REQUEST`, `CANCEL`, and so on) |
| `summary` | string | Event title |
| `description` | string | Event description |
| `timezone` | string | Time zone ID from the embedded `VTIMEZONE` |
| `startDate` | string | Event start (ISO 8601) |
| `endDate` | string | Event end (ISO 8601) |
| `organizer` | string | Event organizer |
| `filename` | string | Attachment filename. Defaults to `invite.ics` for `REQUEST` and `CANCEL`, otherwise `event.ics` |
| `contentType` | string | MIME type of the attachment |
| `encoding` | string | Always `base64` |
| `content` | string | Base64-encoded iCalendar data |

### AI Fields

`summary`, `riskAssessment` and `embeddings` are produced by the OpenAI integration and their contents depend on the configured prompt. See [AI and ChatGPT Integration](/docs/integrations/ai-chatgpt#webhook-enhancement) for the fields each of them carries.

## Example Payload

An IMAP account with `notifyText` and `notifyHeaders` enabled:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T06:42:25.056Z",
  "path": "INBOX",
  "specialUse": "\\Inbox",
  "event": "messageNew",
  "data": {
    "id": "AAAADAAABy4",
    "uid": 1838,
    "path": "INBOX",
    "date": "2025-10-17T06:42:07.000Z",
    "flags": [],
    "unseen": true,
    "size": 549725,
    "subject": "Quarterly Report Review",
    "from": {
      "name": "John Smith",
      "address": "john.smith@example.com"
    },
    "replyTo": [
      {
        "name": "John Smith",
        "address": "john.smith@example.com"
      }
    ],
    "sender": {
      "name": "John Smith",
      "address": "john.smith@example.com"
    },
    "to": [
      {
        "name": "Jane Doe",
        "address": "jane.doe@company.com"
      }
    ],
    "attachments": [
      {
        "id": "AAAADAAABy4y",
        "contentType": "application/pdf",
        "encodedSize": 546048,
        "filename": "Q3-Report.pdf",
        "embedded": false,
        "inline": false
      }
    ],
    "messageId": "<abc123@mail.example.com>",
    "headers": {
      "return-path": ["<john.smith@example.com>"],
      "delivered-to": ["jane.doe@company.com"],
      "mime-version": ["1.0"],
      "from": ["John Smith <john.smith@example.com>"],
      "date": ["Thu, 17 Oct 2025 09:42:07 +0300"],
      "message-id": ["<abc123@mail.example.com>"],
      "subject": ["Quarterly Report Review"],
      "to": ["Jane Doe <jane.doe@company.com>"],
      "content-type": ["multipart/mixed; boundary=\"----=_Part_123\""]
    },
    "text": {
      "id": "AAAADAAABy6TkaMxLjGRozEuMpA",
      "encodedSize": {
        "plain": 1250,
        "html": 2840
      },
      "plain": "Hi Jane,\n\nPlease find attached the Q3 report for your review.\n\nBest regards,\nJohn",
      "html": "<div>Hi Jane,<br><br>Please find attached the Q3 report for your review.<br><br>Best regards,<br>John</div>"
    },
    "seemsLikeNew": true,
    "messageSpecialUse": "\\Inbox"
  }
}
```

## Example: Gmail API Account

On a Gmail API account `path` and `specialUse` are `\All`, the folder is expressed through `labels` and `messageSpecialUse`, and `category` names the inbox tab:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "gmail-user",
  "date": "2025-10-17T07:15:00.000Z",
  "path": "\\All",
  "specialUse": "\\All",
  "event": "messageNew",
  "data": {
    "id": "18b5c7d8e9f01234",
    "emailId": "18b5c7d8e9f01234",
    "threadId": "18b5c7d8e9f01234",
    "date": "2025-10-17T07:14:30.000Z",
    "flags": [],
    "labels": ["\\Inbox"],
    "category": "primary",
    "unseen": true,
    "size": 8500,
    "subject": "Team Meeting Reminder - Friday 3 PM",
    "from": {
      "name": "Sarah Johnson",
      "address": "sarah@company.com"
    },
    "to": [
      {
        "name": "Team",
        "address": "team@company.com"
      }
    ],
    "messageId": "<meeting-reminder@company.com>",
    "text": {
      "id": "AAAADAAABz0TkaMx",
      "encodedSize": {
        "plain": 450,
        "html": 920
      }
    },
    "preview": "Hi team, reminder that we have our weekly sync meeting this Friday at 3 PM.",
    "seemsLikeNew": true,
    "messageSpecialUse": "\\Inbox"
  }
}
```

## Configuration Options

Several settings shape the `messageNew` payload. All of them are set through the [Settings API](/docs/api/post-v-1-settings) or under **Configuration > Webhooks**.

### Text Content Options

| Setting | Description |
|---------|-------------|
| `notifyText` | Include `text.plain` and `text.html`. Set to `true` at first start, so it is on unless switched off |
| `notifyTextSize` | Maximum size in bytes of each text value. Set to `2097152` (2 MB) at first start. Longer content is truncated and `text.hasMore` is set |
| `notifyWebSafeHtml` | Replace `text.html` with a sanitized [web-safe](/docs/receiving/web-safe-html) rendering and set `text.webSafe`. Quoted thread history is folded into a collapsed block, and the plain text body is rendered when the message has no HTML part. `cid:` references to inline images are left as they are in the webhook payload. Requires `notifyText` |

### Header Options

| Setting | Description |
|---------|-------------|
| `notifyHeaders` | Header names to include in `data.headers`. Use `["*"]` for all headers. Names are compared lowercase, so pass them in lowercase when setting this through the API (`["list-id", "x-priority"]`); the admin UI lowercases them for you |

Regardless of this setting, EmailEngine always fetches the headers it needs for its own auto-reply and bounce detection. They appear in the payload only when `notifyHeaders` names them.

### Attachment Options

| Setting | Description |
|---------|-------------|
| `notifyAttachments` | Include attachment content, base64-encoded, in `attachments[].content` |
| `notifyAttachmentSize` | Maximum size in bytes of an attachment to include. Larger attachments keep their metadata but no content |

### Calendar Options

| Setting | Description |
|---------|-------------|
| `notifyCalendarEvents` | Parse `text/calendar` and `application/ics` attachments into `calendarEvents` |

### AI Options

| Setting | Description |
|---------|-------------|
| `generateEmailSummary` | Add `summary` and `riskAssessment` to Inbox messages |
| `openAiGenerateEmbeddings` | Add `embeddings` to Inbox messages |

Both need `openAiAPIKey` to be set as well, and both apply only to messages in the Inbox. See [AI and ChatGPT Integration](/docs/integrations/ai-chatgpt) for what the generated fields contain.

## Handling the Event

### Basic Handler

```javascript
async function handleMessageNew(event) {
  const { account, data } = event;

  console.log(`New email for ${account}:`);
  console.log(`  From: ${data.from?.name} <${data.from?.address}>`);
  console.log(`  Subject: ${data.subject}`);
  console.log(`  Message ID: ${data.id}`);

  if (data.attachments?.length > 0) {
    console.log(`  Attachments: ${data.attachments.length}`);
  }
}
```

### Fetching Full Message Content

If text content is not included in the webhook, fetch it via the API:

```javascript
async function getMessageContent(account, messageId) {
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${account}/message/${messageId}`,
    {
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN'
      }
    }
  );
  return response.json();
}
```

### Downloading Attachments

```javascript
async function downloadAttachment(account, attachmentId) {
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${account}/attachment/${attachmentId}`,
    {
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN'
      }
    }
  );
  return response.arrayBuffer();
}
```

## Filtering New Messages

### Using `seemsLikeNew`

The `seemsLikeNew` field helps distinguish genuinely new messages from moved or copied ones:

```javascript
async function handleMessageNew(event) {
  if (!event.data.seemsLikeNew) {
    console.log('Skipping moved or copied message');
    return;
  }

  await processNewEmail(event.data);
}
```

### Filtering by Folder

`messageSpecialUse` works for every account type, including Gmail API accounts where `path` is always `\All`:

```javascript
async function handleMessageNew(event) {
  if (event.data.messageSpecialUse !== '\\Inbox') {
    return;
  }

  await processInboxEmail(event.data);
}
```

### Filtering Auto-Replies

```javascript
async function handleMessageNew(event) {
  if (event.data.isAutoReply) {
    console.log('Skipping auto-reply');
    return;
  }

  await processEmail(event.data);
}
```

## Best Practices

1. **Respond quickly** - Return a 2xx status before the delivery times out (30 seconds by default) to prevent retries
2. **Process asynchronously** - Queue events for processing after acknowledging receipt
3. **Handle duplicates** - Deduplicate on the `X-EE-Wh-Event-Id` request header, which is stable across retries of the same delivery
4. **Check `seemsLikeNew`** - Filter out moved and copied messages when appropriate
5. **Use message IDs** - Fetch additional data via the API using `data.id` when needed
6. **Switch `notifyText` off if you do not need bodies** - Body content makes payloads large, and `notifyTextSize` bounds them
7. **Limit header exposure** - Only request the headers you need via `notifyHeaders`

## Related Events

- [messageDeleted](/docs/webhooks/messagedeleted) - Triggered when a message is removed
- [messageUpdated](/docs/webhooks/messageupdated) - Triggered when flags or labels change
- [messageBounce](/docs/webhooks/messagebounce) - Triggered when a bounce is detected
- [messageComplaint](/docs/webhooks/messagecomplaint) - Triggered when an ARF complaint is detected

## See Also

- [Webhooks Overview](/docs/webhooks/overview) - Delivery, retries, headers and signing
- [Webhook events reference](/docs/reference/webhook-events) - Every event and its payload in one place
- [Message Operations](/docs/receiving/message-operations) - Fetching the full message through the API
- [Web-safe HTML](/docs/receiving/web-safe-html) - What `notifyWebSafeHtml` does to the HTML body
- [Settings API](/docs/api/post-v-1-settings) - Setting the `notify*` options
