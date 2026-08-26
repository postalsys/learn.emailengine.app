---
title: Messages API
description: API endpoints for message operations - list, search, read, update, and delete emails
sidebar_position: 4
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Messages API

The Messages API reaches the mail in any connected account: list, read, search, update, move, and delete.

## Overview

The Messages API allows you to:

- **List messages** from any mailbox with filtering and pagination
- **Read message** content, headers, and metadata
- **Search messages** using advanced query syntax
- **Update message flags** (read/unread, flagged, etc.)
- **Move messages** between mailboxes
- **Delete messages** permanently or move to trash
- **Download attachments** and message source

### Message Object Structure

```json
{
  "id": "AAAABAABNc",
  "uid": 12345,
  "emailId": "1234567890abcdef",
  "threadId": "thread_abc123",
  "path": "INBOX",
  "date": "2025-01-15T10:30:00.000Z",
  "flags": ["\\Seen"],
  "unseen": false,
  "flagged": false,
  "answered": false,
  "draft": false,
  "size": 15234,
  "subject": "Meeting Tomorrow",
  "from": {
    "name": "John Doe",
    "address": "john@example.com"
  },
  "to": [
    {
      "name": "Jane Smith",
      "address": "jane@example.com"
    }
  ],
  "messageId": "<abc123@example.com>",
  "inReplyTo": "<xyz789@example.com>",
  "text": {
    "id": "AAAAAQAACnAWkeM",
    "encodedSize": {
      "plain": 1013,
      "html": 3487
    }
  },
  "attachments": [
    {
      "id": "AAAAAQAACnAy",
      "contentType": "application/pdf",
      "encodedSize": 52341,
      "embedded": false,
      "inline": false,
      "filename": "document.pdf"
    }
  ]
}
```

### Message IDs Explained

EmailEngine uses several types of IDs:

- **`id`**: EmailEngine's internal message ID (used in API calls)
- **`uid`**: IMAP UID (server-specific, unique within mailbox)
- **`emailId`**: RFC 8474 Email ID (unique across mailboxes)
- **`threadId`**: RFC 8474 Thread ID (groups related messages)
- **`messageId`**: RFC 5322 Message-ID header

[Learn more about IDs →](/docs/advanced/ids-explained)

## Common Operations

### 1. List Messages

Retrieve messages from a mailbox with filtering and pagination.

**Endpoint:** `GET /v1/account/:account/messages`

**Path Parameters:**

| Parameter | Description |
|-----------|-------------|
| `account` | Account identifier |

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `path` | string | Mailbox path (required). Special-use names such as `\Sent` work too, and `\All` selects every message on Gmail and MS Graph accounts |
| `page` | number | Page number (0-indexed, default 0). Only supported for IMAP accounts |
| `pageSize` | number | Messages per page (default 20, max 1000) |
| `cursor` | string | Paging cursor from `nextPageCursor` or `prevPageCursor` in the previous response. Works for every account type, and is the only way to page Gmail API and MS Graph accounts |

