---
title: Webhook Events Reference
description: Complete reference of all webhook events, payload schemas, and field specifications
sidebar_position: 1
---

# Webhook Events Reference

Complete reference for all webhook event types in EmailEngine. Each event includes detailed payload structure, field types, conditional fields, and provider-specific features.

## Event Structure

All webhook events follow this common structure:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "eventName",
  "account": "account-id",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {}
}
```

**Note:** The `eventId` is **NOT** included in the JSON payload. It's sent as the HTTP header `X-EE-Wh-Event-Id`.

### Universal Fields

These fields appear in every webhook event JSON payload:

| Field | Type | Description |
|-------|------|-------------|
| `serviceUrl` | string | Base URL of the EmailEngine instance that generated the event |
| `event` | string | Event type identifier (e.g., "messageNew", "messageSent") |
| `account` | string | Account identifier that triggered the event |
| `date` | string | ISO 8601 timestamp when event occurred |
| `data` | object | Event-specific payload data |

### Optional Universal Fields

| Field | Type | Description |
|-------|------|-------------|
| `path` | string | Mailbox path where the event occurred (message and mailbox events) |
| `specialUse` | string | Special-use flag of the folder (e.g., "\All", "\Inbox", "\Sent") |
| `_route` | object | Present when event is delivered through a Webhook Router, contains `_route.id` |

## Webhook Headers

EmailEngine includes diagnostic headers in every webhook HTTP request:

| Header | Type | Example | Description |
|--------|------|---------|-------------|
| `X-EE-Wh-Event-Id` | string | `af8435d9-ceee-4715-be71-08ac9d2dc04a` | **Unique event identifier (UUID).** Use for idempotency - all retries share the same ID. **This is the ONLY place eventId is available** - it's NOT in the JSON payload. |
| `X-EE-Wh-Id` | string | `907889` | Internal BullMQ job ID of the queued webhook entry |
| `X-EE-Wh-Attempts-Made` | string | `0` | Delivery attempt counter (starts at 0, increases with retries) |
| `X-EE-Wh-Queued-Time` | string | `5s` | Time the event spent in queue before delivery |
| `X-EE-Wh-Custom-Route` | string | `AAABiL8tBKsAAAAG` | Identifier of the custom webhook route (only present for webhook routes) |
| `X-EE-Wh-Signature` | string | `dGhpcyBpcyBh...` | HMAC-SHA256 signature (base64url) of the JSON body using EENGINE_SECRET |
| `Content-Type` | string | `application/json` | Always `application/json` |
| `User-Agent` | string | `emailengine/2.x.x (+https://emailengine.app)` | EmailEngine version and homepage |

### Webhook Signature Verification

The `X-EE-Wh-Signature` header contains an HMAC-SHA256 signature of the request body:

```javascript
const crypto = require('crypto');

function verifyWebhook(req, secret) {
  const signature = req.headers['x-ee-wh-signature'];
  const body = JSON.stringify(req.body);

  const hmac = crypto.createHmac('sha256', secret);
  hmac.update(body);
  const expected = hmac.digest('base64url');

  return signature === expected;
}

// Usage
if (!verifyWebhook(req, process.env.EENGINE_SECRET)) {
  return res.status(401).json({ error: 'Invalid signature' });
}
```

You can also configure custom headers via `webhooksCustomHeaders` in Settings or **Configuration → Webhooks**.

## Account Events

Events related to account connection and status changes.

### accountAdded

Triggered when a new account is registered in EmailEngine.

**Payload:**
```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "accountAdded",
  "account": "user@example.com",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {
    "account": "user@example.com"
  }
}
```

**Fields:**
- `data.account` (string) - Account identifier

To get the account's display name, email address and connection state after this event, query the account API endpoint (`GET /v1/account/:account`).

**Use Cases:**
- Send welcome notification
- Initialize account-specific resources
- Log account creation
- Start onboarding flow

---

### accountDeleted

Triggered when an account is removed from EmailEngine.

**Payload:**
```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "accountDeleted",
  "account": "user@example.com",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {
    "account": "user@example.com"
  }
}
```

**Fields:**
- `data.account` (string) - Deleted account identifier

**Use Cases:**
- Clean up account-related resources
- Remove from billing system
- Archive account data
- Send farewell notification

---

### accountInitialized

Triggered when an account successfully connects for the first time and completes initial mailbox synchronization.

**Payload:**
```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "accountInitialized",
  "account": "user@example.com",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {
    "initialized": true
  }
}
```

**Fields:**
- `data.initialized` (boolean) - Always `true`, indicates account has completed initialization

**Use Cases:**
- Notify user that account is ready
- Start background processing
- Enable account features
- Trigger initial data import

**Note:** To get full account details after initialization, query the account API endpoint (`GET /v1/account/:account`) which returns the account state, connection status, and other metadata.

---

### authenticationError

Triggered when account authentication fails.

**Payload:**
```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "authenticationError",
  "account": "user@example.com",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {
    "response": "Invalid credentials (Failure)",
    "serverResponseCode": "AUTHENTICATIONFAILED"
  }
}
```

**Fields:**
- `data.response` (string) - Human-readable error description (often the server's response text)
- `data.serverResponseCode` (string, optional) - Server or error response code (for example `AUTHENTICATIONFAILED`, or `OauthRenewError` for OAuth2 accounts)
- `data.tokenRequest` (object, optional) - OAuth2 token-renewal details when the failure occurred while refreshing an access token (includes `grant`, `provider`, `status`, `clientId`, `scopes`)

**Use Cases:**
- Prompt user to re-authenticate
- Revoke access tokens
- Send security alerts
- Log authentication failures

---

### authenticationSuccess

Triggered when account authenticates successfully.

**Payload:**
```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "authenticationSuccess",
  "account": "user@example.com",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {
    "user": "user@example.com"
  }
}
```

**Fields:**
- `data.user` (string) - The username (login) that authenticated successfully

**Use Cases:**
- Clear authentication error flags
- Resume account operations
- Log successful authentications
- Update account status

---

### connectError

Triggered when connection to the email server fails.

**Payload:**
```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "connectError",
  "account": "user@example.com",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {
    "response": "connect ECONNREFUSED 192.168.1.100:993",
    "serverResponseCode": "ECONNREFUSED"
  }
}
```

**Fields:**
- `data.response` (string) - Human-readable connection error description
- `data.serverResponseCode` (string, optional) - Connection error code (for example `ECONNREFUSED`, `ETIMEDOUT`, `ENOTFOUND`)

**Use Cases:**
- Monitor server availability
- Trigger network diagnostics
- Alert administrators
- Log connectivity issues

---

## Message Events

Events related to message operations and changes.

### messageNew

Triggered when a new message arrives in any mailbox. Also triggered when messages are moved, copied, or uploaded to folders.

**Note:** IMAP does not distinguish between incoming messages and messages inserted by other means.

**Payload:**
```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "messageNew",
  "account": "user@example.com",
  "path": "INBOX",
  "specialUse": "\\Inbox",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {
    "id": "AAAABAABNc",
    "uid": 12345,
    "path": "INBOX",
    "emailId": "abc123",
    "threadId": "thread_xyz",
    "date": "2025-01-15T10:25:00.000Z",
    "flags": ["\\Seen"],
    "labels": ["\\Important", "\\Inbox"],
    "category": "primary",
    "unseen": false,
    "flagged": false,
    "answered": false,
    "draft": false,
    "size": 8271,
    "subject": "Important Message",
    "from": {
      "name": "John Doe",
      "address": "john@example.com"
    },
    "sender": {
      "name": "John Doe",
      "address": "john@example.com"
    },
    "replyTo": [
      {
        "name": "John Doe",
        "address": "john@example.com"
      }
    ],
    "to": [
      {
        "name": "Jane Smith",
        "address": "jane@example.com"
      }
    ],
    "cc": [
      {
        "name": "Bob Johnson",
        "address": "bob@example.com"
      }
    ],
    "bcc": [
      {
        "name": "Alice Williams",
        "address": "alice@example.com"
      }
    ],
    "messageId": "<abc123@example.com>",
    "inReplyTo": "<previous@example.com>",
    "headers": {
      "list-id": "<mailinglist.example.com>",
      "x-custom-header": ["value1", "value2"]
    },
    "text": {
      "id": "text_123",
      "encodedSize": {
        "plain": 1535,
        "html": 1630
      },
      "plain": "Message content...",
      "html": "<p>Message content...</p>",
      "webSafe": true,
      "hasMore": false
    },
    "attachments": [
      {
        "id": "att_456",
        "contentType": "application/pdf",
        "filename": "document.pdf",
        "encodedSize": 52341,
        "embedded": false,
        "inline": false,
        "contentId": "<part1.abc@example.com>"
      }
    ],
    "messageSpecialUse": "\\Inbox",
    "seemsLikeNew": true,
    "isAutoReply": false,
    "isBounce": false,
    "isComplaint": false
  }
}
```

**Core Fields:**
- `data.id` (string) - EmailEngine message ID (use for API operations)
- `data.uid` (number) - IMAP UID
- `data.path` (string) - Mailbox path
- `data.emailId` (string, optional) - RFC 8474 Email ID (Gmail, modern IMAP servers)
- `data.threadId` (string, optional) - Thread/conversation ID (Gmail, modern IMAP servers)
- `data.date` (string) - Message Date header (ISO 8601)
- `data.flags` (array of strings) - IMAP flags (e.g., "\Seen", "\Flagged", "\Answered", "\Draft")
- `data.unseen` (boolean) - Message is unread (no \Seen flag)
- `data.flagged` (boolean) - Message is flagged
- `data.answered` (boolean) - Message has been replied to
- `data.draft` (boolean) - Message is a draft
- `data.size` (number) - Full RFC 822 message size in bytes
- `data.subject` (string) - Email subject line
- `data.messageId` (string) - RFC 5322 Message-ID header
- `data.inReplyTo` (string, optional) - Message-ID of the message being replied to

**Address Fields:**

Each address object contains:
- `name` (string) - Display name
- `address` (string) - Email address

Fields:
- `data.from` (object) - Sender address
- `data.sender` (object, optional) - Actual sender (when different from From)
- `data.replyTo` (array of objects, optional) - Reply-To addresses
- `data.to` (array of objects) - Recipients
- `data.cc` (array of objects, optional) - CC recipients
- `data.bcc` (array of objects, optional) - BCC recipients

**Content Fields (Conditional):**

Included when **Configuration → Webhooks → Email Content** is enabled (`notifyText: true`):

- `data.text` (object, optional) - Message text content
  - `data.text.id` (string) - Text content identifier
  - `data.text.encodedSize` (object) - Size information
    - `data.text.encodedSize.plain` (number) - Plain text size in bytes
    - `data.text.encodedSize.html` (number) - HTML size in bytes
  - `data.text.plain` (string) - Plain text content (up to `notifyTextSize` limit)
  - `data.text.html` (string) - HTML content (up to `notifyTextSize` limit). Holds the [web-safe](/docs/receiving/web-safe-html) version when `notifyWebSafeHtml: true`
  - `data.text.webSafe` (boolean, optional) - `true` when the HTML above was processed for web display (when `notifyWebSafeHtml: true`)
  - `data.text.hasMore` (boolean) - Content was truncated

**Attachment Fields (Conditional):**

Included when **Configuration → Webhooks → Attachments** is enabled (`notifyAttachments: true`):

- `data.attachments` (array of objects, optional) - Attachment metadata
  - `id` (string) - Attachment identifier
  - `contentType` (string) - MIME type
  - `filename` (string) - Filename
  - `encodedSize` (number) - Size of the attachment as stored in the email (base64 encoded); the decoded file size is approximately 75% of this value
  - `embedded` (boolean, optional) - Is embedded image
  - `inline` (boolean, optional) - Is inline attachment
  - `encodedInMessage` (boolean, optional) - Whether the attachment belongs to an enclosed message/rfc822 part rather than the top-level message
  - `contentId` (string, optional) - Content-ID header value
  - `method` (string, optional) - Calendar method (REQUEST, REPLY, CANCEL, etc.) for iCalendar attachments

**Gmail-Specific Fields:**

- `data.labels` (array of strings, optional) - Gmail labels (e.g., "\Important", "\Inbox", "\Starred")
- `data.category` (string, optional) - Gmail category tab ("primary", "social", "promotions", "updates", "forums")
  - Requires **Configuration → Email Processing → Gmail Features → Detect Gmail Categories** enabled
- `data.messageSpecialUse` (string, optional) - Special-use flag that best classifies the message (prefer over top-level `specialUse`)

**Header Fields (Conditional):**

Included when headers are specified in `notifyHeaders` setting:

- `data.headers` (object, optional) - Requested email headers
  - Keys are lowercase header names
  - Values are strings or arrays for multi-value headers

**Metadata Fields:**

- `data.seemsLikeNew` (boolean, optional) - EmailEngine has no prior record of this message (~99% accuracy)
- `data.isAutoReply` (boolean, optional) - Message appears to be an auto-reply
- `data.isBounce` (boolean, optional) - Message appears to be a bounce
- `data.isComplaint` (boolean, optional) - Message appears to be an abuse complaint

**AI Feature Fields (Conditional):**

Included when AI features are enabled:

- `data.summary` (object, optional) - AI-generated analysis (when `generateEmailSummary: true`). Contains `summary` (text), `sentiment`, `shouldReply` (boolean), `events` (array), `actions` (array), plus generation metadata (`id`, `tokens`, `model`)
- `data.embeddings` (array of numbers, optional) - Vector embeddings (when `openAiGenerateEmbeddings: true`)
- `data.riskAssessment` (object, optional) - AI risk analysis
  - `data.riskAssessment.risk` (integer) - Risk score where a higher value indicates higher risk
  - `data.riskAssessment.assessment` (string) - Human-readable explanation of the risk score
  - `data.riskAssessment.id` (string, optional) - Generation identifier
  - `data.riskAssessment.tokens` (number, optional) - Tokens consumed generating the assessment

**Use Cases:**
- Real-time email notifications
- Auto-reply systems
- Email-to-ticket conversion
- Message classification
- Attachment processing
- Email analytics
- Spam filtering
- Thread management

**Example Integration:**
```javascript
app.post('/webhook', async (req, res) => {
  const event = req.body;
  const eventId = req.headers['x-ee-wh-event-id'];

  if (event.event === 'messageNew') {
    const { data } = event;

    // Check idempotency using header
    if (await isProcessed(eventId)) {
      return res.json({ success: true }); // Already handled
    }

    // Notify user
    await sendPushNotification({
      title: `New email from ${data.from.address}`,
      body: data.subject
    });

    // Process attachments
    if (data.attachments?.length > 0) {
      await processAttachments(data.id, data.attachments);
    }

    // Auto-classify
    if (data.subject?.includes('invoice')) {
      await moveToFolder(data.id, 'Invoices');
    }

    // Mark as processed
    await markProcessed(eventId);
    res.json({ success: true });
  }
});
```

---

### messageDeleted

Triggered when a message is deleted from a mailbox or moved to another folder.

**Payload:**
```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "messageDeleted",
  "account": "user@example.com",
  "path": "INBOX",
  "specialUse": "\\Inbox",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {
    "id": "AAAABAABNc",
    "uid": 12345
  }
}
```

**Fields:**
- `path` (string) - Mailbox path the message was deleted from (top-level field, alongside `account`)
- `specialUse` (string, optional) - Special-use flag of that mailbox (top-level field)
- `data.id` (string) - EmailEngine message ID
- `data.uid` (number) - IMAP UID (no longer valid)

For IMAP accounts the `data` object contains only `id` and `uid`. Gmail API and Microsoft Graph accounts include provider-specific fields (for example Gmail adds `threadId` and the last known `labels`). See [messageDeleted](/docs/webhooks/messagedeleted) for the per-provider payloads.

**Use Cases:**
- Sync deletions to local database
- Track deleted messages
- Compliance logging
- Undo deletion feature
- Analytics (deletion patterns)

---

### messageUpdated

Triggered when message flags or labels change (read/unread, flagged, etc.).

**Payload:**
```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "messageUpdated",
  "account": "user@example.com",
  "path": "INBOX",
  "specialUse": "\\Inbox",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {
    "id": "AAAABAABNc",
    "uid": 12345,
    "changes": {
      "flags": {
        "added": ["\\Seen"],
        "removed": [],
        "value": ["\\Seen", "\\Flagged"]
      },
      "labels": {
        "added": ["\\Important"],
        "removed": ["\\Inbox"],
        "value": ["\\Important", "\\Starred"]
      }
    }
  }
}
```

**Fields:**
- `path` (string) - Mailbox path (top-level field, alongside `account`)
- `specialUse` (string, optional) - Special-use flag of that mailbox (top-level field)
- `data.id` (string) - EmailEngine message ID
- `data.uid` (number) - IMAP UID
- `data.changes` (object) - Change details
  - `data.changes.flags` (object) - Flag changes
    - `data.changes.flags.added` (array of strings) - Newly added flags
    - `data.changes.flags.removed` (array of strings) - Removed flags
    - `data.changes.flags.value` (array of strings) - Current complete flag list
  - `data.changes.labels` (object, optional) - Label changes (Gmail)
    - `data.changes.labels.added` (array of strings) - Newly added labels
    - `data.changes.labels.removed` (array of strings) - Removed labels
    - `data.changes.labels.value` (array of strings) - Current complete label list

**Use Cases:**
- Sync read status across devices
- Track flag changes
- Update UI in real-time
- Analytics (read rates)
- Trigger workflows on status changes

---

### messageMissing

Triggered when a previously seen message is no longer found in the mailbox (may indicate external deletion or mailbox corruption).

**Payload:**
```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "messageMissing",
  "account": "user@example.com",
  "path": "INBOX",
  "specialUse": "\\Inbox",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {
    "id": "AAAABAABNc",
    "uid": 12345,
    "missingRetries": 5,
    "missingDelay": 12450
  }
}
```

**Fields:**
- `path` (string) - Mailbox path (top-level field, alongside `account`)
- `specialUse` (string, optional) - Special-use flag of that mailbox (top-level field)
- `data.id` (string) - EmailEngine message ID
- `data.uid` (number) - IMAP UID (no longer valid)
- `data.missingRetries` (number, optional) - Number of resynchronization attempts made before the message was reported missing
- `data.missingDelay` (number, optional) - Delay in milliseconds before the message was confirmed missing

**Use Cases:**
- Detect sync issues
- Alert on potential data loss
- Trigger resynchronization
- Debugging mailbox problems

---

## Mailbox Events

Events related to mailbox (folder) operations.

### mailboxNew

Triggered when a new mailbox (folder) is created.

**Payload:**
```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "mailboxNew",
  "account": "user@example.com",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {
    "path": "Projects/2025",
    "name": "2025",
    "specialUse": false,
    "uidValidity": "1697551353"
  }
}
```

**Fields:**
- `data.path` (string) - Full mailbox path
- `data.name` (string) - Mailbox name (last component of path)
- `data.specialUse` (string or `false`) - Special-use flag (e.g., "\Sent", "\Drafts", "\Trash"), or `false` when the folder has no special use
- `data.uidValidity` (string) - The folder's UIDVALIDITY value (sent as a string)

**Use Cases:**
- Sync folder structure
- Update folder lists
- Track folder organization
- Folder-based automation rules

---

### mailboxDeleted

Triggered when a mailbox (folder) is deleted.

**Payload:**
```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "mailboxDeleted",
  "account": "user@example.com",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {
    "path": "Old/Archive",
    "name": "Archive",
    "specialUse": false
  }
}
```

**Fields:**
- `data.path` (string) - Deleted mailbox path
- `data.name` (string) - Mailbox name
- `data.specialUse` (string or `false`) - Special-use flag, or `false` when the folder has no special use

**Use Cases:**
- Remove folder from UI
- Archive folder contents
- Cleanup folder-based rules

---

### mailboxReset

Triggered when the message IDs stored for a folder stop being usable, either because the server reports a different UIDVALIDITY (`reason: "uidValidityChange"`) or because EmailEngine's own index for the folder had to be rebuilt (`reason: "syncStateLost"`).

**Important:** All previous UIDs for this folder are now invalid. You should refetch all messages.

**Payload:**
```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "mailboxReset",
  "account": "user@example.com",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {
    "path": "INBOX",
    "name": "INBOX",
    "specialUse": "\\Inbox",
    "uidValidity": "1234567899",
    "prevUidValidity": "1234567890",
    "reason": "uidValidityChange"
  }
}
```

**Fields:**
- `data.path` (string) - Mailbox path that was reset
- `data.name` (string) - Mailbox name (last component of path)
- `data.specialUse` (string or `false`) - Special-use flag of the folder, or `false` when it has no special use
- `data.uidValidity` (string) - New UIDVALIDITY value (sent as a string)
- `data.prevUidValidity` (string) - Previous UIDVALIDITY value (sent as a string)
- `data.reason` (string) - `uidValidityChange` or `syncStateLost`

**Use Cases:**
- Invalidate cached message UIDs
- Trigger full mailbox resynchronization
- Update local database
- Clear message references

---

## Sending Events

Events related to sending emails.

### messageSent

Triggered when a message is successfully accepted by the mail server (MTA).

**Note:** This indicates the server accepted the message, not that it was delivered to recipients. See `messageBounce` for delivery failures.

**Payload:**
```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "messageSent",
  "account": "user@example.com",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {
    "messageId": "<abc123@example.com>",
    "originalMessageId": "<original123@example.com>",
    "response": "250 2.0.0 Ok: queued as ABC123",
    "queueId": "queue_456",
    "envelope": {
      "from": "sender@example.com",
      "to": ["recipient@example.com"]
    }
  }
}
```

**Fields:**
- `data.messageId` (string) - Final Message-ID (may be rewritten by server)
- `data.originalMessageId` (string, optional) - Original Message-ID when the server rewrites it (Amazon SES, AWS WorkMail, Microsoft Graph)
- `data.response` (string) - The mail server's response to the message submission (typically the full SMTP `250` line)
- `data.queueId` (string) - Internal queue identifier
- `data.envelope` (object) - SMTP envelope
  - `data.envelope.from` (string) - Envelope sender (MAIL FROM)
  - `data.envelope.to` (array of strings) - Envelope recipients (RCPT TO)
- `data.networkRouting` (object, optional) - Present only when a custom local address or proxy was used; may contain `localAddress`, `proxy`, `name`, and `requestedLocalAddress`

There is no `to`, `subject`, or `gateway` field in this payload - the recipients are in `envelope.to`. To correlate the event with the original send, match on `queueId` (or `messageId`).

**Message-ID Rewriting:**

Some mail servers (Amazon SES, AWS WorkMail, Microsoft Graph) replace the Message-ID header. When this happens:
- `messageId` contains the final server-assigned ID
- `originalMessageId` contains your original ID

Always use `messageId` for tracking - it's the ID stored on the server.

**Use Cases:**
- Confirm delivery to user
- Update sent status in database
- Track email campaigns
- Delivery analytics
- Trigger follow-up actions

**Example Integration:**
```javascript
if (event.event === 'messageSent') {
  const { data } = event;

  // Update database
  await db.emails.updateOne(
    { queueId: data.queueId },
    { $set: { status: 'sent', messageId: data.messageId } }
  );

  // Notify user
  await notifyUser({
    message: `Message ${data.messageId} sent successfully`
  });

  // Analytics
  await trackEvent('email_sent', {
    account: event.account,
    recipients: data.envelope.to.length
  });
}
```

---

### messageDeliveryError

Triggered when email sending fails. EmailEngine retries automatically. You receive one webhook per failed attempt.

**Payload:**
```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "messageDeliveryError",
  "account": "user@example.com",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {
    "queueId": "queue_456",
    "envelope": {
      "from": "sender@example.com",
      "to": ["invalid@example.com"]
    },
    "messageId": "<abc123@example.com>",
    "error": "Recipient address rejected: User unknown",
    "errorCode": "EPROTOCOL",
    "smtpResponse": "550 5.1.1 <invalid@example.com>: Recipient address rejected: User unknown",
    "smtpResponseCode": 550,
    "smtpCommand": "RCPT TO",
    "job": {
      "id": "42",
      "attemptsMade": 1,
      "attempts": 10,
      "nextAttempt": "2025-01-15T10:07:45.465Z"
    }
  }
}
```

**Fields:**
- `data.queueId` (string) - Internal queue ID
- `data.envelope` (object) - SMTP envelope
  - `data.envelope.from` (string) - Envelope sender
  - `data.envelope.to` (array of strings) - Envelope recipients
- `data.messageId` (string) - Message-ID header
- `data.error` (string) - Error message
- `data.errorCode` (string) - Error code (e.g., "EPROTOCOL", "ECONNECTION", "EAUTH")
- `data.smtpResponse` (string, optional) - Full SMTP error response from the server
- `data.smtpResponseCode` (number, optional) - Numeric SMTP response code
- `data.smtpCommand` (string, optional) - SMTP command that triggered the error (for example `RCPT TO`)
- `data.networkRouting` (object, optional) - Present only when a custom local address or proxy was used
- `data.job` (object) - Queue job status
  - `data.job.id` (string) - BullMQ job identifier
  - `data.job.attemptsMade` (number) - Current attempt number
  - `data.job.attempts` (number) - Maximum attempts (default 10)
  - `data.job.nextAttempt` (string) - ISO 8601 timestamp of next retry

There is no `to`, `subject`, or `response` field in this payload - recipients are in `envelope.to` and the server response text is in `smtpResponse`.

**Common SMTP Error Codes:**
- `550 5.1.1` - User unknown / mailbox not found
- `550 5.7.1` - Relay denied / not authorized
- `552 5.2.2` - Mailbox full
- `554 5.7.1` - Message rejected (spam)

**Use Cases:**
- Alert user to sending failure
- Update failed status
- Custom retry logic
- Bounce handling
- Email validation

---

### messageFailed

Triggered when EmailEngine abandons delivery after all retry attempts fail. This is a permanent failure.

**Payload:**
```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "messageFailed",
  "account": "user@example.com",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {
    "messageId": "<abc123@example.com>",
    "queueId": "queue_456",
    "error": "Error: Invalid login: 535 5.7.8 Error: authentication failed",
    "networkRouting": {
      "localAddress": "192.168.1.100"
    }
  }
}
```

**Fields:**
- `data.messageId` (string) - Message-ID header
- `data.queueId` (string) - Internal queue ID
- `data.error` (string) - Final error message (first line of stack trace)
- `data.networkRouting` (object, optional) - Present only when a custom local address or proxy was used; may contain `localAddress`, `proxy`, `name`, and `requestedLocalAddress`

**Use Cases:**
- Notify sender of permanent failure
- Remove from send queue
- Log failed deliveries
- Update campaign statistics
- Trigger alternative delivery methods

---

### messageBounce

Triggered when a bounce (DSN - Delivery Status Notification) message is received for a sent email.

**Note:** Field coverage depends on the bounce format and what EmailEngine can parse. Different mail servers provide different levels of detail.

**Payload:**
```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "messageBounce",
  "account": "user@example.com",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {
    "bounceMessage": "AAAAAQAABWw",
    "recipient": "bounced@example.com",
    "action": "failed",
    "response": {
      "source": "smtp",
      "message": "550 5.1.1 <bounced@example.com>: Recipient address rejected: User unknown in relay recipient table",
      "status": "5.1.1",
      "category": "user_unknown",
      "recommendedAction": "remove"
    },
    "mta": "mx.example.com",
    "queueId": "BFC608226A",
    "messageId": "<abc123@example.com>",
    "messageHeaders": {
      "from": ["sender@example.com"],
      "to": ["bounced@example.com"],
      "subject": ["Newsletter"],
      "message-id": ["<abc123@example.com>"]
    }
  }
}
```

**Fields:**
- `data.bounceMessage` (string) - EmailEngine ID of the bounce message itself
- `data.recipient` (string, optional) - Bounced recipient address
- `data.action` (string) - DSN action ("failed", "delayed", "delivered", "relayed", "expanded")
- `data.response` (object, optional) - Parsed bounce response
  - `data.response.source` (string, optional) - Source of the diagnostic code (e.g., "smtp")
  - `data.response.message` (string, optional) - Bounce message
  - `data.response.status` (string, optional) - Enhanced status code (e.g., "5.1.1")
  - `data.response.category` (string, optional) - ML-classified bounce category (see table below)
  - `data.response.recommendedAction` (string, optional) - Suggested action: "remove", "retry", "review", "fix_configuration", "retry_different_ip", or "remove_content"
  - `data.response.blocklist` (object, optional) - Blocklist details when the bounce indicates a blocklist issue
    - `data.response.blocklist.name` (string) - Blocklist name (e.g., "Spamhaus ZEN")
    - `data.response.blocklist.type` (string) - "ip" or "domain"
  - `data.response.retryAfter` (number, optional) - Suggested retry delay in seconds (when timing found in error message)
- `data.mta` (string, optional) - Reporting Mail Transfer Agent hostname
- `data.queueId` (string, optional) - MTA queue identifier (Postfix-specific)
- `data.messageId` (string, optional) - Original Message-ID that bounced
- `data.messageHeaders` (object, optional) - Headers parsed from the original bounced message (keys are lowercased header names, values are arrays)

**ML Bounce Categories:**

EmailEngine uses machine learning to classify bounces into detailed categories:

| Category | Description | Action |
|----------|-------------|--------|
| `user_unknown` | Recipient doesn't exist | remove |
| `invalid_address` | Bad syntax or domain not found | remove |
| `mailbox_disabled` | Account suspended/disabled | remove |
| `mailbox_full` | Over quota | retry |
| `greylisting` | Temporary rejection | retry |
| `rate_limited` | Too many connections | retry |
| `server_error` | Timeout/connection failed | retry |
| `ip_blacklisted` | Sender IP on blocklist | retry_different_ip |
| `domain_blacklisted` | Sender domain on blocklist | fix_configuration |
| `auth_failure` | DMARC/SPF/DKIM failure | fix_configuration |
| `relay_denied` | Relaying not permitted | fix_configuration |
| `spam_blocked` | Detected as spam | review |
| `policy_blocked` | Local policy rejection | review |
| `virus_detected` | Infected content | remove_content |
| `geo_blocked` | Geographic rejection | retry_different_ip |
| `unknown` | Unclassified | review |

**Permanent vs temporary failures:**

The payload does not include a `bounceType` field. Use `action` (`failed` indicates a permanent failure, `delayed` a temporary one) together with `response.status` (a `5.x.x` enhanced status code is permanent, `4.x.x` is temporary) and `response.recommendedAction` to decide how to handle the bounce.

**Use Cases:**
- Remove permanently failed recipients (`action: "failed"`) from mailing lists
- Retry temporary failures based on `recommendedAction` and `retryAfter`
- Detect and respond to blocklist issues
- Email validation
- Delivery reporting
- Maintain sender reputation
- Compliance with anti-spam regulations

**Handling Multiple Recipients:**

If a message sent to multiple recipients bounces for several addresses, EmailEngine emits a separate `messageBounce` event for each recipient.

---

### messageComplaint

Triggered when an ARF (Abuse Reporting Format) feedback loop complaint is received. This indicates a recipient marked your email as spam.

**Note:** Field coverage depends on the reporting provider. Some providers omit headers like Message-ID.

**Payload:**
```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "messageComplaint",
  "account": "user@example.com",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {
    "complaintMessage": "AAAAAQAABvE",
    "arf": {
      "source": "Hotmail",
      "feedbackType": "abuse",
      "abuseType": "complaint",
      "originalRcptTo": ["recipient@hotmail.co.uk"],
      "arrivalDate": "2021-10-22T13:04:36.017Z",
      "sourceIp": "1.2.3.4",
      "userAgent": "Mozilla/5.0..."
    },
    "headers": {
      "messageId": "<abc123@example.com>",
      "from": "sender@example.com",
      "to": ["recipient@hotmail.co.uk"],
      "subject": "Newsletter",
      "date": "2021-10-22T16:04:33.000Z"
    }
  }
}
```

**Fields:**
- `data.complaintMessage` (string) - EmailEngine ID of the complaint message
- `data.arf` (object) - ARF feedback loop data
  - `data.arf.source` (string, optional) - Provider name (e.g., "Hotmail", "Yahoo")
  - `data.arf.feedbackType` (string) - Feedback type ("abuse", "fraud", "virus", "not-spam")
  - `data.arf.abuseType` (string, optional) - Specific abuse type
  - `data.arf.originalMailFrom` (string, optional) - Envelope sender of the original email
  - `data.arf.originalRcptTo` (array of strings, optional) - Original recipients
  - `data.arf.arrivalDate` (string, optional) - When complaint was generated (ISO 8601)
  - `data.arf.sourceIp` (string, optional) - IP address that sent the original email
  - `data.arf.userAgent` (string, optional) - User agent string
- `data.headers` (object, optional) - Original message headers (may be incomplete)
  - `data.headers.messageId` (string, optional) - Message-ID of complained message
  - `data.headers.from` (string, optional) - From address
  - `data.headers.to` (array of strings, optional) - Recipients
  - `data.headers.subject` (string, optional) - Subject line
  - `data.headers.date` (string, optional) - Date header

**Use Cases:**
- Remove complainers from mailing lists immediately
- Investigate spam complaints
- Improve email content
- Monitor sender reputation
- Compliance with anti-spam laws (CAN-SPAM, GDPR)
- Alert administrators

---

## Tracking Events

Events related to email tracking (opens and clicks).

**Note:** Tracking requires **Configuration → Email Processing → Email Tracking** to be enabled.

### trackOpen

Triggered when a tracking pixel embedded in an email is requested, indicating the recipient opened the email.

**Warning:** False positives are possible when:
- Webmail clients cache linked images
- Email clients pre-fetch images
- Security software scans emails

**Payload:**
```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "trackOpen",
  "account": "user@example.com",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {
    "messageId": "<abc123@example.com>",
    "remoteAddress": "203.0.113.45",
    "userAgent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.83 Safari/537.36"
  }
}
```

**Fields:**
- `data.messageId` (string) - Message-ID of the opened email
- `data.remoteAddress` (string) - IP address that requested the tracking pixel
- `data.userAgent` (string) - User-Agent header from the request

**Use Cases:**
- Track email open rates
- Engagement analytics
- Campaign performance
- Follow-up timing
- A/B testing

---

### trackClick

Triggered when a tracked link in an email is clicked.

**Warning:** False positives may occur when:
- Security software pre-fetches URLs
- Link scanners check URLs for safety
- Email clients preview links

**Payload:**
```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "trackClick",
  "account": "user@example.com",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {
    "messageId": "<abc123@example.com>",
    "url": "https://example.com/page",
    "remoteAddress": "203.0.113.45",
    "userAgent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.83 Safari/537.36"
  }
}
```

**Fields:**
- `data.messageId` (string) - Message-ID of the email containing the link
- `data.url` (string) - Original URL that was clicked
- `data.remoteAddress` (string) - IP address of the clicker
- `data.userAgent` (string) - User-Agent header from the request

**Use Cases:**
- Track click-through rates
- Identify popular content
- Engagement metrics
- Conversion tracking
- Link performance analysis

---

## List Management Events

Events related to email list subscriptions and unsubscriptions.

### listUnsubscribe

Triggered when a recipient clicks the List-Unsubscribe link or when an email client issues a one-click unsubscribe request (RFC 8058).

**Payload:**
```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "listUnsubscribe",
  "account": "user@example.com",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {
    "recipient": "recipient@example.com",
    "messageId": "<abc123@example.com>",
    "listId": "my-newsletter-list",
    "remoteAddress": "203.0.113.45",
    "userAgent": "Mozilla/5.0..."
  }
}
```

**Fields:**
- `data.recipient` (string) - Email address being unsubscribed
- `data.messageId` (string) - Message-ID of the email that contained the unsubscribe link
- `data.listId` (string, optional) - List identifier (from List-ID header)
- `data.remoteAddress` (string, optional) - IP address of the request
- `data.userAgent` (string, optional) - User-Agent header

**Use Cases:**
- Remove from mailing list immediately
- Send unsubscribe confirmation
- Update preference center
- Compliance (CAN-SPAM, GDPR)
- Analytics (unsubscribe rates)

---

### listSubscribe

Triggered when a recipient resubscribes to a list after previously unsubscribing.

**Payload:**
```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "listSubscribe",
  "account": "user@example.com",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {
    "recipient": "recipient@example.com",
    "listId": "my-newsletter-list",
    "remoteAddress": "203.0.113.45",
    "userAgent": "Mozilla/5.0..."
  }
}
```

**Fields:**
- `data.recipient` (string) - Email address being resubscribed
- `data.listId` (string, optional) - List identifier
- `data.remoteAddress` (string, optional) - IP address of the request
- `data.userAgent` (string, optional) - User-Agent header

**Use Cases:**
- Add to mailing list
- Send welcome-back email
- Update preference center
- Analytics (resubscribe rates)

---

## Export Events

Events related to bulk message export operations. See [Exporting Messages](/docs/receiving/exporting) for details on the export feature.

### exportCompleted

Triggered when a bulk message export completes successfully.

**Payload:**
```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "exportCompleted",
  "account": "user@example.com",
  "date": "2025-01-15T10:35:00.000Z",
  "data": {
    "exportId": "exp_abc123def456abc123def456",
    "folders": ["INBOX", "Sent"],
    "startDate": "2024-01-01T00:00:00.000Z",
    "endDate": "2024-12-31T23:59:59.000Z",
    "messagesExported": 450,
    "messagesSkipped": 5,
    "bytesWritten": 52428800,
    "duration": 15000,
    "expiresAt": "2025-01-16T10:30:00.000Z"
  }
}
```

**Fields:**
- `data.exportId` (string) - Export job identifier (28-character format: `exp_` + 24 hex digits)
- `data.folders` (array of strings) - Folders that were exported
- `data.startDate` (string) - Start of export date range (ISO 8601)
- `data.endDate` (string) - End of export date range (ISO 8601)
- `data.messagesExported` (number) - Total messages successfully exported
- `data.messagesSkipped` (number) - Messages skipped (deleted or inaccessible during export)
- `data.bytesWritten` (number) - Total bytes written to export file
- `data.duration` (number) - Export processing time in milliseconds
- `data.expiresAt` (string) - When the export file expires (ISO 8601)

**Use Cases:**
- Notify user that export is ready for download
- Trigger automated download workflows
- Log export completion for auditing
- Update export job status in your application

---

### exportFailed

Triggered when a bulk message export fails.

**Payload:**
```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "exportFailed",
  "account": "user@example.com",
  "date": "2025-01-15T10:30:00.000Z",
  "data": {
    "exportId": "exp_abc123def456abc123def456",
    "error": "Connection timeout",
    "errorCode": "ConnectionTimeout",
    "phase": "exporting",
    "messagesExported": 250,
    "messagesQueued": 1500
  }
}
```

**Fields:**
- `data.exportId` (string) - Export job identifier
- `data.error` (string) - Error message describing the failure
- `data.errorCode` (string, optional) - Error code for programmatic handling
- `data.phase` (string) - Phase when failure occurred ("indexing" or "exporting")
- `data.messagesExported` (number) - Messages exported before failure
- `data.messagesQueued` (number) - Total messages that were queued for export

**Use Cases:**
- Notify user of export failure
- Retry export operation
- Log failures for debugging
- Alert administrators of persistent failures

---

## Complete Event List

Quick reference table of all events:

| Event | Category | Trigger | Common Use |
|-------|----------|---------|------------|
| `accountAdded` | Account | Account registered | Onboarding |
| `accountDeleted` | Account | Account removed | Cleanup |
| `accountInitialized` | Account | First successful sync | Enable features |
| `authenticationError` | Account | Auth failed | Re-authenticate |
| `authenticationSuccess` | Account | Auth succeeded | Resume operations |
| `connectError` | Account | Connection failed | Monitor connectivity |
| `messageNew` | Message | New message received | Notifications |
| `messageDeleted` | Message | Message deleted | Sync deletions |
| `messageUpdated` | Message | Flags changed | Sync read status |
| `messageMissing` | Message | Message disappeared | Debugging |
| `messageSent` | Sending | Email accepted by MTA | Confirm delivery |
| `messageDeliveryError` | Sending | Send failed (retry) | Alert & retry |
| `messageFailed` | Sending | Send failed (permanent) | Log failure |
| `messageBounce` | Sending | Bounce received | List management |
| `messageComplaint` | Sending | FBL complaint | Remove from list |
| `trackOpen` | Tracking | Email opened | Analytics |
| `trackClick` | Tracking | Link clicked | Engagement |
| `listUnsubscribe` | List | Unsubscribed | Remove from list |
| `listSubscribe` | List | Resubscribed | Add to list |
| `mailboxNew` | Mailbox | Folder created | Sync folders |
| `mailboxDeleted` | Mailbox | Folder deleted | Update UI |
| `mailboxReset` | Mailbox | Stored UIDs no longer valid | Resync folder |
| `exportCompleted` | Export | Export job completed | Download export |
| `exportFailed` | Export | Export job failed | Retry/alert |

## Event Filtering

`webhookEvents` is an allowlist with no default. An event is delivered only if the list contains it or `"*"`, so leaving the list unset delivers nothing at all.

Subscribe to specific events when configuring webhooks:

```json
{
  "webhooks": "https://your-app.com/webhook",
  "webhookEvents": [
    "messageNew",
    "messageSent",
    "messageDeliveryError"
  ]
}
```

**Common Combinations:**

**Basic email monitoring:**
```json
["messageNew", "messageDeleted"]
```

**Sending tracking:**
```json
["messageSent", "messageDeliveryError", "messageFailed", "messageBounce"]
```

**Full monitoring:**
```json
[
  "messageNew",
  "messageDeleted",
  "messageUpdated",
  "messageSent",
  "messageDeliveryError",
  "messageFailed",
  "messageBounce",
  "messageComplaint",
  "trackOpen",
  "trackClick",
  "listUnsubscribe",
  "authenticationError",
  "connectError"
]
```

**Subscribe to all events:**
Use `["*"]` to receive all events.

## Conditional Fields Reference

Many fields only appear under specific conditions:

### Configuration-Dependent Fields

**Text Content** (`notifyText: true`):
- `data.text.plain`
- `data.text.html`
- `data.text.webSafe` (also requires `notifyWebSafeHtml: true`)
- `data.text.hasMore`

**Attachments** (`notifyAttachments: true`):
- `data.attachments[]`

**Custom Headers** (`notifyHeaders: ["Header-Name"]`):
- `data.headers`

**AI Features**:
- `data.summary` (requires `generateEmailSummary: true`)
- `data.embeddings` (requires `openAiGenerateEmbeddings: true`)

### Provider-Specific Fields

**Gmail**:
- `data.labels` - Gmail labels
- `data.category` - Inbox category (requires "Resolve Gmail categories" enabled)
- `data.emailId` - Gmail Email ID
- `data.threadId` - Gmail Thread ID

**Outlook/Modern IMAP**:
- `data.emailId` - RFC 8474 Email ID
- `data.threadId` - Thread ID

### Message-Dependent Fields

**Optional Headers**:
- `data.cc` - Only when CC recipients exist
- `data.bcc` - Only when BCC recipients exist
- `data.replyTo` - Only when Reply-To differs from From
- `data.sender` - Only when Sender differs from From
- `data.inReplyTo` - Only for reply messages

**Metadata**:
- `data.seemsLikeNew` - Only for messageNew events
- `data.isAutoReply` - Only when detected
- `data.isBounce` - Only when detected
- `data.isComplaint` - Only when detected

## Gmail-Specific Features

### Labels vs Flags

Gmail uses labels instead of traditional IMAP flags. EmailEngine provides both:

- `data.flags` - IMAP flags (limited set)
- `data.labels` - Full Gmail label list (recommended)

Common Gmail labels:
- `\Inbox` - In inbox
- `\Important` - Marked important
- `\Starred` - Starred
- `\Sent` - Sent mail
- `\Drafts` - Draft messages
- `\Trash` - In trash
- `UNREAD` - Unread (alternative to lack of \Seen flag)

### Category Tabs

Enable **Configuration → Email Processing → Gmail Features → Detect Gmail Categories** to get:

- `data.category` - One of: "primary", "social", "promotions", "updates", "forums"

### Special Use Folders

For Gmail, prefer `data.messageSpecialUse` over top-level `specialUse`:
- `specialUse` is usually "\All" (all mail)
- `data.messageSpecialUse` indicates the logical folder (e.g., "\Inbox")

### Email ID and Thread ID

Gmail provides stable identifiers:
- `data.emailId` - Unique message identifier (survives moves)
- `data.threadId` - Conversation identifier (groups related messages)

## Outlook-Specific Features

### Folder Structure

Outlook uses "/" as delimiter. Common folders:
- `INBOX` - Inbox
- `Sent Items` - Sent mail
- `Drafts` - Drafts
- `Deleted Items` - Trash
- `Junk Email` - Spam

### Shared Mailboxes

Events from shared mailboxes include the same fields. Use `account` to identify which mailbox.

### Categories

For **Microsoft Graph API** accounts, Outlook categories appear in the `data.labels` array (the same field used for Gmail labels):

```json
{
  "account": "outlook-user",
  "path": "Inbox",
  "data": {
    "id": "AAMkADU1...",
    "labels": ["Blue category", "Red category"],
    "subject": "Meeting notes"
  }
}
```

**Key differences from Gmail labels:**

| Feature | Gmail Labels | Outlook Categories |
|---------|--------------|-------------------|
| **Pre-creation required** | Yes - must exist in Gmail | No - auto-created when set |
| **Folder mapping** | Yes - labels map to folders | No - separate tag system |
| **Backend support** | Gmail IMAP, Gmail API | Microsoft Graph API only |
| **Color support** | No colors | Colors assigned by Outlook (not via API) |
| **Delete/rename via API** | Yes | No - use Outlook directly |

:::note IMAP Backend
When using Outlook with **IMAP/SMTP backend** (not Graph API), categories are not available. IMAP does not expose Outlook categories.
:::

:::note Category Limitations
EmailEngine works with category **names only**. Colors are assigned by Outlook when categories are created. EmailEngine can create new categories (by setting a non-existent name) but cannot delete, rename, or change colors - use Outlook directly for category management.
:::

## AI Features

When AI features are enabled (OpenAI integration):

### Email Summary

Enable: `generateEmailSummary: true`

Adds: `data.summary` (object) - AI-generated analysis of the email content

Example:
```json
{
  "data": {
    "subject": "Q4 Sales Report",
    "summary": {
      "id": "chatcmpl-7IzVIEp5UL3hdQ3aZJ8AHyrJrt3R0",
      "tokens": 2060,
      "model": "gpt-4",
      "sentiment": "neutral",
      "summary": "Sales increased 23% in Q4. Top performers: Product A (+45%), Product B (+12%). Request for Q1 strategy meeting.",
      "shouldReply": true,
      "events": [],
      "actions": [
        { "description": "Schedule Q1 strategy meeting", "dueDate": "2025-01-31" }
      ]
    }
  }
}
```

`data.summary` is an object, not a plain string. The human-readable text is in `data.summary.summary`.

### Vector Embeddings

Enable: `openAiGenerateEmbeddings: true`

Adds: `data.embeddings` (array of numbers) - Vector representation for semantic search

Example use:
```javascript
// Find similar emails
const similar = await findSimilar(message.embeddings, threshold: 0.8);
```

### Risk Assessment

Adds: `data.riskAssessment` (object) - AI-powered risk analysis

```json
{
  "data": {
    "riskAssessment": {
      "id": "chatcmpl-7IzVIEp5UL3hdQ3aZJ8AHyrJrt3R0",
      "tokens": 320,
      "risk": 7,
      "assessment": "The sender domain does not match the reply-to address and the message contains an urgent payment request."
    }
  }
}
```

`risk` is an integer where a higher value indicates higher risk; `assessment` is a human-readable explanation. There is no `score` or `reasons` field.

## Event Handling Best Practices

### Idempotency

Always handle duplicate events using the `X-EE-Wh-Event-Id` header:

```javascript
const processedEvents = new Set();

app.post('/webhook', async (req, res) => {
  const event = req.body;
  const eventId = req.headers['x-ee-wh-event-id'];

  // Check idempotency
  if (processedEvents.has(eventId)) {
    console.log('Duplicate event, skipping');
    return res.json({ success: true });
  }

  // Or use database
  const exists = await db.events.findOne({ eventId });
  if (exists) {
    return res.json({ success: true }); // Already processed
  }

  // Process event
  await processEvent(event);

  // Mark as processed
  await db.events.insertOne({ eventId, processed: true });
  processedEvents.add(eventId);

  res.json({ success: true });
});
```

### Error Handling

Return 2xx status for successful processing, 5xx to trigger retry:

```javascript
app.post('/webhook', async (req, res) => {
  try {
    await processWebhook(req.body);
    res.json({ success: true });
  } catch (error) {
    console.error('Webhook error:', error);
    // Return 5xx to trigger EmailEngine retry
    res.status(500).json({ error: error.message });
  }
});
```

### Async Processing

Queue webhooks for background processing to respond quickly:

```javascript
app.post('/webhook', async (req, res) => {
  // Queue immediately
  await queue.add('webhook', req.body);

  // Respond quickly (< 5 seconds recommended)
  res.json({ success: true });
});

// Process in background
queue.process('webhook', async (job) => {
  await processWebhook(job.data);
});
```

### Handling Conditional Fields

Check for field existence before accessing:

```javascript
function processMessageNew(event) {
  const { data } = event;

  // Safe text access
  if (data.text?.plain) {
    await indexText(data.text.plain);
  }

  // Safe attachment access
  if (data.attachments?.length > 0) {
    await processAttachments(data.attachments);
  }

  // Gmail-specific
  if (data.labels?.includes('\\Important')) {
    await flagAsImportant(data.id);
  }

  // AI features
  if (data.summary) {
    await storeSummary(data.id, data.summary);
  }
}
```

### Event-Specific Handling

Use switch statements for clarity:

```javascript
async function processWebhook(event) {
  switch (event.event) {
    case 'messageNew':
      return handleNewMessage(event.account, event.data);

    case 'messageSent':
      return handleMessageSent(event.account, event.data);

    case 'messageDeliveryError':
      return handleSendError(event.account, event.data);

    case 'messageBounce':
      return handleBounce(event.account, event.data);

    case 'messageComplaint':
      return handleComplaint(event.account, event.data);

    case 'trackOpen':
      return handleOpen(event.account, event.data);

    case 'trackClick':
      return handleClick(event.account, event.data);

    case 'listUnsubscribe':
      return handleUnsubscribe(event.account, event.data);

    case 'accountError':
      return handleAccountError(event.account, event.data);

    default:
      console.log('Unhandled event:', event.event);
  }
}
```

### Webhook Retry Handling

EmailEngine retries failed webhooks up to 10 times with exponential backoff:

```javascript
app.post('/webhook', async (req, res) => {
  const attemptNumber = parseInt(req.headers['x-ee-wh-attempts-made'] || '1');
  const eventId = req.headers['x-ee-wh-event-id'];

  if (attemptNumber > 1) {
    console.log(`Retry attempt ${attemptNumber} for event ${eventId}`);
  }

  // Process webhook...
  await processEvent(req.body);

  res.json({ success: true });
});
```

## Webhook Retry Mechanism

EmailEngine automatically retries failed webhook deliveries:

- **Maximum attempts**: 10
- **Backoff formula**: `delay = 5000ms × 2^(attempt - 1)`
- **Retry delays**:
  - Attempt 1: Immediate
  - Attempt 2: 5 seconds
  - Attempt 3: 10 seconds
  - Attempt 4: 20 seconds
  - Attempt 5: 40 seconds
  - Attempt 6: 80 seconds (1.3 minutes)
  - Attempt 7: 160 seconds (2.7 minutes)
  - Attempt 8: 320 seconds (5.3 minutes)
  - Attempt 9: 640 seconds (10.7 minutes)
  - Attempt 10: 1280 seconds (21.3 minutes)

After 10 failed attempts, the webhook is marked as undeliverable and moved to the Failed queue.

**Monitor webhooks**:
- Dashboard: **System → Queues → notify**
- Pending retries: **Delayed** section
- Undeliverable: **Failed** section

**Configure retention**:
- **Configuration → General → Queue Management**
- Set retention limits for completed/failed jobs

## Testing Events

### Using Webhook Tailing

Real-time webhook monitoring in the EmailEngine UI:

1. Navigate to **Configuration → Webhooks**
2. Click **Tail Webhooks**
3. Trigger events (send email, receive email, change flags)
4. See events in real-time with full payloads

### Using Webhook Testing Services

Test webhooks without implementing an endpoint:

- [Webhook.site](https://webhook.site) - Inspect payloads, headers, test responses
- [RequestBin](https://requestbin.com) - Create temporary endpoints
- [ngrok](https://ngrok.com) - Expose local server for testing

Example ngrok setup:
```bash
# Start local server
node server.js

# Expose via ngrok
ngrok http 3000

# Use ngrok URL in EmailEngine webhook settings
https://abc123.ngrok.io/webhook
```

### Testing Specific Events

**Test messageNew**: Send email to the account

**Test messageSent**: Use Send API

**Test messageDeliveryError**: Send to invalid address

**Test messageBounce**: Send to known bounce address

**Test trackOpen**: Enable tracking, send email, open it

**Test trackClick**: Enable tracking, send email with link, click it

**Test listUnsubscribe**: Add List-Unsubscribe header, click unsubscribe

## See Also

- [Webhooks overview](/docs/webhooks/overview) - Setup, delivery, retries, and debugging
- [Webhook routing](/docs/webhooks/webhook-routing) - Different events to different endpoints
- [Pre-processing functions](/docs/advanced/pre-processing) - Filtering and reshaping payloads
- [Webhooks API](/docs/api-reference/webhooks-api) - Managing routes programmatically
