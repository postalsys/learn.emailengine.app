---
title: "mailboxDeleted"
sidebar_position: 13
description: "Webhook event triggered when a previously tracked folder is no longer found on the mail server"
---

# mailboxDeleted

The `mailboxDeleted` webhook event is triggered when a folder that EmailEngine was tracking for an IMAP account is no longer on the mail server, or has been deleted through the API. It tells your application to drop whatever it holds for that folder.

## When This Event is Triggered

The `mailboxDeleted` event fires when:

- A folder that was in the account's stored folder listing is missing from the listing the server returns on the next sync. EmailEngine does not learn why: the user deleted it in a mail client, an administrator removed it, a retention policy purged it, or the account lost access to it
- A folder is deleted through the [Delete Mailbox API](/docs/api/delete-v-1-account-account-mailbox)

A rename is a deletion followed by a creation as far as the folder listing is concerned, so it produces `mailboxDeleted` for the old path and [`mailboxNew`](/docs/webhooks/mailboxnew) for the new one.

The event covers only folders EmailEngine knew about. A folder created and deleted between two listings is never seen. Nothing is sent when an account is deleted, paused or reconfigured, even though EmailEngine drops its folder state then.

:::note IMAP accounts only
Folder events are produced by the IMAP client. Gmail API and Microsoft Graph accounts do not send `mailboxNew`, `mailboxDeleted` or `mailboxReset`.
:::

## Common Use Cases

- **Database cleanup** - Remove cached messages and folder metadata for the deleted folder
- **Search index updates** - Delete indexed documents associated with the folder
- **UI synchronization** - Update folder trees and navigation menus in your application
- **Audit logging** - Track folder deletions for compliance or security monitoring
- **Sync state cleanup** - Clear sync markers and state data tied to the deleted folder

## Payload Schema

### Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `serviceUrl` | string or null | Yes | The configured EmailEngine service URL. `null` when the `serviceUrl` setting is empty |
| `account` | string | Yes | Account ID the folder belonged to |
| `date` | string | Yes | ISO 8601 timestamp when the webhook was generated |
| `path` | string | Yes | Full path of the deleted folder, for example `Archive/2023` |
| `specialUse` | string | No | Special-use flag of the folder, for example `\Trash`. Present only when the folder had one |
| `event` | string | Yes | Always `mailboxDeleted` |
| `data` | object | Yes | Folder details as last stored |

### Folder Data Fields (`data` object)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `path` | string | Yes | Full folder path, the same value as the top-level `path` |
| `name` | string | Yes | Display name of the folder, the last segment of the path |
| `specialUse` | string or boolean | Yes | Special-use flag such as `\Sent`, or `false` when the folder had none |

There is no event ID in the body. EmailEngine sends it in the `X-EE-Wh-Event-Id` request header. See [Delivery and Retries](/docs/webhooks/overview#delivery-and-retries).

## Example Payload

### Standard Folder Deletion

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T14:22:33.456Z",
  "path": "Projects/Completed",
  "event": "mailboxDeleted",
  "data": {
    "path": "Projects/Completed",
    "name": "Completed",
    "specialUse": false
  }
}
```

### Special Use Folder Deletion

A folder with a special-use flag carries it at both levels:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "support-inbox",
  "date": "2025-10-17T15:45:12.789Z",
  "path": "Drafts",
  "specialUse": "\\Drafts",
  "event": "mailboxDeleted",
  "data": {
    "path": "Drafts",
    "name": "Drafts",
    "specialUse": "\\Drafts"
  }
}
```

