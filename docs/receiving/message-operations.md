---
title: Message Operations
sidebar_position: 4
description: "Complete guide to message operations - listing, fetching, moving, deleting, and updating messages"
keywords:
  - message operations
  - list messages
  - fetch message
  - delete message
  - message flags
  - message metadata
---

# Message Operations

Message operations allow you to list, fetch, move, delete, and update email messages programmatically. The same endpoints serve IMAP, Gmail API, and Microsoft Graph accounts; where a backend behaves differently, the difference is called out below.

## Listing Messages

### Basic Message Listing

List messages in a folder using the [messages listing API](/docs/api/get-v-1-account-account-messages):

```bash
curl "https://emailengine.example.com/v1/account/example/messages?path=INBOX" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

**Response:**

```json
{
  "total": 300,
  "page": 0,
  "pages": 15,
  "nextPageCursor": "imap_kcQIji3UobDDTxc",
  "prevPageCursor": null,
  "messages": [
    {
      "id": "AAAAAQAAAeE",
      "uid": 12345,
      "emailId": "1743d29c-b67d-4747-9016-b8850a5a39bd",
      "threadId": "1743d29c-b67d-4747-9016-b8850a5a39bd",
      "date": "2025-10-13T10:23:45.000Z",
      "flags": ["\\Seen"],
      "labels": ["\\Inbox"],
      "unseen": false,
      "flagged": false,
      "draft": false,
      "size": 45678,
      "subject": "Meeting Tomorrow",
      "from": {
        "name": "John Doe",
        "address": "john@example.com"
      },
      "to": [
        {
          "name": "Jane Smith",
          "address": "jane@company.com"
        }
      ],
      "cc": [],
      "messageId": "<abc123@example.com>",
      "inReplyTo": null,
      "attachments": [
        {
          "id": "AAAAAgAAAeEBAAAAAQAAAeE",
          "contentType": "application/pdf",
          "filename": "agenda.pdf"
        }
      ]
    }
  ]
}
```

Messages are returned newest first. `uid` is present for IMAP accounts only, and `emailId` and `threadId` only when the server provides them (Gmail API, Graph, and IMAP servers with the OBJECTID extension). `total` and `pages` are exact for IMAP accounts, approximate for Gmail API accounts, and can be missing for Graph accounts.

### Pagination

The listing takes `pageSize` (default 20, maximum 1000) and either a `cursor` or a `page` number:

- `cursor` takes the `nextPageCursor` or `prevPageCursor` value from a previous response. It works on every backend and is the form to prefer.
- `page` is a zero-based page number, kept for callers written before cursors. IMAP accounts honor it, and so do Microsoft Graph accounts; a Gmail API account rejects a `page` above 0 with a 400 (`InvalidInput`). When both are supplied, the cursor wins.

```javascript
async function listMessages(accountId, folderPath, cursor = null, pageSize = 20) {
  const params = new URLSearchParams({
    path: folderPath,
    pageSize: pageSize
  });

  if (cursor) {
    params.set('cursor', cursor);
  }

  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/messages?${params}`,
    {
      headers: { 'Authorization': 'Bearer YOUR_ACCESS_TOKEN' }
    }
  );

  return await response.json();
}

// First page
const page1 = await listMessages('example', 'INBOX', null, 20);
console.log(`Showing ${page1.messages.length} of ${page1.total} messages`);

// Next page
const page2 = await listMessages('example', 'INBOX', page1.nextPageCursor, 20);
```

### List All Messages

Follow `nextPageCursor` until it is `null`:

```javascript
async function listAllMessages(accountId, folderPath) {
  const allMessages = [];
  let cursor = null;

  do {
    const response = await listMessages(accountId, folderPath, cursor, 100);
    allMessages.push(...response.messages);
    cursor = response.nextPageCursor;
  } while (cursor);

  return allMessages;
}

const allInbox = await listAllMessages('example', 'INBOX');
console.log(`Total messages: ${allInbox.length}`);
```

### Filter by Flags

The listing endpoint only accepts `path`, `cursor`, `page`, and `pageSize` query parameters - it does not support flag filters. To filter by flags, use the [search API](/docs/api/post-v-1-account-account-search) instead.

Find unseen messages:

```bash
curl -X POST "https://emailengine.example.com/v1/account/example/search?path=INBOX" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "search": {
      "unseen": true
    }
  }'
```

Find flagged messages:

```bash
curl -X POST "https://emailengine.example.com/v1/account/example/search?path=INBOX" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "search": {
      "flagged": true
    }
  }'
```

**JavaScript Example:**

```javascript
async function listUnreadMessages(accountId, folderPath) {
  const params = new URLSearchParams({
    path: folderPath,
    pageSize: 100
  });

  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/search?${params}`,
    {
      method: 'POST',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        search: { unseen: true }
      })
    }
  );

  return await response.json();
}
```

## Fetching Messages

### Get Message by ID

Fetch complete message details using the [get message API](/docs/api/get-v-1-account-account-message-message). Text content is not included by default - add the `textType` query parameter (`plain`, `html`, or `*` for both) to include it:

```bash
curl "https://emailengine.example.com/v1/account/example/message/AAAAAQAAAeE?textType=*" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