To filter messages by flags (unseen, flagged, etc.), use the [Search endpoint](#7-search-messages) instead.

**Examples:**

<Tabs groupId="programming-language">
<TabItem value="curl" label="cURL">

```bash
curl "https://emailengine.example.com/v1/account/user@example.com/messages?path=INBOX&pageSize=50" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

</TabItem>
<TabItem value="python" label="Python">

```python
from urllib.parse import quote

account = 'user@example.com'
response = requests.get(
    f'https://emailengine.example.com/v1/account/{quote(account)}/messages',
    params={'path': 'INBOX', 'pageSize': 50},
    headers={'Authorization': 'Bearer YOUR_ACCESS_TOKEN'}
)

data = response.json()
print(f"Total messages: {data['total']}")
for msg in data['messages']:
    print(f"{msg['from']['address']}: {msg['subject']}")
```

</TabItem>
</Tabs>

**Response:**
```json
{
  "total": 128,
  "page": 0,
  "pages": 3,
  "nextPageCursor": "eyJhIjoxfQ",
  "prevPageCursor": null,
  "messages": [
    {
      "id": "AAAABAABNc",
      "uid": 12345,
      "subject": "Meeting Tomorrow",
      "from": {
        "name": "John Doe",
        "address": "john@example.com"
      },
      "date": "2025-01-15T10:30:00.000Z",
      "unseen": true,
      "size": 15234
    }
  ]
}
```

**Use Cases:**
- Display inbox messages in application UI
- Process unread messages for automation
- Export messages for archival

[Detailed API reference →](/docs/api/get-v-1-account-account-messages)

---

### 2. Get Message Details

Retrieve complete message information including body and attachments.

**Endpoint:** `GET /v1/account/:account/message/:message`

**Path Parameters:**

| Parameter | Description |
|-----------|-------------|
| `account` | Account identifier |
| `message` | Message ID |

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `maxBytes` | number | Maximum bytes to retrieve for text/html |
| `textType` | string | Which text format to return: 'html', 'plain', or '*' for all. By default text content is not returned |
| `webSafeHtml` | boolean | Return the HTML processed for display in a web page, with inline images embedded. See [Web-safe HTML](/docs/receiving/web-safe-html) |
| `embedAttachedImages` | boolean | Embed attached images in the HTML as data URIs |
| `preProcessHtml` | boolean | Pre-process the HTML for compatibility |
| `markAsSeen` | boolean | Mark an unseen message as seen while returning it |

**Examples:**

```bash
curl "https://emailengine.example.com/v1/account/user@example.com/message/AAAABAABNc" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

**Response:**
```json
{
  "id": "AAAABAABNc",
  "uid": 12345,
  "subject": "Meeting Tomorrow",
  "from": {
    "name": "John Doe",
    "address": "john@example.com"
  },
  "to": [
    {
      "name": "Jane Smith",
      "address": "jane@example.com"
    }
  ],
  "date": "2025-01-15T10:30:00.000Z",
  "text": {
    "plain": "Let's meet tomorrow at 10 AM.",
    "html": "<p>Let's meet tomorrow at 10 AM.</p>"
  },
  "headers": {
    "content-type": ["text/plain; charset=utf-8"],
    "date": ["Wed, 15 Jan 2025 10:30:00 +0000"]
  },
  "attachments": []
}
```

**Use Cases:**
- Display full message in email client
- Extract message content for processing
- Download attachments

[Detailed API reference →](/docs/api/get-v-1-account-account-message-message)

---

### 3. Get Message Source

Retrieve raw RFC822 message source.

**Endpoint:** `GET /v1/account/:account/message/:message/source`

**Examples:**

```bash
curl "https://emailengine.example.com/v1/account/user@example.com/message/AAAABAABNc/source" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -o message.eml
```

**Use Cases:**
- Export messages in EML format
- Parse with custom email parser
- Forensic analysis
- Message backup

[Detailed API reference →](/docs/api/get-v-1-account-account-message-message-source)

---

### 4. Update Message Flags

Change message flags like read/unread, flagged, etc.

**Endpoint:** `PUT /v1/account/:account/message/:message`

**Request Body:**
```json
{
  "flags": {
    "add": ["\\Seen", "\\Flagged"],
    "delete": ["\\Draft"]
  }
}
```

`flags.set` replaces the whole flag list instead. Gmail API accounts also take a `labels` object with the same `add`, `delete` and `set` keys, holding label IDs or paths.

**Standard IMAP Flags:**

| Flag | Description |
|------|-------------|
| `\\Seen` | Message has been read |
| `\\Answered` | Message has been replied to |
| `\\Flagged` | Message is flagged/starred |
| `\\Deleted` | Message is marked for deletion |
| `\\Draft` | Message is a draft |

**Examples:**

```bash
curl -X PUT "https://emailengine.example.com/v1/account/user@example.com/message/AAAABAABNc" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "flags": {
      "add": ["\\Seen"]
    }
  }'
```

**Use Cases:**
- Mark messages as read/unread
- Star/flag important messages
- Track replied messages
- Implement custom workflow flags

[Detailed API reference →](/docs/api/put-v-1-account-account-message-message)

---

### 5. Move Message

Move a message to a different mailbox.

**Endpoint:** `PUT /v1/account/:account/message/:message/move`

**Request Body:**
```json
{
  "path": "Archive"
}
```

Gmail API accounts also take `source`, the folder the message is being moved out of, because a move there means swapping labels.

**Examples:**

```bash
curl -X PUT "https://emailengine.example.com/v1/account/user@example.com/message/AAAABAABNc/move" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"path": "Archive"}'
```

**Response:**
```json
{
  "path": "Archive",
  "id": "AAAABQABNd",
  "uid": 12346
}
```

**Note:** The `id` and `uid` fields are only included if the server provides them. Moving a message may change its ID since it's technically a new message in the destination mailbox.

**Use Cases:**
- Archive processed messages
- Move spam to Spam folder
- Organize messages into folders
- Implement auto-filing rules

[Detailed API reference →](/docs/api/put-v-1-account-account-message-message-move)

---

### 6. Delete Message

Delete a message permanently or move to trash.

**Endpoint:** `DELETE /v1/account/:account/message/:message`

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `force` | boolean | If true, delete message even if not in Trash. Not supported for Gmail API accounts |

**Examples:**

```bash
# Delete (moves to Trash, or deletes if already in Trash)
curl -X DELETE "https://emailengine.example.com/v1/account/user@example.com/message/AAAABAABNc" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"

# Force delete (delete even if not in Trash)
curl -X DELETE "https://emailengine.example.com/v1/account/user@example.com/message/AAAABAABNc?force=true" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

**Response:**
```json
{
  "deleted": false,
  "moved": {
    "destination": "Trash",
    "message": "AAAAAwAAAWg"
  }
}
```

`deleted` is `true` only when the message was removed for good. On IMAP accounts a move to Trash reports `false` together with `moved`, whose `destination` and `message` are present when the server reported them. Gmail API and MS Graph accounts report `true` for a move to Trash as well. For Gmail API accounts the message is always moved to Trash; `force` has no effect there.

**Use Cases:**
- Delete spam messages
- Clean up after processing
- User-initiated deletion

[Detailed API reference →](/docs/api/delete-v-1-account-account-message-message)

---

### 7. Search Messages

Search messages using advanced query syntax.

**Endpoint:** `POST /v1/account/:account/search`

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `path` | string | Mailbox path to search in. A search covers one folder, never the whole account; `\All` covers everything on Gmail and MS Graph accounts |
| `page` | number | Page number (0-indexed, default 0). Only supported for IMAP accounts |
| `pageSize` | number | Messages per page (default 20, max 1000) |
| `cursor` | string | Paging cursor from `nextPageCursor` or `prevPageCursor` |
| `useOutlookSearch` | boolean | MS Graph only: run the query through the `$search` parameter instead of `$filter` |

**Request Body:**
```json
{
  "search": {
    "from": "sender@example.com",
    "subject": "invoice",
    "since": "2025-01-01T00:00:00.000Z"
  }
}
```

**Search Criteria:**

| Field | Type | Description |
|-------|------|-------------|
| `from` | string | From address contains |
| `to`, `cc`, `bcc` | string | Recipient address contains. Not supported for MS Graph |
| `subject` | string | Subject contains |
| `body` | string | Body text contains |
| `since` | date | Messages received after date |
| `before` | date | Messages received before date |
| `sentSince` | date | Messages sent after date |
| `sentBefore` | date | Messages sent before date |
| `unseen`, `seen` | boolean | Unread or read messages only |
| `flagged` | boolean | Flagged messages only |
| `draft` | boolean | Draft messages only |
| `answered`, `deleted` | boolean | Replied-to, or marked for deletion. IMAP only |
| `larger` | number | Messages larger than size in bytes. Not supported for MS Graph |
| `smaller` | number | Messages smaller than size in bytes. Not supported for MS Graph |
| `header` | object | Header name to value, searched in the named headers |
| `emailId` | string | A specific message by its `emailId`; `emailIds` takes a list |
| `threadId` | string | Every message in a thread |
| `uid`, `seq` | string | UID or sequence number range such as `100:200` or `150,200,250`. IMAP only |
| `modseq` | number | Messages changed since a modification sequence. IMAP with CONDSTORE only |
| `gmailRaw` | string | A query in Gmail's own search syntax. Gmail accounts only |
| `labels` | object | `{ "has": ["Work"], "not": ["Spam"] }` - filter by Gmail labels or Outlook categories. `has` requires all listed labels, `not` excludes messages with any of them. Gmail (API or IMAP) and MS Graph accounts only; any other IMAP server answers HTTP 422 with the code `MissingServerExtension` |

**Examples:**

```bash
curl -X POST "https://emailengine.example.com/v1/account/user@example.com/search?path=INBOX" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "search": {
      "subject": "invoice",
      "since": "2025-01-01T00:00:00.000Z"
    }
  }'
