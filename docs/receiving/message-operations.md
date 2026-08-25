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

Message operations allow you to list, fetch, move, delete, and update email messages programmatically. These operations work consistently across IMAP, Gmail API, and Microsoft Graph backends.

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
      "bcc": [],
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

### Pagination

List messages with pagination:

```javascript
async function listMessages(accountId, folderPath, page = 0, pageSize = 20) {
  const params = new URLSearchParams({
    path: folderPath,
    page: page,
    pageSize: pageSize
  });

  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/messages?${params}`,
    {
      headers: { 'Authorization': 'Bearer YOUR_ACCESS_TOKEN' }
    }
  );

  return await response.json();
}

// List first page
const page1 = await listMessages('example', 'INBOX', 0, 20);
console.log(`Showing ${page1.messages.length} of ${page1.total} messages`);
console.log(`Page ${page1.page + 1} of ${page1.pages}`);

// List next page
const page2 = await listMessages('example', 'INBOX', 1, 20);
```

### List All Messages

Iterate through all pages:

```javascript
async function listAllMessages(accountId, folderPath) {
  const allMessages = [];
  let page = 0;
  let hasMore = true;

  while (hasMore) {
    const response = await listMessages(accountId, folderPath, page, 100);

    allMessages.push(...response.messages);

    page++;
    hasMore = page < response.pages;
  }

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

:::tip Displaying the HTML
If you intend to render the HTML body in a web page, request it with `webSafeHtml=true` instead. See [Web-Safe HTML](/docs/receiving/web-safe-html).
:::

```bash
curl "https://emailengine.example.com/v1/account/example/message/AAAAAQAAAeE?textType=*" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

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

Fetch raw RFC822 message source using the [message source API](/docs/api/get-v-1-account-account-message-message-source):

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

The response includes the destination `path` and, if provided by the server, the new message `id` and `uid` in the target folder.

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

Move messages to archive folder:

```javascript
async function archiveMessage(accountId, messageId) {
  // Find archive folder
  const folders = await listMailboxes(accountId);
  const archiveFolder = folders.find(f => f.specialUse === '\\Archive');

  if (!archiveFolder) {
    throw new Error('Archive folder not found');
  }

  return await moveMessage(accountId, messageId, archiveFolder.path);
}
```

### Move to Trash

```javascript
async function trashMessage(accountId, messageId) {
  const folders = await listMailboxes(accountId);
  const trashFolder = folders.find(f => f.specialUse === '\\Trash');

  if (!trashFolder) {
    throw new Error('Trash folder not found');
  }

  return await moveMessage(accountId, messageId, trashFolder.path);
}
```

## Deleting Messages

### Delete Message

Delete a message using the [delete message API](/docs/api/delete-v-1-account-account-message-message). By default, the message is moved to Trash. If the message is already in Trash (or if `force=true` is specified), it is permanently deleted.

```bash
curl -X DELETE "https://emailengine.example.com/v1/account/example/message/AAAAAQAAAeE" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

**Response (moved to Trash):**

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

To force permanent deletion without moving to Trash first, add `?force=true` to the URL (not supported for Gmail API accounts).

**JavaScript Example:**

```javascript
async function deleteMessage(accountId, messageId) {
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/message/${messageId}`,
    {
      method: 'DELETE',
      headers: { 'Authorization': 'Bearer YOUR_ACCESS_TOKEN' }
    }
  );

  return await response.json();
}

await deleteMessage('example', 'AAAAAQAAAeE');
```

### Delete Multiple Messages

```javascript
async function deleteMessages(accountId, messageIds) {
  const results = [];

  for (const messageId of messageIds) {
    try {
      const result = await deleteMessage(accountId, messageId);
      results.push({ messageId, success: true });
    } catch (err) {
      results.push({ messageId, success: false, error: err.message });
    }
  }

  return results;
}

const results = await deleteMessages('example', [
  'AAAAAQAAAeE',
  'AAAAAQAAAeF',
  'AAAAAQAAAeG'
]);
```

### Delete Behavior

The delete API has smart behavior:

- **Message not in Trash**: Moves to Trash (recoverable)
- **Message already in Trash**: Permanently deletes (not recoverable)
- **With `?force=true`**: Permanently deletes regardless of location (not supported for Gmail API)

This means you can safely use the delete endpoint in most cases - messages get a "second chance" in Trash before permanent deletion.

## Updating Message Flags

### Set Flags

Update message flags using the [update message API](/docs/api/put-v-1-account-account-message-message):

```bash
curl -X PUT "https://emailengine.example.com/v1/account/example/message/AAAAAQAAAeE" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "flags": {
      "add": ["\\Seen", "\\Flagged"],
      "delete": []
    }
  }'