The endpoint takes these query parameters:

| Parameter             | Type    | Default | Description                                                                                                                   |
| --------------------- | ------- | ------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `textType`            | string  | none    | Which body to include: `plain`, `html`, or `*` for both. Without it no body is returned                                       |
| `maxBytes`            | integer | none    | Cap on the returned body length. A capped body sets `text.hasMore` to `true`; fetch the rest from the text endpoint            |
| `webSafeHtml`         | boolean | `false` | Return sanitized HTML ready to render. Forces `textType=*`, `preProcessHtml=true` and `embedAttachedImages=true`               |
| `preProcessHtml`      | boolean | `false` | Sanitize and repair the HTML without embedding images                                                                          |
| `embedAttachedImages` | boolean | unset   | Replace `cid:` image references with data URIs. An explicit `false` is honored alongside `webSafeHtml`                        |
| `markAsSeen`          | boolean | `false` | Add the `\Seen` flag while fetching, if the message is unseen                                                                  |

:::tip Displaying the HTML
If you intend to render the HTML body in a web page, request it with `webSafeHtml=true`. See [Web-Safe HTML](/docs/receiving/web-safe-html).
:::

**Response:**

```json
{
  "id": "AAAAAQAAAeE",
  "uid": 12345,
  "emailId": "1743d29c-b67d-4747-9016-b8850a5a39bd",
  "threadId": "1743d29c-b67d-4747-9016-b8850a5a39bd",
  "date": "2025-10-13T10:23:45.000Z",
  "flags": ["\\Seen"],
  "labels": ["\\Inbox"],
  "unseen": false,
  "flagged": false,
  "draft": false,
  "size": 45678,
  "subject": "Meeting Tomorrow",
  "from": {
    "name": "John Doe",
    "address": "john@example.com"
  },
  "to": [
    {
      "name": "Jane Smith",
      "address": "jane@company.com"
    }
  ],
  "messageId": "<abc123@example.com>",
  "inReplyTo": null,
  "headers": {
    "date": ["Sun, 13 Oct 2025 10:23:45 +0000"],
    "from": ["John Doe <john@example.com>"],
    "to": ["Jane Smith <jane@company.com>"],
    "subject": ["Meeting Tomorrow"],
    "message-id": ["<abc123@example.com>"]
  },
  "text": {
    "id": "AAAAAgAAAeETEXT",
    "encodedSize": {
      "plain": 52,
      "html": 78
    },
    "plain": "Hi Jane,\n\nLet's meet tomorrow at 10am.\n\nBest,\nJohn",
    "html": "<p>Hi Jane,</p><p>Let's meet tomorrow at 10am.</p><p>Best,<br>John</p>"
  },
  "attachments": [
    {
      "id": "AAAAAgAAAeEBAAAAAQAAAeE",
      "contentType": "application/pdf",
      "encodedSize": 45000,
      "filename": "agenda.pdf"
    }
  ]
}
```