```

**Use Cases:**
- Find messages from specific sender
- Search for messages with attachments
- Filter by date range
- Full-text search across messages

[Detailed API reference →](/docs/api/post-v-1-account-account-search)

---

## Message Object Reference

### Complete Field Reference

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | EmailEngine message ID |
| `uid` | number | IMAP UID. IMAP accounts only |
| `path` | string | Mailbox path. In a listing, present when listing `\All` or another virtual folder; in message details, not returned for Gmail API accounts |
| `emailId` | string | RFC 8474 Email ID, when the server supports one |
| `threadId` | string | RFC 8474 Thread ID, when the server supports one |
| `date` | string | ISO date string |
| `flags` | array | IMAP flags |
| `unseen` | boolean | True if unread |
| `flagged` | boolean | True if flagged |
| `answered` | boolean | True if replied to (the `\Answered` flag). IMAP accounts only |
| `draft` | boolean | True if draft |
| `size` | number | Message size in bytes. Not returned for MS Graph accounts |
| `subject` | string | Subject line |
| `from` | object | Sender address |
| `sender` | object | The `Sender` header, when it differs from `From`. Message details only |
| `to` | array | Recipient addresses |
| `cc` | array | CC addresses |
| `bcc` | array | BCC addresses. Message details only |
| `replyTo` | array | Reply-To addresses |
| `messageId` | string | RFC 5322 Message-ID |
| `inReplyTo` | string | Message-ID being replied to |
| `text` | object | Text content metadata; the `text.plain` and `text.html` values are included when the `textType` parameter is requested |
| `headers` | object | Raw headers, each value an array because a header can repeat. Message details only |
| `labels` | array | Gmail labels, on Gmail accounts |
| `preview` | string | A short plaintext preview, on Gmail API and MS Graph accounts |
| `category` | string | Gmail inbox tab (`primary`, `social`, `promotions`, `updates`, `forums`). Gmail API accounts only |
| `specialUse` | string | The special-use role of the folder the message is in, such as `\Sent`. Message details on IMAP accounts only |
| `messageSpecialUse` | string | The special-use role of the message itself, such as `\Sent` or `\Junk` |
| `isAutoReply` | boolean | True when the message looks like an automatic reply |
| `bounces` | array | Bounces recorded for a message this account sent. IMAP accounts only |
| `attachments` | array | Attachment metadata |

### Nested Structures

**Address Object:**
```json
{
  "name": "John Doe",
  "address": "john@example.com"
}
```

**Text Object:**
```json
{
  "id": "AAAAAQAACnAWkeM",
  "encodedSize": {
    "plain": 1013,
    "html": 3487
  },
  "plain": "Message text content",
  "html": "<p>Message HTML content</p>",
  "hasMore": false,
  "webSafe": false
}
```

`encodedSize` reports the size of each MIME part separately, before decoding. `plain` and `html` are present only when `textType` asked for them, `hasMore` says whether `maxBytes` truncated the result, and `webSafe` is set when `webSafeHtml` was requested and `html` has been processed for display.

**Attachment Object:**
```json
{
  "id": "AAAAAQAACnAy",
  "contentType": "application/pdf",
  "filename": "document.pdf",
  "encodedSize": 52341,
  "embedded": false,
  "inline": false,
  "encodedInMessage": false,
  "contentId": "<part1.abc@example.com>",
  "method": "REQUEST"
}
```

`encodedSize` is the size as stored in the message, base64 encoded, so the decoded file is roughly three quarters of it. `embedded` marks a part of a `multipart/related` body, `inline` a part meant to be shown in place rather than listed, `encodedInMessage` a part that belongs to an attached `message/rfc822` rather than to the message itself, and `contentId` is what a `cid:` URL in the HTML refers to. `method` is present on iCalendar attachments and carries the calendar method (`REQUEST`, `REPLY`, `CANCEL`).

## Filtering & Search

### Query Parameters

**List Messages Filters:**

The message listing endpoint only accepts the `path`, `page`, `pageSize`, and `cursor` query parameters - unknown parameters return HTTP 400. To filter by flags, use the search endpoint instead:

```bash
BASE=https://emailengine.example.com
ACCOUNT=user%40example.com
AUTH="Authorization: Bearer YOUR_ACCESS_TOKEN"

