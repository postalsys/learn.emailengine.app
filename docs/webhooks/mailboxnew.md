---
title: "mailboxNew"
sidebar_position: 12
description: "Webhook event triggered when a new folder is discovered on the mail server"
---

# mailboxNew

The `mailboxNew` webhook event is triggered when EmailEngine finds a folder on an IMAP account's mail server that was not in its stored folder listing, and has finished the folder's first sync. It tells your application that a folder exists and that message events for it can follow.

## When This Event is Triggered

The `mailboxNew` event fires when a folder appears in the server's folder listing that EmailEngine had not stored before, and its first sync completes. That covers:

- A folder created in a mail client, in webmail, by an administrator, or through the [Create Mailbox API](/docs/api/post-v-1-account-account-mailbox)
- The new path of a renamed folder, which the listing shows as a deletion plus a creation
- A folder that became visible, for example after a permission change on a shared namespace
- Every folder of a newly added account, and every folder again after a [flush](/docs/api/put-v-1-account-account-flush), because the stored listing starts empty

The event is sent after the folder's first sync, so by the time it arrives the folder can be listed and its messages fetched. Messages that were already in the folder are indexed as the baseline and do not produce [`messageNew`](/docs/webhooks/messagenew).

:::note IMAP accounts only
Folder events are produced by the IMAP client. Gmail API and Microsoft Graph accounts do not send `mailboxNew`, `mailboxDeleted` or `mailboxReset`.
:::

## Common Use Cases

- **Folder tree synchronization** - Update folder lists and navigation menus in your application
- **Database initialization** - Create folder metadata records for the new folder
- **Search index setup** - Initialize search index structures for the new folder
- **Subscription management** - Start watching folders that match a pattern
- **Audit logging** - Track folder creation for compliance or security monitoring
- **User notifications** - Alert users when new folders appear in their accounts

## Payload Schema

### Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `serviceUrl` | string or null | Yes | The configured EmailEngine service URL. `null` when the `serviceUrl` setting is empty |
| `account` | string | Yes | Account ID the folder belongs to |
| `date` | string | Yes | ISO 8601 timestamp when the webhook was generated |
| `path` | string | Yes | Full path of the new folder, for example `Projects/Active` |
| `specialUse` | string | No | Special-use flag of the folder, for example `\Archive`. Present only when the folder has one |
| `event` | string | Yes | Always `mailboxNew` |
| `data` | object | Yes | Folder details |

### Folder Data Fields (`data` object)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `path` | string | Yes | Full folder path, the same value as the top-level `path` |
| `name` | string | Yes | Display name of the folder, the last segment of the path |
| `specialUse` | string or boolean | Yes | Special-use flag such as `\Sent`, or `false` when the folder has none |
| `uidValidity` | string | Yes | The folder's IMAP UIDVALIDITY, as a string |

There is no event ID in the body. EmailEngine sends it in the `X-EE-Wh-Event-Id` request header. See [Delivery and Retries](/docs/webhooks/overview#delivery-and-retries).

## Example Payload

### Standard Folder Creation

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T14:22:33.456Z",
  "path": "Projects/Active",
  "event": "mailboxNew",
  "data": {
    "path": "Projects/Active",
    "name": "Active",
    "specialUse": false,
    "uidValidity": "1697551353"
  }
}
```

### Special Use Folder

A folder with a special-use flag carries it at both levels:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "support-inbox",
  "date": "2025-10-17T15:45:12.789Z",
  "path": "Archive",
  "specialUse": "\\Archive",
  "event": "mailboxNew",
  "data": {
    "path": "Archive",
    "name": "Archive",
    "specialUse": "\\Archive",
    "uidValidity": "1697555112"
  }
}
```

