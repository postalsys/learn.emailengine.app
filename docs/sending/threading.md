---
title: Email Threading
sidebar_position: 5
description: Understanding and maintaining email conversation threads across multiple messages
---

# Email Threading

Email threading allows related messages to be grouped together as conversations. This is essential for creating email experiences that feel natural to users and ensuring your messages appear together in email clients.

## Quick Start

### For Simple Replies

Use EmailEngine's reference API for automatic threading:

```bash
curl -XPOST "https://emailengine.example.com/v1/account/example/submit" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "reference": {
      "message": "AAAADQAABl0",
      "action": "reply"
    },
    "html": "<p>Your reply</p>"
  }'
```

See [Replies & Forwards](./replies-forwards.md) for details.

### For Custom Thread Sequences

Control Message-ID, In-Reply-To, and References headers manually:

```bash
curl -XPOST "https://emailengine.example.com/v1/account/example/submit" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "from": { "address": "sender@example.com" },
    "to": { "address": "recipient@example.com" },
    "subject": "Your inquiry",
    "html": "<p>Follow-up message</p>",
    "messageId": "<new-uuid@example.com>",
    "headers": {
      "in-reply-to": "<original-uuid@example.com>",
      "references": "<original-uuid@example.com>"
    }
  }'
```

See [Sending Threaded Messages](./threading/sending-threaded) for step-by-step instructions.

## Threading Documentation

This guide is organized into focused topics:

### 1. [Threading Overview](./threading/overview)

Learn the fundamentals of email threading:

- How threading works (Message-ID, In-Reply-To, References)
- Why subject lines matter
- Thread ID formats
- RFC standards (5256, 8474)
- EmailEngine's threading approach

**Start here if you're new to email threading.**

### 2. [Provider-Specific Threading](./threading/provider-support)

Understand how different email providers handle threading:

- Gmail/Google Workspace (native threading, `\All` folder support)
- Microsoft Graph API (native threading, `\All` folder support)
- Outlook/Microsoft 365 IMAP (no native threading, manual only)
- Yahoo/AOL/Verizon (OBJECTID support, no `\All` folder)
- Generic IMAP providers (no native threading, manual only)

**Key information:**

- Microsoft Graph API backend has native threading; IMAP backend doesn't
- Gmail and Graph API support `\All` folder for cross-folder search
- Yahoo/AOL require folder-by-folder search despite native threading
- Generic IMAP providers require manual thread building from Message-ID headers

### 3. [Searching Thread Messages](./threading/searching-threads)

Find all messages in a conversation thread:

- Using the `\All` folder (Gmail, Graph API)
- Folder-by-folder search (Yahoo, generic IMAP)
- Building threads manually from headers
- Provider-specific search strategies

**Best for:** Retrieving complete conversation history, building thread views in your UI.

### 4. [Sending Threaded Messages](./threading/sending-threaded)

Send multiple emails that appear as a single conversation:

- Step-by-step guide for threaded sending
- Managing Message-IDs and References headers
- Best practices (angle brackets, Gmail 20-message limit)
- Persisting thread data in your database

**Best for:** Drip campaigns, follow-up sequences, multi-step workflows.

## Common Use Cases

### Building an Email Client

If you're building an email application (webmail, mobile app, desktop client):

1. Read [Threading Overview](./threading/overview) for fundamentals
2. Check [Provider Support](./threading/provider-support) for accounts you'll support
3. Implement search using [Searching Thread Messages](./threading/searching-threads)
4. Handle replies with [Replies & Forwards](./replies-forwards.md)

### Email Marketing / Drip Campaigns

For automated email sequences that appear as conversations:

1. Understand basics in [Threading Overview](./threading/overview)
2. Follow [Sending Threaded Messages](./threading/sending-threaded) guide
3. Store Message-IDs in your database
4. Build References headers for each follow-up

### CRM / Support System Integration

For managing customer email conversations:

1. Learn [Provider Support](./threading/provider-support) for customer email providers
2. Use [Searching Thread Messages](./threading/searching-threads) to retrieve history
3. Send replies using [Replies & Forwards](./replies-forwards.md)
4. Track threads in your CRM database

## Key Concepts

### Thread IDs

EmailEngine provides a `threadId` property that identifies related messages. The format varies by provider:

| Provider               | Thread ID Format  | Example                 |
| ---------------------- | ----------------- | ----------------------- |
| Gmail/Google Workspace | Long numeric      | `"1759349012996310407"` |
| Microsoft Graph API    | Conversation ID   | `"AAQkAGI2TH..."`       |
| Yahoo/AOL/Verizon      | Short numeric     | `"501"`                 |
| Generic IMAP           | N/A (manual only) | N/A                     |

### The `\All` Folder

Gmail and Microsoft Graph API support a special `\All` folder that searches across all mailboxes:

```bash
curl -XPOST "https://emailengine.example.com/v1/account/gmail/search?path=%5CAll" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "search": { "threadId": "1759349012996310407" }
  }'
```

This returns all messages in the thread from Inbox, Sent, and any other folder in a single response.

**Supported:**

- Gmail (IMAP + OAuth2)
- Gmail (Gmail API)
- Microsoft Graph API

**Not supported:**

- Outlook/Microsoft 365 (IMAP + OAuth2)
- Yahoo/AOL/Verizon
- Generic IMAP

### Generic IMAP Threading

For generic IMAP providers without native threading, threads must be built manually:

**Without native threading:**

- No `threadId` in responses
- Cannot search by thread ID
- Must build threads manually from Message-ID headers
- Requires analyzing In-Reply-To and References headers

See [Provider Support](./threading/provider-support) and [Searching Threads](./threading/searching-threads.md#building-threads-manually) for implementation details.

## Threading Headers Explained

### Message-ID

Every email must have a unique identifier:

```
Message-ID: <56b3c6d2-f7c0-4272-8beb-e25fdb7c19f1@example.com>
```

**Best practices:**

- Always wrapped in angle brackets `< >`
- Use UUID for uniqueness
- Use the sender's domain from the `From` address (preferred), or your service domain as fallback
- Store these IDs for building References later

### In-Reply-To

References the message being replied to:

```
In-Reply-To: <56b3c6d2-f7c0-4272-8beb-e25fdb7c19f1@example.com>
```

Set automatically by EmailEngine when using the reference API.

### References

The complete chain of Message-IDs in the conversation:

```
References: <original@example.com> <reply1@example.com> <reply2@example.com>
```

**Important:**

- Space-separated list
- Each ID in angle brackets
- Oldest first, newest last
- Gmail only reads last 20

## Provider Comparison

| Feature                   | Gmail           | Graph API       | Outlook IMAP   | Yahoo/AOL      | Generic IMAP    |
| ------------------------- | --------------- | --------------- | -------------- | -------------- | --------------- |
| Native Threading          | Yes             | Yes             | No             | Yes            | No              |
| `\All` Folder             | Yes             | Yes             | No             | No             | No              |
| Manual Threading Required | No              | No              | Yes            | No             | Yes             |
| Cross-Folder Search       | Single API call | Single API call | Multiple calls | Multiple calls | Manual building |

## See Also

- [Replies and forwards](/docs/sending/replies-forwards) - Letting EmailEngine build the threading headers for you
- [Searching messages](/docs/receiving/searching) - The search terms the thread queries here are built from
- [Message IDs](/docs/advanced/ids-explained) - What a `threadId` is, and why it is not portable between providers
- [Messages API](/docs/api-reference/messages-api) - Where `threadId` appears in a message payload