```

**Response:**

```json
{
  "flags": {
    "add": ["\\Seen", "\\Flagged"]
  }
}
```

The response echoes back the flag operations that were performed.

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

Update flags on multiple messages:

```javascript
async function markAllAsRead(accountId, messageIds) {
  const results = [];

  for (const messageId of messageIds) {
    try {
      await markAsRead(accountId, messageId);
      results.push({ messageId, success: true });
    } catch (err) {
      results.push({ messageId, success: false, error: err.message });
    }
  }

  return results;
}

// Mark all unread messages as read
const unread = await listUnreadMessages('example', 'INBOX');
const messageIds = unread.messages.map(m => m.id);
await markAllAsRead('example', messageIds);
```

## Working with Gmail Labels and Outlook Categories

EmailEngine uses the `labels` array for both Gmail labels and Microsoft Outlook/Graph API categories.

**Gmail Labels:**
- Labels must be pre-created in Gmail
- Labels map to folders (e.g., label "Work" corresponds to a folder)
- Used with Gmail IMAP and Gmail API accounts

**Microsoft Outlook Categories:**
- Categories are **automatically created** when you set them (no pre-creation needed)
- Categories are **not folders** - they're an additional filter/tag
- Categories don't map to mailbox folders (unlike Gmail labels)
- Only available when using **Microsoft Graph API** backend

### Add Labels/Categories

For Gmail or Microsoft Graph accounts, add labels or categories to a message:

```bash
curl -X PUT "https://emailengine.example.com/v1/account/example/message/AAAAAQAAAeE" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "labels": {
      "add": ["Work", "Important"]
    }
  }'
```

**Example - Add categories to Outlook message:**

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

:::tip Microsoft Graph Auto-Creation
When using Microsoft Graph API, categories are automatically created if they don't exist. You don't need to pre-create them in Outlook.
:::

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

```javascript
async function setLabels(accountId, messageId, labels) {
  // First, get current labels
  const message = await getMessage(accountId, messageId);
  const currentLabels = message.labels || [];

  // Determine which to add and remove
  const toAdd = labels.filter(l => !currentLabels.includes(l));
  const toRemove = currentLabels.filter(l => !labels.includes(l));

  if (toAdd.length === 0 && toRemove.length === 0) {
    return message; // No changes needed
  }

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
          add: toAdd,
          delete: toRemove
        }
      })
    }
  ).then(r => r.json());
}

// Set exact labels (replaces all existing)
await setLabels('example', 'AAAAAQAAAeE', ['Work', 'Urgent']);
```

### Key Differences: Gmail Labels vs Outlook Categories

| Feature | Gmail Labels | Outlook Categories |
|---------|--------------|-------------------|
| **Pre-creation required** | Yes - must exist in Gmail | No - auto-created when set |
| **Folder mapping** | Yes - labels map to folders | No - separate filter/tag system |
| **Backend support** | Gmail IMAP, Gmail API | Microsoft Graph API only |
| **Use case** | Organize messages into folders | Add color-coded tags to messages |
| **Color support** | No colors | Colors assigned by Outlook (not via API) |
| **Delete/rename** | Yes - via Gmail | No - use Outlook directly |

:::info Important
Outlook categories are **not a replacement for folders**. They're an additional organizational layer. Unlike Gmail where a label creates a folder, Outlook categories are independent tags that don't affect folder structure.
:::

:::note Outlook Category Limitations
EmailEngine can only work with category **names**, not colors. When you assign a new category name to a message, Outlook automatically creates the category with a default color. To change colors, rename, or delete categories, you must use Outlook directly - these operations are not available through EmailEngine.
:::

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

```javascript
async function autoArchiveOldMessages(accountId, daysOld = 90) {
  const cutoffDate = new Date();
  cutoffDate.setDate(cutoffDate.getDate() - daysOld);

  const messages = await listAllMessages(accountId, 'INBOX');

  let archived = 0;

  for (const message of messages) {
    const messageDate = new Date(message.date);

    if (messageDate < cutoffDate) {
      try {
        await archiveMessage(accountId, message.id);
        archived++;
      } catch (err) {
        console.error(`Failed to archive ${message.id}:`, err);
      }
    }
  }

  console.log(`Archived ${archived} messages older than ${daysOld} days`);
  return archived;
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