### Nested Folder Creation

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "admin",
  "date": "2025-10-17T16:30:00.000Z",
  "path": "Archive/2024/Q4/December",
  "event": "mailboxNew",
  "data": {
    "path": "Archive/2024/Q4/December",
    "name": "December",
    "specialUse": false,
    "uidValidity": "1697558200"
  }
}
```

## Handling the Event

### Basic Handler

```javascript
async function handleMailboxNew(event) {
  const { account, path, data } = event;

  console.log(`New folder for ${account}:`);
  console.log(`  Path: ${path}`);
  console.log(`  Name: ${data.name}`);
  console.log(`  Special Use: ${data.specialUse || 'none'}`);
  console.log(`  UIDVALIDITY: ${data.uidValidity}`);

  // Initialize resources for this folder
  await initializeFolder(account, path, data);
}
```

### Database Initialization

```javascript
async function handleMailboxNew(event, headers) {
  const { account, path, date, data } = event;
  const eventId = headers['x-ee-wh-event-id'];

  try {
    // Create folder record
    await db.folders.create({
      data: {
        accountId: account,
        path: path,
        name: data.name,
        specialUse: data.specialUse || null,
        uidValidity: data.uidValidity,
        createdAt: new Date(date),
        eventId: eventId
      }
    });

    console.log(`Initialized folder ${path} for account ${account}`);

    // Log the creation
    await auditLog.create({
      eventId,
      timestamp: new Date(date),
      account,
      action: 'folder_created',
      folder: path,
      folderName: data.name,
      specialUse: data.specialUse
    });

  } catch (err) {
    if (err.code === 'P2002') {
      // Folder already exists - a redelivered webhook, or a flush that re-announced the folder
      console.log(`Folder ${path} already exists, skipping`);
      return;
    }
    console.error('Failed to initialize folder:', err);
    throw err; // Respond with an error status so EmailEngine retries the delivery
  }
}
```

### UI Synchronization

```javascript
async function handleMailboxNew(event) {
  const { account, path, data } = event;

  // Broadcast to connected clients
  await websocketServer.broadcast({
    type: 'folder:created',
    account,
    folder: {
      path,
      name: data.name,
      specialUse: data.specialUse,
      uidValidity: data.uidValidity
    }
  });

  // Add to folder cache
  await folderCache.set(`${account}:${path}`, {
    path,
    name: data.name,
    specialUse: data.specialUse,
    uidValidity: data.uidValidity
  });

  // Invalidate folder list cache
  await folderCache.delete(`${account}:folder-list`);
}
```

### Watching Folders that Match a Pattern

```javascript
async function handleMailboxNew(event) {
  const { account, path, data } = event;

  // Define patterns for folders your application processes
  const watchPatterns = [
    /^Projects\//,
    /^Clients\//,
    /^Archive\/\d{4}/
  ];

  const shouldWatch = watchPatterns.some(pattern => pattern.test(path));

  if (shouldWatch) {
    await watchList.add({
      account,
      folder: path,
      events: ['messageNew', 'messageDeleted', 'messageUpdated']
    });

    console.log(`Watching folder ${path} for account ${account}`);
  }

  // Always track the folder
  await folderStore.add(account, {
    path,
    name: data.name,
    specialUse: data.specialUse,
    watched: shouldWatch
  });
}
```

### Search Index Initialization

```javascript
async function handleMailboxNew(event) {
  const { account, path, date, data } = event;

  // Create folder metadata document in search index
  await searchIndex.create({
    id: `folder:${account}:${path}`,
    body: {
      type: 'folder',
      accountId: account,
      path: path,
      name: data.name,
      specialUse: data.specialUse || null,
      uidValidity: data.uidValidity,
      createdAt: date
    }
  });

  console.log(`Search index initialized for folder ${account}/${path}`);
}
```

### Hierarchical Folder Setup

```javascript
async function handleMailboxNew(event) {
  const { account, path, data } = event;

  // Parse folder hierarchy
  const pathParts = path.split('/');
  const depth = pathParts.length;
  const parentPath = pathParts.slice(0, -1).join('/') || null;

  // Create folder record with hierarchy info
  await db.folders.create({
    data: {
      accountId: account,
      path: path,
      name: data.name,
      specialUse: data.specialUse || null,
      uidValidity: data.uidValidity,
      depth: depth,
      parentPath: parentPath
    }
  });

  // Update parent folder if exists
  if (parentPath) {
    await db.folders.updateMany({
      where: {
        accountId: account,
        path: parentPath
      },
      data: {
        hasChildren: true
      }
    });
  }

  console.log(`Folder ${path} added at depth ${depth}`);
}
```

The path separator is whatever the server uses. `/` is common, but Microsoft Exchange and some other servers use `.` or `\`. The [List Mailboxes API](/docs/api/get-v-1-account-account-mailboxes) returns each folder's `delimiter`.

## Important Considerations

### Initial Account Sync

When an account is added, or after a flush, every folder is new to EmailEngine, so it sends one `mailboxNew` per folder as each finishes its first sync. Expect a burst for a newly added account, and treat a `mailboxNew` for a folder you already know about as a re-announcement rather than an error:

```javascript
async function handleMailboxNew(event) {
  const { account, path } = event;

  const known = await db.folders.findFirst({
    where: { accountId: account, path }
  });

  if (known) {
    // Re-announced after a flush, or a redelivery. Refresh what changed
    await db.folders.update({
      where: { id: known.id },
      data: { uidValidity: event.data.uidValidity, specialUse: event.data.specialUse || null }
    });
    return;
  }

  await processNewFolder(account, path, event.data);
}
```

### Rename Operations

EmailEngine matches folders by path and does not detect renames. Renaming a folder produces:

1. `mailboxDeleted` for the old path
2. `mailboxNew` for the new path, after its first sync

If you need to follow renames, match on folder contents rather than assuming path continuity. The messages of the renamed folder are indexed afresh under the new path, and `messageNew` is not sent for them.

### UIDVALIDITY

`uidValidity` identifies the folder's current numbering of messages. If the server assigns the folder a new UIDVALIDITY later, every UID EmailEngine stored for it becomes invalid; EmailEngine then rebuilds its index and sends [`mailboxReset`](/docs/webhooks/mailboxreset) with the old and new values. Store the value from this event if you want to compare:

```javascript
async function handleMailboxNew(event) {
  const { account, path, data } = event;

  // Store UIDVALIDITY for future comparison
  await folderState.set(account, path, {
    uidValidity: data.uidValidity,
    lastSync: new Date()
  });
}
```

### Special Use Folders

The `specialUse` field carries the IMAP special-use attribute defined in RFC 6154, when the server advertises one:

| Value | Description |
|-------|-------------|
| `\All` | All messages (virtual folder) |
| `\Archive` | Archive folder |
| `\Drafts` | Draft messages |
| `\Flagged` | Flagged or starred messages |
| `\Junk` | Spam or junk folder |
| `\Sent` | Sent messages |
| `\Trash` | Deleted messages |

`\Inbox` is used for the INBOX folder, which the server does not flag but which has a fixed role.

```javascript
async function handleMailboxNew(event) {
  const { account, path, data } = event;

  // Map special use folders for the application
  if (data.specialUse) {
    await accountSettings.update(account, {
      [`${data.specialUse.replace('\\', '').toLowerCase()}Folder`]: path
    });

    console.log(`Mapped ${data.specialUse} to ${path} for account ${account}`);
  }

  // Regular folder initialization
  await initializeFolder(account, path, data);
}
```

### Idempotency

A failed or timed-out delivery is retried, so the same event can arrive more than once. Deduplicate on the event ID header:

```javascript
async function handleMailboxNew(event, headers) {
  const { account, path } = event;
  const eventId = headers['x-ee-wh-event-id'];

  // Check if we've already processed this event
  const processed = await eventLog.exists(eventId);
  if (processed) {
    console.log(`Event ${eventId} already processed, skipping`);
    return;
  }

  // Process the event
  await processNewFolder(account, path, event.data);

  // Mark as processed
  await eventLog.create({
    eventId,
    processedAt: new Date()
  });
}
```

## Related Events

- [mailboxDeleted](/docs/webhooks/mailboxdeleted) - Triggered when a folder is removed
- [mailboxReset](/docs/webhooks/mailboxreset) - Triggered when a folder's index is rebuilt
- [messageNew](/docs/webhooks/messagenew) - Triggered when new messages arrive in a folder
- [accountInitialized](/docs/webhooks/accountinitialized) - Triggered when the initial account sync completes

## See Also

- [Webhooks Overview](/docs/webhooks/overview) - Configuring the webhook URL and the `webhookEvents` allowlist
- [Mailbox Operations](/docs/receiving/mailbox-operations) - Listing, creating, renaming and deleting folders
- [List Mailboxes API](/docs/api/get-v-1-account-account-mailboxes) - Reading the current folder listing, with delimiters and special-use flags
- [Create Mailbox API](/docs/api/post-v-1-account-account-mailbox) - Creating a folder, which also triggers this event
