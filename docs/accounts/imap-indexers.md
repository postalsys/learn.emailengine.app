---
title: IMAP Indexers
sidebar_position: 11
description: Understanding full and fast indexing strategies for IMAP accounts
---

# IMAP Indexers

EmailEngine uses an indexing strategy to track email changes in IMAP accounts. The indexer determines how EmailEngine detects new, updated, and deleted messages, affecting both webhook notifications and Redis storage usage.

:::note IMAP Only
Indexers apply to accounts that sync over IMAP: password accounts and OAuth2 accounts whose application uses the **IMAP and SMTP** base scope. Gmail API and MS Graph accounts use the change detection mechanisms of those APIs and have no indexer setting.
:::

## Indexing Strategies

The setting is `imapIndexer` and accepts two values, `full` and `fast`. The [OpenAPI spec](/docs/api-reference/openapi-spec) describes them as:

- `full`: Track every change, including deletions. Slower but complete
- `fast`: Detect new messages only. Cheaper, but deletions and flag changes are missed

### Full Indexer (Default)

The full indexer maintains a complete reference of all emails in each mailbox, stored in Redis.

**How it works:**
- Stores an entry for every message in each folder: UID, flags, MODSEQ, internal date, and for Gmail the email ID and labels
- Compares the current mailbox state against the stored entries on each sync
- Acts on untagged `EXPUNGE`, `VANISHED`, and `FETCH` responses from the server, so deletions and flag changes are detected as they happen

**Capabilities:**

| Event | Detected |
|-------|----------|
| New messages | Yes |
| Deleted messages | Yes |
| Flag changes (read/unread, starred, etc.) | Yes |
| Message moves between folders | Yes (as delete + new) |

**Trade-offs:**
- Higher Redis memory usage (one entry per message)
- Slower initial sync (every message is indexed)
- Complete change detection

**Best for:**
- Applications that need to track all email changes
- CRM integrations requiring deletion and flag sync
- Full mailbox synchronization use cases

### Fast Indexer

The fast indexer only tracks the next expected UID (`UIDNEXT`) of each mailbox.

**How it works:**
- Stores the last known `UIDNEXT` for each folder
- Detects new messages by looking for UIDs at or above the stored value
- Ignores untagged `EXPUNGE` and `FETCH` responses, because there is no per-message index to compare them against

**Capabilities:**

| Event | Detected |
|-------|----------|
| New messages | Yes |
| Deleted messages | No |
| Flag changes (read/unread, starred, etc.) | No |
| Message moves between folders | Partial (new in destination only) |

**Trade-offs:**
- Minimal Redis memory usage
- Faster initial sync (no per-message fetch)
- Limited change detection

**Best for:**
- Processing pipelines that only need new emails
- High-volume accounts where storage is a concern
- Feed-forward processing (AI analysis, archival)

## Comparison

| Feature | Full Indexer | Fast Indexer |
|---------|--------------|--------------|
| `messageNew` webhooks | Yes | Yes |
| `messageDeleted` webhooks | Yes | No |
| `messageUpdated` webhooks | Yes | No |
| Redis storage | One entry per message | One value per folder |
| Initial sync | Fetches every message | Reads the folder status only |

## Configuration

### Global Default

Set the default indexer for new IMAP accounts with the [settings API](/docs/api/post-v-1-settings):

```bash
curl -X POST https://emailengine.example.com/v1/settings \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "imapIndexer": "full"
  }'
```

Or in the admin interface: **Configuration** > **Email Processing**, card **IMAP Processing**, field **Indexing Method**.

:::info New Accounts Only
The indexer is copied onto the account when the account is created: an account registered without `imapIndexer` gets the global value at that moment, and keeps it. Changing the global default afterwards affects only accounts created later. To move an existing account to a different indexer, use the [flush API](/docs/api/put-v-1-account-account-flush) described below.
:::

### Per-Account Setting

Set the indexer when creating an account:

```bash
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user123",
    "imap": {
      "host": "imap.example.com",
      "port": 993,
      "secure": true,
      "auth": {
        "user": "user@example.com",
        "pass": "password"
      }
    },
    "imapIndexer": "fast"
  }'
```

### Changing Indexer for Existing Account

Use the [flush API](/docs/api/put-v-1-account-account-flush) to change the indexer for an existing account:

```bash
curl -X PUT https://emailengine.example.com/v1/account/user123/flush \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "flush": true,
    "imapIndexer": "full"
  }'
```

The response is `{"flush": true}`. The account is paused, its folder listing and per-folder index are deleted from Redis, the new indexer is stored, and the account resumes and rebuilds the index with the new strategy.

:::warning Index Reset
A flush rebuilds the index from scratch. During re-indexing:
- The account shows as `syncing` until the rebuild completes, then emits [`accountInitialized`](/docs/webhooks/accountinitialized) again
- `notifyFrom` is reset to the time of the flush unless the request carries its own value, so messages that were already in the mailbox do not produce `messageNew` webhooks by default
- Normal operations resume after the sync completes
:::

## Use Cases

### Email Processing Pipeline

For pipelines that only process incoming emails (AI analysis, vector embeddings, archival):

```bash
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "pipeline-account",
    "imapIndexer": "fast",
    "notifyFrom": "2024-01-01T00:00:00.000Z",
    "imap": {
      "host": "imap.example.com",
      "port": 993,
      "secure": true,
      "auth": { "user": "user@example.com", "pass": "password" }
    }
  }'
```

### CRM Integration

For CRM systems that need complete email sync including deletions:

```bash
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "crm-account",
    "imapIndexer": "full",
    "imap": {
      "host": "imap.example.com",
      "port": 993,
      "secure": true,
      "auth": { "user": "user@example.com", "pass": "password" }
    }
  }'
```

### Processing Existing Emails

To trigger webhooks for existing emails (for example, an initial data import), flush the account with a `notifyFrom` in the past:

```bash
curl -X PUT https://emailengine.example.com/v1/account/user123/flush \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "flush": true,
    "notifyFrom": "1970-01-01T00:00:00.000Z"
  }'
```

This triggers `messageNew` webhooks for every message in the account that is newer than `notifyFrom`, which for this value means all of them.

## Monitoring

Check the current indexer setting for an account:

```bash
curl https://emailengine.example.com/v1/account/user123 \
  -H "Authorization: Bearer YOUR_TOKEN"
```

The response includes the `imapIndexer` field showing the active strategy.

## See Also

- [Managing accounts](/docs/accounts/managing-accounts) - Account creation, updates, and the account lifecycle
- [Continuous Email Processing](/docs/receiving/continuous-processing) - Building email processing pipelines
- [Webhooks](/docs/webhooks/overview) - Webhook event reference
- [Flush API](/docs/api/put-v-1-account-account-flush) - API reference for flush operations