# A specific mailbox
curl "$BASE/v1/account/$ACCOUNT/messages?path=Archive" -H "$AUTH"

# Pagination
curl "$BASE/v1/account/$ACCOUNT/messages?path=INBOX&page=2&pageSize=100" -H "$AUTH"

# Unread messages: filtering by flag needs the search endpoint
curl -X POST "$BASE/v1/account/$ACCOUNT/search?path=INBOX" \
  -H "$AUTH" -H "Content-Type: application/json" \
  -d '{"search": {"unseen": true}}'

# Flagged messages
curl -X POST "$BASE/v1/account/$ACCOUNT/search?path=INBOX" \
  -H "$AUTH" -H "Content-Type: application/json" \
  -d '{"search": {"flagged": true}}'
```

### Search Syntax

**Advanced Search Examples:**

Each of these is the request body for `POST /v1/account/{account}/search`. Terms within one `search` object are combined with AND.

Messages from a domain:

```json
{ "search": { "from": "@example.com" } }
```

A date range, on the internal date the server assigned:

```json
{ "search": { "since": "2025-01-01T00:00:00.000Z", "before": "2025-02-01T00:00:00.000Z" } }
```

Several criteria at once, here unread invoices over 100 kB from one sender:

```json
{
  "search": {
    "from": "client@example.com",
    "subject": "invoice",
    "unseen": true,
    "larger": 100000
  }
}
```

Body text search, which the mail server performs and which is therefore slower than header searches:

```json
{ "search": { "body": "urgent payment" } }
```

## Common Patterns

### Pagination

Walk a folder page by page, stopping once you reach the last one:

```javascript
async function* eachMessage(account, path = 'INBOX') {
  const base = `https://emailengine.example.com/v1/account/${encodeURIComponent(account)}/messages`;

  for (let page = 0; ; page++) {
    const res = await fetch(`${base}?path=${encodeURIComponent(path)}&page=${page}&pageSize=100`, {
      headers: { Authorization: 'Bearer YOUR_ACCESS_TOKEN' }
    });

    const { messages, pages } = await res.json();
    yield* messages;

    if (page >= pages - 1) return;
  }
}
```

Yielding as you go, rather than accumulating into one array, keeps memory flat on a large mailbox. Messages arriving while you walk shift the page boundaries, so deduplicate on `id` if completeness matters.

### Real-time Sync with Webhooks

Polling a mailbox to notice new mail is the pattern to avoid. Subscribe once, and EmailEngine tells you:

```bash
curl -X POST https://emailengine.example.com/v1/settings \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "webhooks": "https://your-app.com/webhook",
    "webhooksEnabled": true,
    "webhookEvents": ["messageNew", "messageDeleted"]
  }'
