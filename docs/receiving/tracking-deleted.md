---
title: Tracking Deleted Messages
sidebar_position: 9
description: "How EmailEngine detects message deletions on IMAP, Gmail API and Microsoft Graph accounts, and how to tell a deletion from a move"
keywords:
  - deleted messages
  - IMAP EXPUNGE
  - message deletion
  - sync tracking
  - UIDVALIDITY
---

# Tracking Deleted Messages

Detecting that a message is gone is harder than detecting that one arrived. A new message announces itself; a deleted one leaves a gap that has to be noticed by comparing what the server has now with what EmailEngine saw before. This page explains how EmailEngine does that on each backend, what the `messageDeleted` webhook carries, and how to tell a deletion from a move.

## Why Deletion Tracking is Challenging

**IMAP Connection Limitations**
- An IMAP connection can watch one selected folder at a time with IDLE or NOOP
- Providers cap the number of simultaneous connections per account, so one connection per folder is not an option
- Changes in every other folder can only be found by asking

**Reconnection Issues**
- Network interruptions cause disconnects
- Forced logouts lose the notification state
- Nothing is announced for events while disconnected

**Sequence Number Complexity**
- EXPUNGE responses use sequence numbers, not UIDs
- Sequence numbers shift as messages are removed
- An accurate sequence-to-UID mapping has to be maintained to know which message went

## How EmailEngine Tracks Deletions

### IMAP Accounts

EmailEngine keeps an index of every folder it syncs: the UID, flags and a few headers of each message, plus the folder's UIDVALIDITY, UIDNEXT, message count and, where the server supports CONDSTORE, HIGHESTMODSEQ. Deletions are found in two ways.

**In the watched folder, as they happen.** The folder EmailEngine keeps selected (INBOX, or All Mail on Gmail) reports removals immediately as untagged `EXPUNGE` responses, or `VANISHED` responses when the server supports QRESYNC. EmailEngine resolves the sequence number or UID against its index, removes the entry, and emits `messageDeleted`.

**In every folder, on the resync cycle.** Every `imap.resyncDelay` seconds (default 900) EmailEngine runs `STATUS` on each folder and compares the result with the stored values:

```
Before: messages=100, UIDNEXT=150, HIGHESTMODSEQ=12345
After:  messages=95,  UIDNEXT=150, HIGHESTMODSEQ=12351

Conclusion: the folder changed, and since UIDNEXT did not move,
nothing was added - 5 messages were removed
```

If the counts, UIDNEXT and HIGHESTMODSEQ all match the stored values, nothing changed and the folder is skipped. Otherwise the folder is opened and synced. When the drift can be explained by additions alone, a partial sync fetches only the new messages. When it cannot, a full sync walks the server's message list and compares it against the index:

```
Index:  [100, 101, 102, 103, 104, 105]
Server: [100, 102, 104, 105, 106]

Deleted: [101, 103]
Added:   [106]
```

Each UID missing from the server produces a `messageDeleted` event. On a server without CONDSTORE, a full sync also runs at least every 30 minutes for a folder whose counts did not change, because such a server cannot report flag changes any other way.

**UIDVALIDITY change.** If a folder's UIDVALIDITY differs from the stored one, the server has invalidated every UID EmailEngine knows. The index is rebuilt silently and a single [`mailboxReset`](/docs/webhooks/mailboxreset) event is emitted instead of a `messageDeleted` per message.

:::warning Fast indexer
Deletion tracking needs the full indexer, which is the default. An account set to the fast indexer keeps no per-message index, ignores EXPUNGE responses, and never emits `messageDeleted`. See [IMAP indexers](/docs/accounts/imap-indexers).
:::

### Gmail API Accounts

EmailEngine reads the account's history log, which lists each message that was permanently removed. Moving a message to Trash, or between labels, is a label change in Gmail and arrives as [`messageUpdated`](/docs/webhooks/messageupdated) with the label difference; `messageDeleted` is emitted only when the message is gone from the account.

### Microsoft Graph Accounts

Graph sends a `deleted` change notification for a message that was removed from a folder. Before emitting `messageDeleted`, EmailEngine checks whether the message still exists in the mailbox. A message that was moved rather than deleted still exists, so no `messageDeleted` is sent for it; the destination folder reports it as a new message instead.

## Deletion Webhooks

### messageDeleted Event

