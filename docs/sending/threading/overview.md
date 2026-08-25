---
title: Threading Overview
sidebar_position: 1
description: Understanding how email threading works and why it matters
---

# Email Threading Overview

Email threading allows related messages to be grouped together as conversations. Understanding how threading works is essential for building email applications that feel natural to users.

## How Email Threading Works

Email threading is typically managed on the client side as virtual entities. Emails are threaded based on:

1. **Message-ID**: Unique identifier for each message
2. **In-Reply-To**: References the Message-ID being replied to
3. **References**: Chain of all Message-IDs in the thread
4. **Subject**: Must remain consistent (with Re:/Fwd: prefixes)

### RFC 5256 Limitations

Previous attempts to define server-side threading (RFC5256) were mainly useful for mailing-list type threads, assuming that all related emails were located in the same folder. This approach proved ineffective for one-to-one threads, where half of the emails are in the Inbox and the other half in the Sent Mail folder.

### RFC 8474 Modern Approach

RFC8474 introduced a new method where each email is assigned a thread ID, making it possible to identify emails belonging to the same thread across different folders. However, this solution is not yet widely supported by email servers, meaning that, for the time being, email threading must still be managed on the client side for most users.

## Threading Headers Explained

### Message-ID

Every email must have a unique Message-ID:

```
Message-ID: <56b3c6d2-f7c0-4272-8beb-e25fdb7c19f1@example.com>
```

**Format**: `<unique-id@domain>`

**Best practices**:

- Always wrap in angle brackets `< >`
- Use UUID or similar unique identifier
- Use the sender's domain from the `From` address (e.g., if `From: user@company.com`, use `@company.com`)
- Alternatively, use your service domain as fallback
- Store these IDs for building References later

### In-Reply-To

When replying, reference the original message:

```
In-Reply-To: <56b3c6d2-f7c0-4272-8beb-e25fdb7c19f1@example.com>
```

This creates a parent-child relationship between messages.

### References

The full chain of Message-IDs in the conversation:

```
References: <original@example.com> <reply1@example.com> <reply2@example.com>
```

**Important**:

- Space-separated list of Message-IDs
- Each ID wrapped in angle brackets
- Oldest message first, newest last
- Gmail only reads last 20 entries

## Threading Challenges

### Split Across Folders

One major challenge with email threading is that conversation messages are often split across folders:

- **Inbox**: Contains received messages
- **Sent**: Contains sent messages
- **Other folders**: Messages may be moved by filters

This makes it difficult to retrieve all messages in a thread with a single query.

### Provider Differences

Different email providers handle threading differently:

- **Gmail**: Native thread support (X-GM-THRID)
- **Outlook**: Depends on backend (Graph API has native support, IMAP doesn't)
- **Yahoo/AOL**: THREADID support via OBJECTID extension
- **Generic IMAP**: No native threading, must build manually

See [Provider Support](./provider-support) for detailed information.

### Subject Line Sensitivity

Email clients use subject matching as part of threading. Changing the subject (beyond adding Re:/Fwd:) can break threads in some clients, even with perfect headers.

## EmailEngine's Threading Solution

EmailEngine exposes native thread IDs whenever the underlying mail server provides them:

1. **Gmail (API)**: Uses Gmail's native thread IDs from the Gmail API
2. **Microsoft Graph API**: Uses Outlook's conversation IDs
3. **IMAP with extensions**: The IMAP backend returns `threadId` whenever the server supports the OBJECTID extension (RFC 8474) or Gmail's X-GM-EXT-1 extension. This covers Gmail over IMAP and OBJECTID-capable servers such as Yahoo and AOL.

Only IMAP servers without these extensions lack native thread IDs. For those accounts, threads must be built manually by analyzing Message-ID, In-Reply-To, and References headers from email messages.

## Thread ID Format

EmailEngine provides a `threadId` property in message data when the server supports threading. The format depends on the provider:

| Provider                           | Format                | Example                 |
| ---------------------------------- | --------------------- | ----------------------- |
| Gmail/Google Workspace (API)       | Long numeric string   | `"1759349012996310407"` |
| Gmail over IMAP (X-GM-EXT-1)       | Long numeric string   | `"1759349012996310407"` |
| Microsoft Graph API                | Graph conversation ID | `"AAQkAGI2TH..."`       |
| IMAP with OBJECTID (Yahoo, AOL)    | Server-assigned ID    | `"M6d99fb603329c098"`   |

IMAP servers without the OBJECTID or X-GM-EXT-1 extensions do not provide native thread IDs.

## Where Thread IDs Appear

The `threadId` property is available in:

- Message listing responses
- Message detail responses
- Message search responses
- Webhook payloads (e.g., `messageNew`)

### Important: Message Lists vs Thread Lists

EmailEngine does not support thread grouping in message lists. When you list messages and there are multiple messages in the same thread, EmailEngine lists them as separate individual messages, not grouped by thread - even if the mail server supports threading.

**To list messages in a thread:**
Use the search API with a `threadId` field in the `search` body object (if the mail server supports it). The `path` query parameter is required:

```bash
curl -XPOST "https://emailengine.example.com/v1/account/example/search?path=INBOX" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "search": {
      "threadId": "1759349012996310407"
    }
  }'
```

See [Searching Threads](./searching-threads) for complete details.

**Example webhook**:

```json
{
  "event": "messageNew",
  "data": {
    "id": "AAABkPHBeR0",
    "threadId": "1759349012996310407",
    "subject": "Project discussion",
    "from": {
      "address": "colleague@example.com"
    }
  }
}
```

## See Also

- [Provider support](/docs/sending/threading/provider-support) - Which backends assign a thread ID at all
- [Searching threads](/docs/sending/threading/searching-threads) - Retrieving every message in a conversation
- [Sending threaded messages](/docs/sending/threading/sending-threaded) - Keeping a sequence you send in one thread
- [Replies and forwards](/docs/sending/replies-forwards) - Letting EmailEngine build the headers