`headers` holds every header of the message, keyed in lower case, each value an array because a header can repeat. The response also carries `isAutoReply: true` when the message looks like an automatic reply (see [Tracking Email Replies](/docs/receiving/tracking-replies#filtering-auto-responses)) and `bounces` when EmailEngine has matched a bounce to it.

**JavaScript Example:**

```javascript
async function getMessage(accountId, messageId) {
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/message/${messageId}?textType=*`,
    {
      headers: { 'Authorization': 'Bearer YOUR_ACCESS_TOKEN' }
    }
  );

  if (!response.ok) {
    throw new Error(`Failed to fetch message: ${response.statusText}`);
  }

  return await response.json();
}

const message = await getMessage('example', 'AAAAAQAAAeE');
console.log(`From: ${message.from.address}`);
console.log(`Subject: ${message.subject}`);
console.log(`Body: ${message.text?.plain}`);
```

### Get Message Source

Fetch raw RFC822 message source using the [message source API](/docs/api/get-v-1-account-account-message-message-source). The response body is the message itself with a `message/rfc822` content type, not JSON:

```bash
curl "https://emailengine.example.com/v1/account/example/message/AAAAAQAAAeE/source" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

**Returns raw email:**

```
From: John Doe <john@example.com>
To: Jane Smith <jane@company.com>
Subject: Meeting Tomorrow
Date: Sun, 13 Oct 2025 10:23:45 +0000
Message-ID: <abc123@example.com>
Content-Type: text/plain; charset=utf-8

Hi Jane,

Let's meet tomorrow at 10am.

Best,
John
```

**JavaScript Example:**

```javascript
async function getMessageSource(accountId, messageId) {
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/message/${messageId}/source`,
    {
      headers: { 'Authorization': 'Bearer YOUR_ACCESS_TOKEN' }
    }
  );

  return await response.text();
}

const source = await getMessageSource('example', 'AAAAAQAAAeE');
console.log(source);
```

## Moving Messages

### Move to Different Folder

Move a message to another folder using the [move message API](/docs/api/put-v-1-account-account-message-message-move):

```bash
curl -X PUT "https://emailengine.example.com/v1/account/example/message/AAAAAQAAAeE/move" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "path": "Work/Projects"
  }'
```

**Response:**

```json
{
  "path": "Work/Projects",
  "id": "BBBBBQAAAeE",
  "uid": 5678
}
```

The response includes the destination `path` and, if the server provides them, the message's new `id` in the target folder and, for IMAP accounts, its new `uid`. An IMAP message gets a new `id` when it moves, because the ID encodes the folder and UID; see [Message IDs](/docs/advanced/ids-explained).

On a Gmail API account a move is a label change: the target label is added and, if you pass `source`, that label is removed. Without `source` the message keeps its old label as well:

```json
{
  "path": "Work/Projects",
  "source": "INBOX"
}
```

`path` accepts the special-use aliases `\Inbox`, `\Sent`, `\Drafts`, `\Trash` and `\Junk` in place of a folder path.

**JavaScript Example:**

```javascript
async function moveMessage(accountId, messageId, targetFolder) {
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/message/${messageId}/move`,
    {
      method: 'PUT',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ path: targetFolder })
    }
  );

  return await response.json();
}

// Move to archive
await moveMessage('example', 'AAAAAQAAAeE', 'Archive/2025');
```

### Archive Messages

`\Archive` is not one of the path aliases the message endpoints resolve, so look the folder up in the [mailbox listing](/docs/receiving/mailbox-operations#using-special-use-folders) first:

```javascript
async function listMailboxes(accountId) {
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/mailboxes`,
    {
      headers: { 'Authorization': 'Bearer YOUR_ACCESS_TOKEN' }
    }
  );

  return (await response.json()).mailboxes;
}

async function archiveMessage(accountId, messageId) {
  const folders = await listMailboxes(accountId);
  const archiveFolder = folders.find(f => f.specialUse === '\\Archive');

  if (!archiveFolder) {
    throw new Error('Archive folder not found');
  }

  return await moveMessage(accountId, messageId, archiveFolder.path);
}
```

Only IMAP servers flag an archive folder. Gmail API and Graph accounts do not; on Gmail, archiving is removing the `\Inbox` label (see [Working with Gmail Labels and Outlook Categories](#working-with-gmail-labels-and-outlook-categories)).

### Move to Trash

`\Trash` is an alias every backend resolves, so no lookup is needed:

```javascript
async function trashMessage(accountId, messageId) {
  return await moveMessage(accountId, messageId, '\\Trash');
}
```

The [delete endpoint](#deleting-messages) does the same thing by default, and also handles the case where the message is already in Trash.

## Deleting Messages

### Delete Message

Delete a message using the [delete message API](/docs/api/delete-v-1-account-account-message-message):

```bash
curl -X DELETE "https://emailengine.example.com/v1/account/example/message/AAAAAQAAAeE" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

