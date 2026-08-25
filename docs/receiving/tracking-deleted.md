---
title: Tracking Deleted Messages
sidebar_position: 9
description: "How EmailEngine detects and tracks message deletions across IMAP accounts"
keywords:
  - deleted messages
  - IMAP EXPUNGE
  - message deletion
  - sync tracking
  - UIDVALIDITY
---

# Tracking Deleted Messages

Tracking deleted messages on an IMAP account is one of the more challenging aspects of email synchronization. While it's straightforward to detect new messages, identifying deletions requires careful handling of IMAP's complexity.

## Why Deletion Tracking is Challenging

**IMAP Connection Limitations**
- IDLE/NOOP only monitors one folder at a time
- Connection limits prevent opening one connection per folder
- Gmail heavily limits simultaneous connections (3-15 connections typically)

**Reconnection Issues**
- Network interruptions cause disconnects
- Forced logouts lose notification state
- No notifications for events while disconnected

**Sequence Number Complexity**
- EXPUNGE notifications use sequence numbers, not UIDs
- Sequence numbers change as messages are deleted
- Must maintain accurate sequence-to-UID mapping

## How EmailEngine Tracks Deletions

EmailEngine solves these challenges by:

1. **Monitoring folder state** via UIDNEXT and message counts
2. **Using MODSEQ** when available (CONDSTORE extension)
3. **Maintaining UID sequences** for accurate tracking
4. **Sending webhooks** when deletions are detected

### Detection Methods

**Method 1: UIDNEXT + Message Count**

The UIDNEXT value predicts the next message UID (usually highest UID + 1). By tracking both UIDNEXT and message count:

```
Before: messages=100, UIDNEXT=150
After:  messages=95, UIDNEXT=150

Conclusion: 5 messages were deleted
```

If both values are unchanged, no messages were deleted.

**Method 2: MODSEQ (if supported)**

The MODSEQ value increments whenever folder content changes:

```
Before: MODSEQ=12345
After:  MODSEQ=12345

Conclusion: No changes (no deletions)
```

Note: MODSEQ changes for any modification (flags, deletions, additions), so it's more sensitive than UIDNEXT.

**Method 3: UID Sequence Comparison**

Compare the list of message UIDs before and after:

```
Before: [100, 101, 102, 103, 104, 105]
After:  [100, 102, 104, 105, 106]

Deleted: [101, 103]
Added: [106]
```

## Deletion Webhooks

### messageDeleted Event

When EmailEngine detects a deleted message, it sends a [`messageDeleted` webhook](/docs/reference/webhook-events):

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "example",
  "date": "2025-01-15T10:30:00.000Z",
  "path": "INBOX",
  "event": "messageDeleted",
  "data": {
    "id": "AAAAAQAAAeE",
    "uid": 12345
  }
}
```

**Important Fields:**
- `path` - Folder where message was deleted (at root level, not inside data)
- `data.id` - EmailEngine's message ID (now deleted)
- `data.uid` - IMAP UID of deleted message

:::note Provider Differences
The payload varies by account type:
- **IMAP accounts**: `data` contains `id` and `uid`
- **Gmail API accounts**: `data` contains `id`, `threadId`, `flags`, `labels`, and `category`
- **Outlook API accounts**: `data` contains only `id`
:::

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

A message might appear "deleted" from one folder because it was moved to another. The ability to detect moves depends on the account type:

**Gmail API accounts:** The `messageDeleted` event includes the Gmail message `id`, which you can use to correlate with a subsequent `messageNew` event (Gmail assigns the same ID to moved messages).

**IMAP accounts:** The `messageDeleted` event only includes `id` (packed UID) and `uid`, which are folder-specific. When a message is moved, it gets a new UID in the destination folder, so direct correlation is not possible. However, if your IMAP server supports the OBJECTID extension, you can use `emailId` from `messageNew` events to detect moves by matching against previously stored message data.

```javascript
// Track moved messages using emailId from messageNew events
// Note: This requires storing emailId when messages are first received
const recentDeletions = new Map();

async function handleMessageDeleted(event) {
  const { path, data } = event;

  // Look up the emailId from our stored message data
  const storedMessage = await db.messages.findOne({ emailEngineId: data.id });
  if (storedMessage && storedMessage.emailId) {
    // Store deletion temporarily using emailId
    recentDeletions.set(storedMessage.emailId, {
      id: data.id,
      path: path,
      emailId: storedMessage.emailId,
      timestamp: Date.now()
    });

    // Clean up old entries after 60 seconds
    setTimeout(() => {
      recentDeletions.delete(storedMessage.emailId);
    }, 60000);
  }

  await updateMessageStatus(data.id, 'deleted');
}

async function handleMessageNew(event) {
  const { path, data } = event;

  // Check if this is a moved message (using emailId from the new message)
  const recentDeletion = data.emailId ? recentDeletions.get(data.emailId) : null;

  if (recentDeletion) {
    console.log('Message was moved, not deleted');
    console.log(`From: ${recentDeletion.path}`);
    console.log(`To: ${path}`);

    // Update status to moved
    await updateMessageStatus(recentDeletion.id, 'moved');
    await updateMessageLocation(data.emailId, path, data.id);

    recentDeletions.delete(data.emailId);
  } else {
    // Genuinely new message
    await createMessage(data);
  }
}

async function updateMessageLocation(emailId, newPath, newId) {
  await db.messages.update(
    { emailId: emailId },
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

:::note
The `emailId` field is only available if the IMAP server supports the OBJECTID extension (RFC 8474). Not all IMAP servers support this extension.
:::

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
await db.messages.createIndex({ status: 1, deletedAt: 1 });
await db.deletionLog.createIndex({ accountId: 1, deletedAt: -1 });
```

## See Also

- [IMAP indexers](/docs/accounts/imap-indexers) - The fast indexer does not detect deletions at all
- [messageDeleted](/docs/webhooks/messagedeleted) - The event and its payload
- [Message operations](/docs/receiving/message-operations) - Deleting a message yourself
- [mailboxReset](/docs/webhooks/mailboxreset) - When the server invalidates the index rather than the message
