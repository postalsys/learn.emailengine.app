---
title: "messageDeleted"
sidebar_position: 4
description: "Webhook event triggered when a previously present email is no longer found in a folder"
---

# messageDeleted

The `messageDeleted` webhook event is triggered when EmailEngine detects that a previously tracked email has been removed from a mailbox folder. This event helps you keep external systems synchronized with mailbox state changes.

## When This Event is Triggered

The `messageDeleted` event fires when:

- A message is permanently deleted from a folder
- A message is moved to another folder (the source folder triggers deletion)
- A message is expunged from the server
- The IMAP EXPUNGE command is issued for the message
- Gmail: A message loses all labels or is moved to Trash
- Outlook: A message is deleted via the Graph API

The event is triggered after EmailEngine confirms the message is no longer present in the monitored folder.

:::note Requires the full indexer
On IMAP accounts, deletions are detected only under the full [indexer](/docs/accounts/imap-indexers), which is the default. The fast indexer tracks the highest UID it has seen and nothing else, so it never notices a message going away.
:::

## Common Use Cases

- **Database synchronization** - Remove or archive records when emails are deleted
- **Search index updates** - Remove deleted messages from your search indexes
- **CRM integration** - Update ticket or contact records when associated emails are deleted
- **Audit logging** - Track message deletions for compliance purposes
- **Storage cleanup** - Remove cached attachments or processed data for deleted messages
- **Analytics** - Track deletion patterns and user behavior

## Payload Schema

### Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `serviceUrl` | string | No | The configured EmailEngine service URL |
| `account` | string | Yes | Account ID where the message was deleted |
| `date` | string | Yes | ISO 8601 timestamp when the webhook was generated |
| `path` | string | Yes | Mailbox folder path where the message was located (e.g., "INBOX") |
| `specialUse` | string | No | Special use flag of the folder (e.g., "\Inbox", "\Sent", "\Trash") |
| `event` | string | Yes | Event type, always "messageDeleted" for this event |
| `data` | object | Yes | Message identification data (see below) |

### Message Data Fields (`data` object)

The `messageDeleted` event includes minimal data since the message content is no longer available:

#### IMAP Accounts

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | EmailEngine's unique message ID (base64url encoded) |
| `uid` | number | Yes | IMAP UID of the deleted message within the folder |

#### Gmail API Accounts

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Gmail message ID |
| `threadId` | string | No | Gmail thread ID the message belonged to |
| `flags` | array | No | Last known flags before deletion |
| `labels` | array | No | Last known Gmail labels before deletion |
| `category` | string | No | Last known category (e.g., "primary", "social") |

#### Microsoft Graph (Outlook) Accounts

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Outlook message ID |

## Example Payloads

### IMAP Account

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T06:44:14.660Z",
  "path": "INBOX",
  "specialUse": "\\Inbox",
  "event": "messageDeleted",
  "data": {
    "id": "AAAADAAABy4",
    "uid": 1838
  }
}
```

### Gmail API Account

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "gmail-user",
  "date": "2025-10-17T08:15:22.123Z",
  "path": "[Gmail]/All Mail",
  "event": "messageDeleted",
  "data": {
    "id": "18b5c7d8e9f01234",
    "threadId": "18b5c7d8e9f01234",
    "flags": ["\\Seen"],
    "labels": ["INBOX", "IMPORTANT"],
    "category": "primary"
  }
}
```

### Microsoft Outlook Account

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "outlook-user",
  "date": "2025-10-17T09:30:45.789Z",
  "path": "Inbox",
  "event": "messageDeleted",
  "data": {
    "id": "AAMkADI2NGVhZTVlLTI1OGItNDUwZS05ZDVkLWQzN2E2MDUyYzc3YQBGAAAAAAI"
  }
}
```

## Handling the Event

### Basic Handler

```javascript
async function handleMessageDeleted(event) {
  const { account, path, data } = event;

  console.log(`Message deleted from ${account}:`);
  console.log(`  Message ID: ${data.id}`);
  console.log(`  Folder: ${path}`);
  if (data.uid) {
    console.log(`  UID: ${data.uid}`);
  }

  // Remove from your database
  await removeMessageFromDatabase(account, data.id);
}
```

### Database Synchronization

```javascript
async function handleMessageDeleted(event) {
  const { account, data } = event;

  try {
    // Remove from primary database
    await db.messages.delete({
      where: {
        accountId: account,
        messageId: data.id
      }
    });

    // Remove from search index
    await searchIndex.delete(`${account}:${data.id}`);

    // Clean up any cached attachments
    await cache.deletePattern(`attachments:${account}:${data.id}:*`);

    console.log(`Cleaned up message ${data.id} for account ${account}`);
  } catch (err) {
    console.error('Failed to clean up deleted message:', err);
    throw err; // Retry the webhook
  }
}
```

### Audit Logging

```javascript
async function handleMessageDeleted(event) {
  const { account, path, date, data, eventId } = event;

  // Log deletion for compliance
  await auditLog.create({
    eventId,
    timestamp: new Date(date),
    account,
    action: 'message_deleted',
    folder: path,
    messageId: data.id,
    uid: data.uid || null,
    metadata: {
      threadId: data.threadId,
      labels: data.labels,
      lastFlags: data.flags
    }
  });
}
```

## Important Considerations

### Message Content is Unavailable

When a `messageDeleted` event is received, the message content is no longer available on the server. The webhook only provides identification data (`id`, `uid`, `threadId`) to help you locate and remove records in your external systems.

If you need message content for deletion processing (e.g., archiving before deletion), you should:
1. Store message metadata when handling `messageNew` events
2. Reference that stored data when processing deletions

### Deletion vs. Move

When a message is moved between folders:
1. A `messageDeleted` event fires for the source folder
2. A `messageNew` event fires for the destination folder

To distinguish between permanent deletion and folder moves, you can:
- Check if a `messageNew` event arrives shortly after with the same `messageId`
- Track message IDs across your system to detect moves

### Gmail Label Changes

For Gmail API accounts, removing the last label from a message (except for Trash/Spam) triggers a deletion event. The `labels` field in the payload shows the last known labels before deletion.

### Idempotency

Handle deletion events idempotently since webhooks may be retried:

```javascript
async function handleMessageDeleted(event) {
  const { account, data } = event;

  // Check if already processed
  const exists = await db.messages.findUnique({
    where: {
      accountId: account,
      messageId: data.id
    }
  });

  if (!exists) {
    console.log(`Message ${data.id} already deleted, skipping`);
    return;
  }

  await db.messages.delete({
    where: {
      accountId: account,
      messageId: data.id
    }
  });
}
```

## Related Events

- [messageNew](/docs/webhooks/messagenew) - Triggered when a new message arrives
- [messageUpdated](/docs/webhooks/messageupdated) - Triggered when flags/labels change
- [mailboxDeleted](/docs/webhooks/mailboxdeleted) - Triggered when an entire folder is deleted

## See Also

- [Webhooks Overview](/docs/webhooks/overview) - Complete webhook setup guide
- [Message Operations](/docs/receiving/message-operations) - Working with messages via API
- [Settings API](/docs/api/post-v-1-settings) - Configure webhook settings
