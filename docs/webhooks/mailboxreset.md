---
title: "mailboxReset"
sidebar_position: 14
description: "Webhook event triggered when EmailEngine rebuilds a folder's index, invalidating previously tracked message UIDs"
---

# mailboxReset

The `mailboxReset` webhook event is triggered when EmailEngine has to rebuild its index for an IMAP folder from scratch. It is rare but significant: every message UID EmailEngine previously tracked for that folder is no longer meaningful.

## When This Event is Triggered

The `mailboxReset` event fires when:

- The IMAP server reports a different UIDVALIDITY for the folder than the one EmailEngine stored (`reason: "uidValidityChange"`). Servers do this when a folder is recreated, repaired, migrated or restored from backup
- EmailEngine's own stored index for the folder is missing while the account has synced before and the server still has messages in the folder (`reason: "syncStateLost"`), for example after the folder's keys were evicted from Redis

UIDVALIDITY is the IMAP mechanism that tells a client whether previously assigned UIDs are still valid. When it changes, they are not, and the folder must be fully resynchronized. A lost local index has the same practical consequence.

:::note No message events are replayed
Rebuilding the baseline deliberately does not emit `messageNew` for the messages it re-indexes. Without that suppression, a reset on a large folder would replay the entire mailbox to your webhook endpoint. Treat `mailboxReset` itself as the signal to reconcile, and fetch the folder's messages through the API if you need them.
:::

:::note IMAP accounts only
Folder events are produced by the IMAP client. Gmail API and Microsoft Graph accounts do not send `mailboxNew`, `mailboxDeleted` or `mailboxReset`.
:::

## Common Use Cases

- **Full resync trigger** - Initiate a complete resynchronization of your local message cache
- **Database cleanup** - Clear cached message data for the affected folder since UIDs are invalid
- **Search index rebuild** - Mark the folder's search index for rebuild
- **Audit logging** - Track mailbox reset events for operational monitoring
- **Alert systems** - Notify administrators about unusual mailbox resets that may indicate server issues

## Payload Schema

### Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `serviceUrl` | string or null | Yes | The configured EmailEngine service URL. `null` when the `serviceUrl` setting is empty |
| `account` | string | Yes | Account ID the folder belongs to |
| `date` | string | Yes | ISO 8601 timestamp when the webhook was generated |
| `path` | string | Yes | Full path of the folder that was reset, for example `INBOX` |
| `specialUse` | string | No | Special-use flag of the folder, for example `\Inbox`. Present only when the folder has one |
| `event` | string | Yes | Always `mailboxReset` |
| `data` | object | Yes | Reset details |

### Reset Data Fields (`data` object)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `path` | string | Yes | Full folder path, the same value as the top-level `path` |
| `name` | string | Yes | Display name of the folder, the last segment of the path |
| `specialUse` | string or boolean | Yes | Special-use flag such as `\Inbox`, or `false` when the folder has none |
| `uidValidity` | string or boolean | Yes | The folder's current UIDVALIDITY as a string, or `false` if the server did not report a usable one |
| `prevUidValidity` | string or boolean | No | The UIDVALIDITY EmailEngine had stored, as a string, or `false` if the stored value was not usable. Present only with `reason: "uidValidityChange"` |
| `reason` | string | Yes | Why the folder was reset, see below |

There is no event ID in the body. EmailEngine sends it in the `X-EE-Wh-Event-Id` request header. See [Delivery and Retries](/docs/webhooks/overview#delivery-and-retries).

### Reset Reasons

| Reason | Meaning |
|--------|---------|
| `uidValidityChange` | The server issued a new UIDVALIDITY for the folder, which invalidates every UID EmailEngine had stored |
| `syncStateLost` | EmailEngine's own index for the folder was missing and had to be rebuilt from the server |

Both mean the same thing for your application: the message IDs you previously stored for this folder no longer refer to those messages. Branch on `reason` only if you want to distinguish a server-side renumbering from a local index rebuild.

## Example Payload

### IMAP Account - Inbox Reset

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T14:22:33.456Z",
  "path": "INBOX",
  "specialUse": "\\Inbox",
  "event": "mailboxReset",
  "data": {
    "path": "INBOX",
    "name": "INBOX",
    "specialUse": "\\Inbox",
    "uidValidity": "1697556153",
    "prevUidValidity": "1695234567",
    "reason": "uidValidityChange"
  }
}
```

### Custom Folder Reset

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "support-inbox",
  "date": "2025-10-17T15:45:12.789Z",
  "path": "Support/Tickets",
  "event": "mailboxReset",
  "data": {
    "path": "Support/Tickets",
    "name": "Tickets",
    "specialUse": false,
    "uidValidity": "1697560312",
    "prevUidValidity": "1690234567",
    "reason": "uidValidityChange"
  }
}
```

### Lost Local Index

