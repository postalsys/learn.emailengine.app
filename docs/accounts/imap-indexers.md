---
title: IMAP Indexers
sidebar_position: 11
description: Understanding full and fast indexing strategies for IMAP accounts
---

# IMAP Indexers

EmailEngine uses an indexing strategy to track email changes in IMAP accounts. The indexer determines how EmailEngine detects new, updated, and deleted messages, affecting both webhook notifications and Redis storage usage.

:::note IMAP Only
Indexers only apply to IMAP accounts. Gmail API and MS Graph accounts use their own change detection mechanisms provided by the respective APIs.
:::

## Indexing Strategies

### Full Indexer (Default)

The full indexer maintains a complete reference of all emails in each mailbox, stored in Redis.

**How it works:**
- Stores a reference (UID and flags) for every message in each folder
- Compares current mailbox state against stored references
- Detects new messages, deleted messages, and flag changes

**Capabilities:**

| Event | Detected |
|-------|----------|
| New messages | Yes |
| Deleted messages | Yes |
| Flag changes (read/unread, starred, etc.) | Yes |
| Message moves between folders | Yes (as delete + new) |

**Trade-offs:**
- Higher Redis memory usage (stores reference for every message)
- Slower initial sync (must index all messages)
- Complete change detection

**Best for:**
- Applications that need to track all email changes
- CRM integrations requiring deletion and flag sync
- Full mailbox synchronization use cases

### Fast Indexer

The fast indexer only tracks the highest UID (unique identifier) seen in each mailbox.

**How it works:**
- Stores only the last known UID for each folder
- Detects new messages by checking for UIDs higher than stored value
- Does not track individual message states

**Capabilities:**

| Event | Detected |
|-------|----------|
| New messages | Yes |
| Deleted messages | No |
| Flag changes (read/unread, starred, etc.) | No |
| Message moves between folders | Partial (new in destination only) |

**Trade-offs:**
- Minimal Redis memory usage
- Faster initial sync
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
| Redis storage | High (all messages) | Low (one UID per folder) |
| Initial sync speed | Slower | Faster |
| Ongoing sync speed | Similar | Similar |

## Configuration

### Global Default

Set the default indexer for all new IMAP accounts via API:

```bash
curl -X POST https://emailengine.example.com/v1/settings \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "imapIndexer": "full"
  }'
```

Or via the web interface: **Configuration** > **Email Processing** > **IMAP Processing** > **Indexing Method**

:::info New Accounts Only
Changing the global default only affects newly created accounts. Existing accounts continue using their current indexer setting. To migrate existing accounts to a different indexer, you must update each account individually using the [flush API](/docs/api/put-v-1-account-account-flush).
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

This resets the account index and rebuilds it using the new indexer strategy.

:::warning Index Reset
Changing the indexer flushes the existing index. During re-indexing:
- The account temporarily shows as syncing
- `messageNew` webhooks may be sent for existing messages (depending on `notifyFrom`)
- Normal operations resume after sync completes
:::

## Use Cases

### Email Processing Pipeline

For pipelines that only process incoming emails (AI analysis, vector embeddings, archival):

```bash
# Use fast indexer - only need new messages
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
# Use full indexer - need all changes
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

To trigger webhooks for existing emails (e.g., initial data import):

```bash
# Flush with notifyFrom in the past
curl -X PUT https://emailengine.example.com/v1/account/user123/flush \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "flush": true,
    "notifyFrom": "1970-01-01T00:00:00.000Z"
  }'
```

This triggers `messageNew` webhooks for all existing emails in the account.

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