```

The [`messageNew`](/docs/webhooks/messagenew) payload already carries the envelope, headers, and optionally the text body, so most handlers never need to fetch the message afterwards. See [Webhook Overview](/docs/webhooks/overview) for building the receiver.

### Bulk Operations

Do not loop a request per message. `PUT /v1/account/{account}/messages` applies one update to everything matching a search, in a single call:

```bash
curl -X PUT "https://emailengine.example.com/v1/account/user%40example.com/messages?path=INBOX" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "search": { "unseen": true },
    "update": { "flags": { "add": ["\\Seen"] } }
  }'
```

The folder is the `path` query parameter, while `search` holds the matching criteria. `PUT /v1/account/{account}/messages/move` and `PUT /v1/account/{account}/messages/delete` follow the same shape to move or delete in bulk.

:::caution An empty `search` matches every message
`search` is required, but `{}` is valid and selects the whole folder. Run the criteria through the [search endpoint](#7-search-messages) first and check the count before reusing them in a bulk delete or move.
:::

### Attachment Handling

Attachment IDs come from the message, so fetching an attachment is always a two-step operation:

```javascript
const account = encodeURIComponent('user@example.com');
const base = `https://emailengine.example.com/v1/account/${account}`;
const headers = { Authorization: 'Bearer YOUR_ACCESS_TOKEN' };

const message = await fetch(`${base}/message/AAAABAABNc`, { headers }).then(r => r.json());

for (const attachment of message.attachments || []) {
  const res = await fetch(`${base}/attachment/${attachment.id}`, { headers });
  // The response is the raw file, not JSON
  await writeFile(attachment.filename, Buffer.from(await res.arrayBuffer()));
}
```

The attachment endpoint returns the decoded file with its own content type, not a JSON wrapper. See [Attachments](/docs/receiving/attachments) for inline images, size limits, and streaming large files.

## See Also

- [Message Operations](/docs/receiving/message-operations) - Flags, moves, deletes, and folder handling in depth
- [Searching Messages](/docs/receiving/searching) - The full search grammar and provider differences
- [Attachments](/docs/receiving/attachments) - Downloading, inline images, and size limits
- [Sending API](/docs/api-reference/sending-api) - Submitting, scheduling, and the outbox
- [Message IDs Explained](/docs/advanced/ids-explained) - How `id`, `uid`, `emailId`, and `messageId` differ
