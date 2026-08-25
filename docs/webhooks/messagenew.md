---
title: "messageNew"
sidebar_position: 3
description: "Webhook event triggered when a new email is detected in a mailbox folder"
---

# messageNew

The `messageNew` webhook event is triggered when EmailEngine detects a new email in a monitored mailbox folder. This is one of the most commonly used webhook events, enabling real-time processing of incoming emails.

## When This Event is Triggered

The `messageNew` event fires when:

- A new email arrives in a synced mailbox (INBOX, Sent, or any other monitored folder)
- An existing email is moved into a monitored folder from another folder
- An email is copied into a monitored folder
- Initial sync discovers emails not previously seen by EmailEngine

The event is triggered after EmailEngine has fetched and parsed the message metadata from the IMAP server.

:::note The initial sync does not replay the mailbox
A newly connected IMAP account only reports messages received after `notifyFrom`, which defaults to the moment the account was registered. The rest of the mailbox is indexed silently, and is reachable through the API without ever producing an event. Set `notifyFrom` to an earlier date, when [registering the account](/docs/api/post-v-1-account) or on a [flush](/docs/accounts/imap-indexers#changing-indexer-for-existing-account), to have older messages reported as well. Gmail API and MS Graph accounts ignore the field and never replay history.
:::

## Common Use Cases

- **Support ticket creation** - Automatically create tickets from incoming support emails
- **Lead capture** - Process inquiry emails and add contacts to your CRM
- **Order processing** - Parse order confirmation emails
- **AI analysis** - Feed incoming emails to language models for classification or summarization
- **Email archival** - Store emails in external databases or document management systems
- **Notification forwarding** - Send alerts via Slack, SMS, or other channels
- **Spam filtering** - Apply custom spam detection rules

## Payload Schema

### Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `serviceUrl` | string | No | The configured EmailEngine service URL |
| `account` | string | Yes | Account ID that received the message |
| `date` | string | Yes | ISO 8601 timestamp when the webhook was generated |
| `path` | string | Yes | Mailbox folder path (e.g., "INBOX", "Sent Mail") |
| `specialUse` | string | No | Special use flag of the folder (e.g., "\Inbox", "\Sent", "\Trash") |
| `event` | string | Yes | Event type, always "messageNew" for this event |
| `data` | object | Yes | Message data object (see below) |

**Note:** The unique event identifier is sent as the HTTP header `X-EE-Wh-Event-Id`, not in the JSON payload.

### Message Data Fields (`data` object)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | EmailEngine's unique message ID (base64url encoded) |
| `uid` | number | Yes | IMAP UID of the message within the folder |
| `path` | string | No | Mailbox folder path |
| `emailId` | string | No | IMAP EMAILID (if supported by server) |
| `threadId` | string | No | Thread identifier for conversation threading |
| `date` | string | Yes | Message date from headers (ISO 8601) |
| `flags` | array | No | Array of IMAP flags (e.g., ["\Seen", "\Flagged"]) |
| `labels` | array | No | Gmail labels (Gmail accounts only) |
| `unseen` | boolean | No | True if message has not been read |
| `flagged` | boolean | No | True if message is flagged/starred |
| `answered` | boolean | No | True if message has been replied to |
| `draft` | boolean | No | True if message is a draft |
| `size` | number | No | Message size in bytes |
| `subject` | string | No | Email subject line (decoded) |
| `from` | object | No | Sender address object |
| `replyTo` | array | No | Reply-To addresses |
| `sender` | object | No | Sender header (if different from From) |
| `to` | array | No | Recipient addresses |
| `cc` | array | No | CC recipient addresses |
| `bcc` | array | No | BCC recipient addresses (rarely available) |
| `messageId` | string | No | Message-ID header value |
| `inReplyTo` | string | No | In-Reply-To header for threading |
| `attachments` | array | No | List of attachment objects |
| `headers` | object | No | Selected email headers (if configured) |
| `text` | object | No | Text content object |
| `bounces` | array | No | Associated bounce information |
| `deliveryReport` | object | No | Parsed delivery status notification, set when the message is a "delivered" or "delayed" DSN (see below) |
| `isAutoReply` | boolean | No | True if detected as auto-reply |
| `seemsLikeNew` | boolean | No | True if message appears genuinely new (not moved/copied) |
| `category` | string | No | Message category (e.g., "primary", "social", "promotions") |
| `messageSpecialUse` | string | No | Special use tag (e.g., "\Inbox", "\Sent", "\Junk") |
| `calendarEvents` | array | No | Parsed calendar event data (if enabled) |
| `summary` | object | No | AI-generated summary (if OpenAI integration enabled) |
| `riskAssessment` | object | No | AI-generated risk assessment (if enabled) |
| `embeddings` | object | No | AI-generated embeddings (if enabled) |

### Delivery Report Structure

When an incoming message is a delivery status notification ([RFC 3464](https://www.rfc-editor.org/rfc/rfc3464)) reporting a successful delivery or a delay, EmailEngine parses its `message/delivery-status` part into `data.deliveryReport`. Failures are not reported here - those raise a separate [`messageBounce`](/docs/webhooks/messagebounce) event instead.

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

A notification that reports on several recipients is described one recipient at a time, so `action`, `status`, and `diagnosticCode` always belong together rather than being mixed across recipients.

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
| `id` | string | Attachment ID for downloading via API |
| `contentType` | string | MIME type (e.g., "application/pdf") |
| `encodedSize` | number | Size in bytes (transfer encoded) |
| `filename` | string | Filename (if provided) |
| `contentId` | string | Content-ID for inline attachments |
| `embedded` | boolean | True if part of multipart/related |
| `inline` | boolean | True if displayed inline |
| `method` | string | Calendar method (for .ics files) |
| `content` | string | Base64-encoded content (if configured) |

### Text Object Structure

The `text` object contains message body information:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Text part ID for fetching full content |
| `encodedSize` | object | Object with `plain` and `html` size values |
| `plain` | string | Plain text body (if `notifyText` enabled) |
| `html` | string | HTML body (if `notifyText` enabled). Holds the [web-safe](/docs/receiving/web-safe-html) version if `notifyWebSafeHtml` is enabled |
| `webSafe` | boolean | True if the `html` value was processed for web display |
| `hasMore` | boolean | True if text was truncated due to size limit |

### Calendar Event Object Structure

If `notifyCalendarEvents` is enabled and the message contains calendar data:

| Field | Type | Description |
|-------|------|-------------|
| `eventId` | string | Calendar event UID |
| `attachment` | string | Associated attachment ID |
| `method` | string | iCal method (REQUEST, CANCEL, etc.) |
| `summary` | string | Event title/summary |
| `description` | string | Event description |
| `timezone` | string | Event timezone |
| `startDate` | string | Event start (ISO 8601) |
| `endDate` | string | Event end (ISO 8601) |
| `organizer` | string | Event organizer |
| `filename` | string | Source filename |
| `contentType` | string | MIME type |
| `encoding` | string | Content encoding |
| `content` | string | Base64-encoded iCal data |

### AI Summary Object Structure

If OpenAI integration is enabled with `generateEmailSummary`:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | OpenAI completion ID |
| `tokens` | number | Tokens used |
| `model` | string | Model used (e.g., "gpt-4") |
| `sentiment` | string | Detected sentiment |
| `summary` | string | Generated summary text |
| `shouldReply` | boolean | AI suggestion if reply is needed |
| `events` | array | Extracted calendar events |
| `actions` | array | Extracted action items |

## Example Payload

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
    "messageSpecialUse": "\\Inbox",
    "threadId": "617dd693-d78b-48df-bc20-6330a5b1fa85"
  }
}
```

## Example with AI Summary

When OpenAI integration is enabled:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T07:15:00.000Z",
  "path": "INBOX",
  "specialUse": "\\Inbox",
  "event": "messageNew",
  "data": {
    "id": "AAAADAAABz0",
    "uid": 1850,
    "path": "INBOX",
    "date": "2025-10-17T07:14:30.000Z",
    "flags": [],
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
      },
      "plain": "Hi team,\n\nReminder that we have our weekly sync meeting this Friday at 3 PM.\n\nPlease come prepared with your status updates.\n\nThanks,\nSarah"
    },
    "seemsLikeNew": true,
    "messageSpecialUse": "\\Inbox",
    "summary": {
      "id": "chatcmpl-abc123xyz",
      "tokens": 580,
      "model": "gpt-4",
      "sentiment": "neutral",
      "summary": "Meeting reminder for weekly team sync on Friday at 3 PM. Team members should prepare status updates.",
      "shouldReply": false,
      "events": [
        {
          "description": "Weekly team sync meeting",
          "startTime": "2025-10-18T15:00:00"
        }
      ],
      "actions": [
        {
          "description": "Prepare status updates for meeting",
          "dueDate": "2025-10-18"
        }
      ]
    },
    "riskAssessment": {
      "risk": 1,
      "assessment": "Internal email from known sender. Authentication checks passed."
    },
    "threadId": "718ee794-c89c-49ef-bd21-7441a6b2fb96"
  }
}
```

## Configuration Options

Several settings affect the `messageNew` webhook payload:

### Text Content Options

| Setting | Description |
|---------|-------------|
| `notifyText` | Include message body text in webhook (default: false) |
| `notifyTextSize` | Maximum text size to include (bytes) |
| `notifyWebSafeHtml` | Replace the HTML body with a [web-safe](/docs/receiving/web-safe-html) version safe for web display. Requires `notifyText` |

### Header Options

| Setting | Description |
|---------|-------------|
| `notifyHeaders` | Array of header names to include (use `["*"]` for all) |

### Attachment Options

| Setting | Description |
|---------|-------------|
| `notifyAttachments` | Include attachment content in webhook |
| `notifyAttachmentSize` | Maximum attachment size to include |

### Calendar Options

| Setting | Description |
|---------|-------------|
| `notifyCalendarEvents` | Parse and include calendar event data |

### AI Options

| Setting | Description |
|---------|-------------|
| `generateEmailSummary` | Add `summary` and `riskAssessment` to incoming messages |
| `openAiGenerateEmbeddings` | Add `embeddings` to incoming messages |

Both need `openAiAPIKey` to be set as well. Configure them through the [Settings API](/docs/api/post-v-1-settings) or under **Configuration > AI Processing**. See [AI and ChatGPT Integration](/docs/integrations/ai-chatgpt) for what the generated fields contain.

## Handling the Event

### Basic Handler

```javascript
async function handleMessageNew(event) {
  const { account, data } = event;

  console.log(`New email for ${account}:`);
  console.log(`  From: ${data.from?.name} <${data.from?.address}>`);
  console.log(`  Subject: ${data.subject}`);
  console.log(`  Message ID: ${data.id}`);

  // Process based on your needs
  if (data.attachments?.length > 0) {
    console.log(`  Attachments: ${data.attachments.length}`);
    // Download attachments via API if needed
  }
}
```

### Fetching Full Message Content

If text content is not included in the webhook, fetch it via API:

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

The `seemsLikeNew` field helps distinguish genuinely new messages from moved/copied ones:

```javascript
async function handleMessageNew(event) {
  if (!event.data.seemsLikeNew) {
    // This message was likely moved from another folder
    console.log('Skipping moved/copied message');
    return;
  }

  // Process genuinely new message
  await processNewEmail(event.data);
}
```

### Filtering by Folder

```javascript
async function handleMessageNew(event) {
  // Only process INBOX messages
  if (event.data.messageSpecialUse !== '\Inbox') {
    return;
  }

  // Or check by path
  if (event.path !== 'INBOX') {
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
4. **Check `seemsLikeNew`** - Filter out moved/copied messages when appropriate
5. **Use message IDs** - Fetch additional data via API using `data.id` when needed
6. **Configure text inclusion** - Enable `notifyText` only if you need body content in webhooks
7. **Limit header exposure** - Only request headers you actually need via `notifyHeaders`

## Related Events

- [messageDeleted](/docs/webhooks/messagedeleted) - Triggered when a message is removed
- [messageUpdated](/docs/webhooks/messageupdated) - Triggered when flags/labels change
- [messageBounce](/docs/webhooks/messagebounce) - Triggered when a bounce is detected

## See Also

- [Webhooks Overview](/docs/webhooks/overview) - Complete webhook setup guide
- [Message Operations](/docs/receiving/message-operations) - Working with messages via API
- [Settings API](/docs/api/post-v-1-settings) - Configure webhook settings