### Nested Folder Deletion

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "admin",
  "date": "2025-10-17T16:30:00.000Z",
  "path": "Archive/2024/Q1/January",
  "event": "mailboxDeleted",
  "data": {
    "path": "Archive/2024/Q1/January",
    "name": "January",
    "specialUse": false
  }
}
```

## Handling the Event

### Basic Handler

```javascript
async function handleMailboxDeleted(event) {
  const { account, path, data } = event;

  console.log(`Folder deleted for ${account}:`);
  console.log(`  Path: ${path}`);
  console.log(`  Name: ${data.name}`);
  console.log(`  Special Use: ${data.specialUse || 'none'}`);

  // Clean up any resources associated with this folder
  await cleanupFolder(account, path);
}
```

### Database Cleanup

```javascript
async function handleMailboxDeleted(event, headers) {
  const { account, path, date } = event;
  const eventId = headers['x-ee-wh-event-id'];

  try {
    // Delete all cached messages for this folder
    const deletedMessages = await db.messages.deleteMany({
      where: {
        accountId: account,
        folder: path
      }
    });

    // Delete the folder record
    await db.folders.delete({
      where: {
        accountId_path: { accountId: account, path }
      }
    });

    console.log(`Cleaned up folder ${path}: ${deletedMessages.count} messages removed`);

    // Log the deletion
    await auditLog.create({
      eventId,
      timestamp: new Date(date),
      account,
      action: 'folder_deleted',
      folder: path,
      deletedMessageCount: deletedMessages.count
    });

  } catch (err) {
    console.error('Failed to cleanup deleted folder:', err);
    throw err; // Respond with an error status so EmailEngine retries the delivery
  }
}
```

### UI Synchronization

```javascript
async function handleMailboxDeleted(event) {
  const { account, path, data } = event;

  // Broadcast to connected clients
  await websocketServer.broadcast({
    type: 'folder:deleted',
    account,
    folder: {
      path,
      name: data.name,
      specialUse: data.specialUse
    }
  });

  // Remove from folder cache
  await folderCache.delete(`${account}:${path}`);

  // If any users have this folder selected, redirect them
  const affectedSessions = await sessionStore.findByActiveFolder(account, path);
  for (const session of affectedSessions) {
    await websocketServer.sendToSession(session.id, {
      type: 'folder:redirect',
      message: 'The folder you were viewing has been deleted',
      redirectTo: 'INBOX'
    });
  }
}
```

### Search Index Cleanup

```javascript
async function handleMailboxDeleted(event) {
  const { account, path } = event;

  // Delete all indexed documents for this folder
  const result = await searchIndex.deleteByQuery({
    query: {
      bool: {
        must: [
          { term: { accountId: account } },
          { term: { folder: path } }
        ]
      }
    }
  });

  console.log(`Removed ${result.deleted} documents from search index for ${account}/${path}`);

  // Also delete the folder metadata document
  await searchIndex.delete({
    id: `folder:${account}:${path}`,
    ignore: [404] // Don't error if not found
  });
}
```

### Alert on Special-Use Folder Deletion

```javascript
async function handleMailboxDeleted(event, headers) {
  const { account, path, date, data } = event;
  const eventId = headers['x-ee-wh-event-id'];

  // Servers normally refuse to delete these; one disappearing is worth a look
  if (data.specialUse) {
    await alertService.send({
      severity: 'warning',
      title: 'Special-use folder deleted',
      message: `Folder "${path}" (${data.specialUse}) was deleted on account ${account}`,
      details: {
        account,
        folder: path,
        folderName: data.name,
        specialUse: data.specialUse,
        timestamp: date,
        eventId
      }
    });
  }

  // Proceed with normal cleanup
  await cleanupFolder(account, path);
}
```

### Cleanup with Child Folder Handling

```javascript
async function handleMailboxDeleted(event) {
  const { account, path } = event;

  // Each child folder gets its own event, but cleaning them up here as well
  // keeps the local state consistent if a child event is delayed or lost
  const deletedFolders = await db.folders.deleteMany({
    where: {
      accountId: account,
      OR: [
        { path: path },
        { path: { startsWith: `${path}/` } }
      ]
    }
  });

  // Delete messages in this folder and child folders
  const deletedMessages = await db.messages.deleteMany({
    where: {
      accountId: account,
      OR: [
        { folder: path },
        { folder: { startsWith: `${path}/` } }
      ]
    }
  });

  console.log(`Deleted ${deletedFolders.count} folders and ${deletedMessages.count} messages`);
}
```

## Important Considerations

### Folder vs Message Deletion

The `mailboxDeleted` event means the folder itself is gone. This is different from messages being deleted within a folder:

- **mailboxDeleted** - The entire folder no longer exists
- **messageDeleted** - Individual messages removed from a folder that still exists

When a folder is deleted, EmailEngine discards its message index without sending `messageDeleted` for the messages it contained. Treat `mailboxDeleted` as covering all of them.

### Rename Operations

EmailEngine matches folders by path and does not detect renames. Renaming a folder produces:

1. `mailboxDeleted` for the old path
2. `mailboxNew` for the new path, after its first sync

If you need to follow renames, match on folder contents rather than assuming path continuity. The messages of the renamed folder are indexed afresh under the new path, and [`messageNew`](/docs/webhooks/messagenew) is not sent for them.

### Timing and Ordering

The event is sent when EmailEngine notices the folder is missing, which is on the next folder listing after the deletion, not at the moment of deletion:

- The `date` field is when the webhook was generated, not when the folder was deleted
- When a folder with subfolders is deleted, each subfolder that disappears from the listing gets its own event
- Events for several folders are queued together and can be delivered in any order

### Data Retention

Before deleting cached data, consider whether you need to retain any information for audit compliance, recovery, analytics or legal holds:

```javascript
async function handleMailboxDeleted(event) {
  const { account, path, date } = event;

  // Archive before deleting
  const messages = await db.messages.findMany({
    where: { accountId: account, folder: path }
  });

  if (messages.length > 0) {
    await archiveService.archiveFolderContents({
      account,
      folder: path,
      messages,
      deletedAt: date,
      reason: 'folder_deleted'
    });
  }

  // Then proceed with cleanup
  await db.messages.deleteMany({
    where: { accountId: account, folder: path }
  });
}
```

## Related Events

- [mailboxNew](/docs/webhooks/mailboxnew) - Triggered when a new folder is found
- [mailboxReset](/docs/webhooks/mailboxreset) - Triggered when a folder's index is rebuilt
- [messageDeleted](/docs/webhooks/messagedeleted) - Triggered when individual messages are deleted
- [accountDeleted](/docs/webhooks/accountdeleted) - Triggered when an entire account is removed

## See Also

- [Webhooks Overview](/docs/webhooks/overview) - Configuring the webhook URL and the `webhookEvents` allowlist
- [Mailbox Operations](/docs/receiving/mailbox-operations) - Listing, creating, renaming and deleting folders
- [List Mailboxes API](/docs/api/get-v-1-account-account-mailboxes) - Reading the current folder listing
- [Delete Mailbox API](/docs/api/delete-v-1-account-account-mailbox) - Deleting a folder, which also triggers this event