When the stored index was missing, there is no previous UIDVALIDITY to report and `prevUidValidity` is omitted:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "recovered-account",
  "date": "2025-10-17T16:30:00.000Z",
  "path": "Archive/2024",
  "event": "mailboxReset",
  "data": {
    "path": "Archive/2024",
    "name": "2024",
    "specialUse": false,
    "uidValidity": "1697564200",
    "reason": "syncStateLost"
  }
}
```

## Handling the Event

### Basic Handler

```javascript
async function handleMailboxReset(event) {
  const { account, path, data } = event;

  console.log(`Mailbox reset detected for ${account}:`);
  console.log(`  Folder: ${path}`);
  console.log(`  Reason: ${data.reason}`);
  console.log(`  New UIDVALIDITY: ${data.uidValidity}`);
  console.log(`  Previous UIDVALIDITY: ${data.prevUidValidity || 'unknown'}`);

  // Trigger full resync for this folder
  await triggerFolderResync(account, path);
}
```

### Database Cleanup

```javascript
async function handleMailboxReset(event) {
  const { account, path, data } = event;

  try {
    // Clear all cached messages for this folder
    // UIDs are no longer valid after UIDVALIDITY change
    const deletedCount = await db.messages.deleteMany({
      where: {
        accountId: account,
        folder: path
      }
    });

    console.log(`Cleared ${deletedCount} cached messages for ${account}/${path}`);

    // Update folder metadata with new UIDVALIDITY
    await db.folders.upsert({
      where: {
        accountId_path: { accountId: account, path }
      },
      update: {
        uidValidity: data.uidValidity,
        lastReset: new Date(event.date),
        syncStatus: 'pending'
      },
      create: {
        accountId: account,
        path,
        name: data.name,
        uidValidity: data.uidValidity,
        syncStatus: 'pending'
      }
    });

    // Trigger resync
    await resyncQueue.add('folder-resync', {
      account,
      path,
      reason: data.reason
    });

  } catch (err) {
    console.error('Failed to handle mailbox reset:', err);
    throw err; // Respond with an error status so EmailEngine retries the delivery
  }
}
```

### Alert on Reset

```javascript
async function handleMailboxReset(event) {
  const { account, path, date, data } = event;

  // Log the reset event
  await auditLog.create({
    timestamp: new Date(date),
    account,
    action: 'mailbox_reset',
    folder: path,
    metadata: {
      reason: data.reason,
      newUidValidity: data.uidValidity,
      prevUidValidity: data.prevUidValidity,
      folderName: data.name,
      specialUse: data.specialUse
    }
  });

  // Alert if this is a critical folder
  const criticalFolders = ['INBOX', 'Sent', 'Drafts'];
  if (criticalFolders.some(f =>
    path.toUpperCase().includes(f.toUpperCase())
  )) {
    await alertService.send({
      severity: 'warning',
      title: 'Critical Mailbox Reset Detected',
      message: `Folder ${path} on account ${account} was reset (${data.reason})`,
      details: {
        account,
        folder: path,
        previousUidValidity: data.prevUidValidity,
        newUidValidity: data.uidValidity,
        timestamp: date
      }
    });
  }
}
```

### Search Index Rebuild

```javascript
async function handleMailboxReset(event) {
  const { account, path, data } = event;

  // Delete all indexed documents for this folder
  await searchIndex.deleteByQuery({
    query: {
      bool: {
        must: [
          { term: { accountId: account } },
          { term: { folder: path } }
        ]
      }
    }
  });

  console.log(`Cleared search index for ${account}/${path}`);

  // Mark folder for reindexing
  await searchIndex.update({
    id: `folder:${account}:${path}`,
    doc: {
      uidValidity: data.uidValidity,
      needsReindex: true,
      resetAt: event.date
    },
    doc_as_upsert: true
  });
}
```

## Important Considerations

### What UIDVALIDITY Means

UIDVALIDITY is an IMAP concept that guarantees message UID uniqueness within a mailbox. When UIDVALIDITY changes:

- All previously assigned UIDs become invalid
- You cannot rely on old UID-to-message mappings
- A full folder resync is required to rebuild the message list
- Any cached message data keyed by UID should be discarded

### What EmailEngine Does on a Reset

When EmailEngine detects a UIDVALIDITY change, or finds its index for the folder missing, it:

1. Discards the stored message index for the folder
2. Records every message currently on the server as already seen, without sending `messageNew` for any of them
3. Stores the server's current state as the new baseline
4. Sends this webhook

Messages that arrive after the reset are reported with `messageNew` as usual. Nothing is sent for the messages that were in the folder at the time of the reset, so reconcile from the API rather than waiting for events.

### EmailEngine Message IDs Change Too

EmailEngine's message IDs (the `id` field on messages and in message events) are derived from the folder and the UID, so a reset invalidates them along with the UIDs. Match on the `Message-ID` header if you need to relate messages across a reset:

```javascript
async function handleMailboxReset(event) {
  const { account, path } = event;

  // Instead of deleting, mark records as needing revalidation
  await db.messages.updateMany({
    where: {
      accountId: account,
      folder: path
    },
    data: {
      uidValid: false,
      needsRevalidation: true
    }
  });

  // Then list the folder through the API and match on the Message-ID header
}
```

### Rare But Important

UIDVALIDITY changes are relatively rare in normal operation. Common causes include:

- Mail server migration
- Database repairs or corruption recovery
- Server software updates that reset counters
- Mailbox import and export operations
- Administrative maintenance

Frequent resets on one account point at a server that reassigns UIDVALIDITY on every folder open, which is a server problem rather than something the handler can work around.

## Related Events

- [mailboxNew](/docs/webhooks/mailboxnew) - Triggered when a new folder is found
- [mailboxDeleted](/docs/webhooks/mailboxdeleted) - Triggered when a folder is removed
- [messageNew](/docs/webhooks/messagenew) - Triggered for messages that arrive after the reset
- [messageDeleted](/docs/webhooks/messagedeleted) - Triggered when individual messages are deleted

## See Also

- [Webhooks Overview](/docs/webhooks/overview) - Configuring the webhook URL and the `webhookEvents` allowlist
- [Mailbox Operations](/docs/receiving/mailbox-operations) - Listing folders and their current UIDVALIDITY
- [List Messages API](/docs/api/get-v-1-account-account-messages) - Reconciling the folder's contents after a reset
- [IDs Explained](/docs/advanced/ids-explained) - How EmailEngine message IDs are built from the folder and UID