**Response (IMAP, moved to Trash):**

```json
{
  "deleted": false,
  "moved": {
    "destination": "Trash",
    "message": "AAAAAwAAAWg"
  }
}
```

**Response (permanently deleted):**

```json
{
  "deleted": true
}
```

**JavaScript Example:**

```javascript
async function deleteMessage(accountId, messageId, force = false) {
  const params = force ? '?force=true' : '';

  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/message/${messageId}${params}`,
    {
      method: 'DELETE',
      headers: { 'Authorization': 'Bearer YOUR_ACCESS_TOKEN' }
    }
  );

  return await response.json();
}

await deleteMessage('example', 'AAAAAQAAAeE');
```

### Delete Behavior

What a delete does, and what the response says, depends on the backend:

| Backend         | Default                                                                                                   | With `force=true`                                  |
| --------------- | --------------------------------------------------------------------------------------------------------- | -------------------------------------------------- |
| IMAP            | Moves to the `\Trash` folder and returns `deleted: false` with `moved`. A message already in `\Trash` or `\Junk`, or on an account with no Trash folder, is expunged and returns `deleted: true` | Expunges wherever the message is; `deleted: true` |
| Gmail API       | Adds the `TRASH` label and returns `deleted: true` with `moved.message`                                   | Not supported; the message still goes to Trash    |
| Microsoft Graph | Moves to Deleted Items and returns `deleted: true` with `moved`                                           | Deletes permanently; `deleted: true`               |

So `deleted: false` is the only reliable signal that a message was moved rather than removed, and it only occurs on IMAP accounts. Check for `moved` when the distinction matters.

### Bulk Actions

Three endpoints act on every message matching a search in one call, instead of one request per message:

- [`PUT /v1/account/{account}/messages`](/docs/api/put-v-1-account-account-messages) updates flags and labels
- [`PUT /v1/account/{account}/messages/move`](/docs/api/put-v-1-account-account-messages-move) moves
- [`PUT /v1/account/{account}/messages/delete`](/docs/api/put-v-1-account-account-messages-delete) deletes, with the same `force` query parameter and per-backend behavior as a single delete

The folder is the `path` query parameter, and the body carries the same `search` object the [search endpoint](/docs/receiving/searching) takes. On Gmail API and Microsoft Graph accounts, `search.emailIds` acts on the listed `emailId` values and every other criterion is ignored; IMAP accounts ignore `emailIds` and select by the remaining criteria:

```bash
curl -X PUT "https://emailengine.example.com/v1/account/example/messages/delete?path=INBOX" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "search": {
      "seen": true,
      "before": "2025-01-01"
    }
  }'
```

**Response (IMAP, moved to Trash):**

```json
{
  "deleted": false,
  "moved": {
    "destination": "Trash",
    "idMap": [
      ["AAAAAQAAAeE", "AAAAAwAAAWg"],
      ["AAAAAQAAAeF", "AAAAAwAAAWh"]
    ]
  }
}
```

`idMap` pairs each source ID with its ID in Trash and is returned by IMAP accounts; Gmail API and Graph accounts return `moved.emailIds` instead, and a Graph account deleting with `force=true` lists the removed messages under `deletedMessages.emailIds`. `search` is required, but `{}` is valid and matches the whole folder, so run the criteria through the search endpoint first when the count matters.

## Updating Message Flags

### Set Flags

Update message flags using the [update message API](/docs/api/put-v-1-account-account-message-message):

```bash
curl -X PUT "https://emailengine.example.com/v1/account/example/message/AAAAAQAAAeE" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "flags": {
      "add": ["\\Seen", "\\Flagged"]
    }
  }'