When EmailEngine detects a deleted message, it sends a [`messageDeleted` webhook](/docs/webhooks/messagedeleted):

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "example",
  "date": "2025-01-15T10:30:00.000Z",
  "path": "INBOX",
  "specialUse": "\\Inbox",
  "event": "messageDeleted",
  "data": {
    "id": "AAAAAQAAAeE",
    "uid": 12345
  }
}
```

**Important Fields:**
- `path` - Folder the message was deleted from (at root level, not inside `data`)
- `data.id` - EmailEngine's message ID of the deleted message
- `data.uid` - IMAP UID of the deleted message

:::note Provider Differences
The payload varies by account type:
- **IMAP accounts**: `data` contains `id` and `uid`
- **Gmail API accounts**: `data` contains `id`, `threadId`, `flags`, `labels`, and `category`
- **Microsoft Graph accounts**: `data` contains only `id`
:::

The message itself can no longer be fetched. Keep whatever you need about a message from its `messageNew` event.

### messageMissing Event

A related event, [`messageMissing`](/docs/webhooks/messagemissing), is emitted when EmailEngine was told about a new message but could not fetch it, because it was removed before the fetch. It carries `id`, plus `uid`, `missingRetries` and `missingDelay` for IMAP accounts, and no message content. Treat it as "this message existed briefly"; there will be no `messageNew` for it.

### Handling Deletion Webhooks

```javascript
const express = require('express');
const app = express();

app.use(express.json());

app.post('/webhooks/emailengine', async (req, res) => {
  const event = req.body;

  // Acknowledge immediately
  res.status(200).json({ success: true });

  // Process asynchronously
  if (event.event === 'messageDeleted') {
    await handleMessageDeleted(event);
  }
});

async function handleMessageDeleted(event) {
  const { account, path, data } = event;

  console.log(`Message deleted from ${account}:`);
  console.log(`- Folder: ${path}`);
  console.log(`- UID: ${data.uid}`);
  console.log(`- EmailEngine ID: ${data.id}`);

  // Update your database
  await updateMessageStatus(data.id, 'deleted');

  // Sync deletion to external system
  await syncDeletion(account, { ...data, path });
}

async function updateMessageStatus(messageId, status) {
  // Update in your database
  await db.messages.update(
    { emailEngineId: messageId },
    { $set: { status: status, deletedAt: new Date() } }
  );
}

app.listen(3000);
```

## Tracking Deletions in Your Application

### Store Message State

Maintain a local copy of message state:

```javascript
// Database schema example
const messageSchema = {
  emailEngineId: String,      // EmailEngine message ID
  accountId: String,           // Email account
  folderPath: String,          // Folder location
  uid: Number,                 // IMAP UID
  emailId: String,             // Unique email ID
  threadId: String,            // Thread ID
  messageId: String,           // Message-ID header
  subject: String,
  from: Object,
  date: Date,
  status: String,              // 'active', 'deleted', 'moved'
  deletedAt: Date,
  createdAt: Date,
  updatedAt: Date
};
```

### Sync Deletions

When a deletion webhook arrives:

```javascript
async function syncDeletion(accountId, deletionData) {
  const message = await db.messages.findOne({
    emailEngineId: deletionData.id
  });

  if (!message) {
    console.log('Message not found in local database');
    return;
  }

  // Mark as deleted
  await db.messages.update(
    { _id: message._id },
    {
      $set: {
        status: 'deleted',
        deletedAt: new Date()
      }
    }
  );

  console.log(`Synced deletion of: ${message.subject}`);

  // Trigger additional actions
  await onMessageDeleted(message);
}

async function onMessageDeleted(message) {
  // Examples of actions:

  // 1. Update search index
  await searchIndex.delete(message.emailEngineId);

  // 2. Clean up attachments
  await cleanupAttachments(message.emailEngineId);

  // 3. Update statistics
  await analytics.recordDeletion({
    accountId: message.accountId,
    folderPath: message.folderPath,
    date: new Date()
  });

  // 4. Notify users
  if (message.important) {
    await notifyUser({
      type: 'message-deleted',
      subject: message.subject,
      from: message.from
    });
  }
}
```

## Deleted vs Moved

### Distinguishing Moves from Deletions

A message might appear "deleted" from one folder because it was moved to another. What you see depends on the account type:

**Gmail API accounts:** A move is a label change and arrives as `messageUpdated`, not `messageDeleted`. If you receive `messageDeleted`, the message is gone.

**Microsoft Graph accounts:** A move produces no `messageDeleted` at all, only a `messageNew` in the destination folder carrying the same `emailId` as before. If you receive `messageDeleted`, the message is gone.

**IMAP accounts:** A move is a `messageDeleted` in the source folder followed by a `messageNew` in the destination, and the message gets a new UID and a new EmailEngine `id` there. The two events have to be correlated by something that survives the move:

- `emailId`, when the server supports the OBJECTID extension (RFC 8474) or is Gmail. It is in the `messageNew` payload and identifies the message across folders
- Otherwise the `messageId` field, the Message-ID header. Copies of one message share it, so also check that the original is the one that was just reported deleted

**Gmail over IMAP** is the exception among IMAP accounts. EmailEngine syncs `[Gmail]/All Mail`, `[Gmail]/Spam` and `[Gmail]/Trash` rather than every label, so a move between labels is a label change on the All Mail copy and arrives as `messageUpdated`. Only moving a message to Trash or Spam removes it from All Mail, and that produces the `messageDeleted` and `messageNew` pair described above.

```javascript
// Track moved messages by matching messageNew events against recent deletions.
// This requires storing emailId (or messageId) when messages are first received.
const recentDeletions = new Map();

async function handleMessageDeleted(event) {
  const { path, data } = event;

  // Look up the stored copy of the deleted message
  const storedMessage = await db.messages.findOne({ emailEngineId: data.id });
  const key = storedMessage && (storedMessage.emailId || storedMessage.messageId);

  if (key) {
    recentDeletions.set(key, {
      id: data.id,
      path: path,
      timestamp: Date.now()
    });

    // Clean up old entries after 60 seconds
    setTimeout(() => {
      recentDeletions.delete(key);
    }, 60000);
  }

  await updateMessageStatus(data.id, 'deleted');
}

async function handleMessageNew(event) {
  const { path, data } = event;

  // A moved message carries the same emailId (or Message-ID) as the one just deleted
  const key = data.emailId || data.messageId;
  const recentDeletion = key ? recentDeletions.get(key) : null;

  if (recentDeletion) {
    console.log('Message was moved, not deleted');
    console.log(`From: ${recentDeletion.path}`);
    console.log(`To: ${path}`);

    // Update status to moved
    await updateMessageLocation(recentDeletion.id, path, data.id);

    recentDeletions.delete(key);
  } else {
    // Genuinely new message
    await createMessage(data);
  }
}

async function updateMessageLocation(oldId, newPath, newId) {
  await db.messages.update(
    { emailEngineId: oldId },
    {
      $set: {
        status: 'active',
        folderPath: newPath,
        emailEngineId: newId,
        updatedAt: new Date()
      },
      $unset: {
        deletedAt: ''
      }
    }
  );
}
```

The 60 second window covers a move inside the watched folder's connection. A move between two folders that are both found on the resync cycle can report the deletion and the arrival up to a cycle apart, so widen the window when moves between non-watched folders matter.

## Batch Deletion Detection

### Detect Mass Deletions

Alert when many messages are deleted at once:

```javascript
const deletionCounts = new Map();

async function handleMessageDeleted(event) {
  const { account, path, data } = event;
  const key = `${account}:${path}`;

  // Track deletions per folder per minute
  if (!deletionCounts.has(key)) {
    deletionCounts.set(key, {
      count: 0,
      timestamp: Date.now()
    });
  }

  const stats = deletionCounts.get(key);
  const now = Date.now();

  // Reset if more than 1 minute passed
  if (now - stats.timestamp > 60000) {
    stats.count = 0;
    stats.timestamp = now;
  }

  stats.count++;

  // Alert if more than 50 deletions in 1 minute
  if (stats.count > 50) {
    await alertMassDeletion({
      account: account,
      folder: path,
      count: stats.count,
      timeWindow: '1 minute'
    });
  }

  await syncDeletion(account, { ...data, path });
}

async function alertMassDeletion(info) {
  console.warn('MASS DELETION DETECTED:', info);

  // Send alert to admin
  await sendAdminAlert({
    type: 'mass-deletion',
    ...info
  });
}
```

A folder that is emptied while EmailEngine is disconnected, or one that is not the watched folder, reports all of its deletions in one burst when the next sync runs, so a burst is not by itself evidence that the deletions happened together.

## Recovery and Audit

### Maintain Deletion Audit Log

Keep a record of all deletions:

```javascript
const deletionLogSchema = {
  accountId: String,
  folderPath: String,
  messageId: String,
  emailId: String,
  subject: String,
  from: Object,
  date: Date,
  deletedAt: Date,
  deletionSource: String  // 'user', 'auto-archive', 'retention-policy'
};