```

`flags` takes `add`, `delete`, and `set`. `set` replaces the whole flag list, and when it is present `add` and `delete` are ignored.

**Response (IMAP):**

```json
{
  "flags": {
    "add": true
  }
}
```

An IMAP account reports each operation as `true` or `false`. Gmail API and Graph accounts echo the requested flags back and add `result`, the message's flag list after the update:

```json
{
  "flags": {
    "add": ["\\Seen", "\\Flagged"],
    "result": ["\\Seen", "\\Flagged"]
  }
}
```

### Common Flag Operations

**Mark as read:**

```javascript
async function markAsRead(accountId, messageId) {
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/message/${messageId}`,
    {
      method: 'PUT',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        flags: {
          add: ['\\Seen']
        }
      })
    }
  );

  return await response.json();
}
```

**Mark as unread:**

```javascript
async function markAsUnread(accountId, messageId) {
  return await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/message/${messageId}`,
    {
      method: 'PUT',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        flags: {
          delete: ['\\Seen']
        }
      })
    }
  ).then(r => r.json());
}
```

**Toggle flag/star:**

```javascript
async function toggleFlag(accountId, messageId, currentlyFlagged) {
  return await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/message/${messageId}`,
    {
      method: 'PUT',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        flags: currentlyFlagged
          ? { delete: ['\\Flagged'] }
          : { add: ['\\Flagged'] }
      })
    }
  ).then(r => r.json());
}
```

**Mark as answered:**

```javascript
async function markAsAnswered(accountId, messageId) {
  return await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/message/${messageId}`,
    {
      method: 'PUT',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        flags: {
          add: ['\\Answered']
        }
      })
    }
  ).then(r => r.json());
}
```

### Batch Flag Updates

Mark every unseen message in a folder as read with one bulk update:

```javascript
async function markFolderAsRead(accountId, folderPath) {
  const params = new URLSearchParams({ path: folderPath });

  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/messages?${params}`,
    {
      method: 'PUT',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        search: { unseen: true },
        update: { flags: { add: ['\\Seen'] } }
      })
    }
  );

  return await response.json();
}

await markFolderAsRead('example', 'INBOX');
```

## Working with Gmail Labels and Outlook Categories

EmailEngine uses the `labels` array for both Gmail labels and Microsoft Outlook categories. The two are different things, and [Mailbox Operations](/docs/receiving/mailbox-operations#working-with-gmail-labels-and-outlook-categories) describes each. What matters here is how the update endpoint treats them:

| Backend         | What `labels` holds       | `add` / `delete`               | `set`                                             |
| --------------- | ------------------------- | ------------------------------ | ------------------------------------------------- |
| Gmail IMAP      | Gmail labels              | Supported                      | Supported                                         |
| Gmail API       | Gmail labels              | Supported                      | Rejected with a 400 (`UnsupportedOperation`)      |
| Microsoft Graph | Outlook categories        | Supported (v2.58.0 and later)  | Supported                                         |
| Other IMAP      | Not applicable            | Ignored                        | Ignored                                           |

The value each backend expects differs. Gmail over IMAP and Graph take the label or category name. A Gmail API account takes the label ID, the `id` of the label in the [mailbox listing](/docs/receiving/mailbox-operations#listing-mailboxes) (`Label_971539351003152516`), or a special-use flag such as `\Inbox`; a label name is passed to Gmail unchanged and rejected as an unknown label.

### Add Labels/Categories

Gmail over IMAP, by label name:

```bash
curl -X PUT "https://emailengine.example.com/v1/account/example/message/AAAAAQAAAeE" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "labels": {
      "add": ["Work", "Invoices"]
    }
  }'
```

Gmail API, by label ID:

```bash
curl -X PUT "https://emailengine.example.com/v1/account/gmail-account/message/AAAAAQAAAeE" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "labels": {
      "add": ["Label_971539351003152516"]
    }
  }'
```

**Example - Add categories to an Outlook message:**

```bash
curl -X PUT "https://emailengine.example.com/v1/account/outlook-account/message/AAMkADU" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "labels": {
      "add": ["Blue category", "Red category"]
    }
  }'
```

A category name that the mailbox has not seen before is written to the message as-is; EmailEngine does not manage the mailbox's category list.

### Remove Labels/Categories

```javascript
async function removeLabel(accountId, messageId, label) {
  return await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/message/${messageId}`,
    {
      method: 'PUT',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        labels: {
          delete: [label]
        }
      })
    }
  ).then(r => r.json());
}
```

### Set Labels/Categories (Replace All)

`labels.set` replaces the list in one call on Gmail IMAP and Graph accounts. For a Gmail API account, which rejects `set`, compute the difference and send `add` and `delete` together. The message reports its labels by name while the update takes IDs, so the mailbox listing is what translates between the two:

```javascript
async function setLabels(accountId, messageId, labels) {
  // First, get current labels (reported by name)
  const message = await getMessage(accountId, messageId);
  const currentLabels = message.labels || [];

  // Determine which to add and remove
  const toAdd = labels.filter(l => !currentLabels.includes(l));
  const toRemove = currentLabels.filter(l => !labels.includes(l));

  if (toAdd.length === 0 && toRemove.length === 0) {
    return message; // No changes needed
  }

  // Translate names to the label IDs a Gmail API account updates by
  const labelIds = new Map((await listMailboxes(accountId)).map(f => [f.path, f.id]));
  const toId = name => (name.startsWith('\\') ? name : labelIds.get(name) || name);

  return await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/message/${messageId}`,
    {
      method: 'PUT',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        labels: {
          add: toAdd.map(toId),
          delete: toRemove.map(toId)
        }
      })
    }
  ).then(r => r.json());
}

// Set exact labels (replaces all existing)
await setLabels('example', 'AAAAAQAAAeE', ['Work', 'Urgent']);
```

On a Gmail account the system labels come back as special-use flags (`\Inbox`, `\Sent`), and can be added or removed under those names. Removing `\Inbox` is how a Gmail message is archived.

## Common Patterns

### Process Unread Messages

```javascript
async function processUnreadMessages(accountId, processor) {
  const unread = await listUnreadMessages(accountId, 'INBOX');

  for (const message of unread.messages) {
    try {
      // Get full message content
      const fullMessage = await getMessage(accountId, message.id);

      // Process the message
      await processor(fullMessage);

      // Mark as read
      await markAsRead(accountId, message.id);

      console.log(`Processed message: ${message.subject}`);
    } catch (err) {
      console.error(`Failed to process ${message.id}:`, err);
    }
  }
}

// Usage
await processUnreadMessages('example', async (message) => {
  // Your processing logic
  console.log(`Processing: ${message.subject}`);
  // Extract data, send to API, etc.
});
```

### Auto-Archive Old Messages

Let the server select the messages, then move them all with one bulk request:

```javascript
async function autoArchiveOldMessages(accountId, daysOld = 90) {
  const cutoffDate = new Date();
  cutoffDate.setDate(cutoffDate.getDate() - daysOld);

  const folders = await listMailboxes(accountId);
  const archiveFolder = folders.find(f => f.specialUse === '\\Archive');

  if (!archiveFolder) {
    throw new Error('Archive folder not found');
  }

  const params = new URLSearchParams({ path: 'INBOX' });

  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/messages/move?${params}`,
    {
      method: 'PUT',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        search: { before: cutoffDate.toISOString().split('T')[0] },
        path: archiveFolder.path
      })
    }
  );

  return await response.json();
}

// Archive messages older than 90 days
await autoArchiveOldMessages('example', 90);
```

### Sync Flags to Database

```javascript
async function syncMessageFlags(accountId, folderPath, db) {
  const messages = await listAllMessages(accountId, folderPath);

  for (const message of messages) {
    await db.updateMessage({
      accountId: accountId,
      messageId: message.id,
      flags: message.flags,
      unseen: message.unseen,
      flagged: message.flagged,
      answered: message.flags.includes('\\Answered')
    });
  }

  console.log(`Synced ${messages.length} message flags`);
}
```

## See Also

- [Searching messages](/docs/receiving/searching) - Finding the messages to operate on
- [Attachments](/docs/receiving/attachments) - Downloading what a message carries
- [Web-safe HTML](/docs/receiving/web-safe-html) - Rendering a body in your own UI
- [Message IDs](/docs/advanced/ids-explained) - Which identifier survives a move, and which does not
- [Messages API](/docs/api-reference/messages-api) - The endpoint reference