async function logDeletion(message, source = 'user') {
  await db.deletionLog.insert({
    accountId: message.accountId,
    folderPath: message.folderPath,
    messageId: message.emailEngineId,
    emailId: message.emailId,
    subject: message.subject,
    from: message.from,
    date: message.date,
    deletedAt: new Date(),
    deletionSource: source
  });
}

// When deletion occurs
async function onMessageDeleted(message) {
  await logDeletion(message);

  // Continue with other actions...
}
```

### Query Deletion History

```javascript
async function getDeletionHistory(accountId, options = {}) {
  const query = { accountId };

  if (options.folder) {
    query.folderPath = options.folder;
  }

  if (options.since) {
    query.deletedAt = { $gte: new Date(options.since) };
  }

  const deletions = await db.deletionLog.find(query)
    .sort({ deletedAt: -1 })
    .limit(options.limit || 100)
    .toArray();

  return deletions;
}

// Get recent deletions
const recent = await getDeletionHistory('example', {
  since: '2025-10-01',
  folder: 'INBOX',
  limit: 50
});

console.log(`Found ${recent.length} deleted messages`);
```

## Soft Delete Pattern

### Implement Soft Deletes

Instead of immediately deleting, mark as deleted:

```javascript
async function handleMessageDeleted(event) {
  const { path, data } = event;

  // Soft delete: mark as deleted but keep in database
  await db.messages.update(
    { emailEngineId: data.id },
    {
      $set: {
        status: 'deleted',
        deletedAt: new Date(),
        // Keep original data for potential recovery
        originalFolderPath: path,
        originalUid: data.uid
      }
    }
  );

  // Schedule permanent deletion after 30 days
  await schedulePermanentDeletion(data.id, 30);
}

async function schedulePermanentDeletion(messageId, daysDelay) {
  const deleteAt = new Date();
  deleteAt.setDate(deleteAt.getDate() + daysDelay);

  await db.scheduledDeletions.insert({
    messageId: messageId,
    deleteAt: deleteAt
  });
}

// Periodic cleanup job
async function permanentlyDeleteOldMessages() {
  const now = new Date();

  const toDelete = await db.scheduledDeletions.find({
    deleteAt: { $lte: now }
  }).toArray();

  for (const item of toDelete) {
    // Permanently delete
    await db.messages.remove({ emailEngineId: item.messageId });
    await db.scheduledDeletions.remove({ _id: item._id });

    console.log(`Permanently deleted: ${item.messageId}`);
  }
}

// Run daily
setInterval(permanentlyDeleteOldMessages, 24 * 60 * 60 * 1000);
```

## Performance Considerations

### Batch Webhook Processing

Process deletion webhooks in batches:

```javascript
const deletionQueue = [];
let processingTimer = null;

async function handleMessageDeleted(event) {
  deletionQueue.push(event.data);

  // Process in batches every 5 seconds
  if (!processingTimer) {
    processingTimer = setTimeout(async () => {
      await processBatchDeletions();
      processingTimer = null;
    }, 5000);
  }
}

async function processBatchDeletions() {
  if (deletionQueue.length === 0) return;

  const batch = deletionQueue.splice(0, deletionQueue.length);

  console.log(`Processing ${batch.length} deletions`);

  // Batch database update
  const messageIds = batch.map(d => d.id);

  await db.messages.updateMany(
    { emailEngineId: { $in: messageIds } },
    {
      $set: {
        status: 'deleted',
        deletedAt: new Date()
      }
    }
  );

  console.log(`Batch update complete`);
}
```

### Index for Efficient Queries

Create appropriate database indexes:

```javascript
// MongoDB indexes
await db.messages.createIndex({ emailEngineId: 1 });
await db.messages.createIndex({ accountId: 1, status: 1 });
await db.messages.createIndex({ emailId: 1 });
await db.messages.createIndex({ messageId: 1 });
await db.messages.createIndex({ status: 1, deletedAt: 1 });
await db.deletionLog.createIndex({ accountId: 1, deletedAt: -1 });
```

## See Also

- [IMAP indexers](/docs/accounts/imap-indexers) - The fast indexer does not detect deletions at all
- [messageDeleted](/docs/webhooks/messagedeleted) - The event and its payload
- [messageMissing](/docs/webhooks/messagemissing) - A message that vanished before EmailEngine could fetch it
- [Message operations](/docs/receiving/message-operations) - Deleting a message yourself
- [mailboxReset](/docs/webhooks/mailboxreset) - When the server invalidates the index rather than the message
