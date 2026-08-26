---
title: Threading Overview
sidebar_position: 1
description: How email threading works, which headers drive it, and which backends give EmailEngine a native thread ID
---

# Email Threading Overview

Email threading groups related messages into a conversation. This page covers the headers that mail clients thread on, the thread identifiers some servers assign, and where EmailEngine exposes them.

## How Email Threading Works

Threading is done by the client, from four pieces of each message:

1. **Message-ID**: the unique identifier of the message
2. **In-Reply-To**: the `Message-ID` of the message being replied to
3. **References**: the chain of `Message-ID`s in the thread so far
4. **Subject**: expected to stay the same apart from `Re:` and `Fwd:` prefixes

### RFC 5256

RFC 5256 defined server-side threading over a single folder. That works for mailing-list traffic, where the whole thread sits in one folder, and poorly for one-to-one conversations, where half the messages are in the Inbox and the other half in Sent Mail.

### RFC 8474

RFC 8474 (the `OBJECTID` extension) gives each message a server-assigned thread ID that holds across folders. Support depends on the server, so client-side threading from headers remains the fallback for most accounts.

## Threading Headers Explained

### Message-ID

Every message has a unique `Message-ID`:

```
Message-ID: <56b3c6d2-f7c0-4272-8beb-e25fdb7c19f1@example.com>
```

**Format**: `<unique-id@domain>`

When you set it yourself:

- Wrap it in angle brackets `< >`
- Use a UUID or a comparably unique value
- Use the domain of the `From` address, or your own service domain
- Store it, because every follow-up needs it for `References`

If you do not set `messageId` on submit, EmailEngine generates one and returns it in the submit response.

### In-Reply-To

Names the message being replied to:

```
In-Reply-To: <56b3c6d2-f7c0-4272-8beb-e25fdb7c19f1@example.com>
```

This is the parent-child link between two messages. EmailEngine sets it for `reply` and `reply-all` submissions that use `reference`.

### References

The full chain of `Message-ID`s in the conversation:

```
References: <original@example.com> <reply1@example.com> <reply2@example.com>
```

- Space-separated
- Each ID in angle brackets
- Oldest first, newest last

When you use `reference`, EmailEngine builds this header from the referenced message's `Message-ID`, `In-Reply-To`, and `References`, adding missing angle brackets and removing duplicates. It sets no upper bound on the chain, and it does not touch a `references` header you set yourself.

## Threading Challenges

### Split Across Folders

A conversation is rarely in one folder:

- **Inbox**: received messages
- **Sent**: your side of the conversation
- **Other folders**: messages moved by filters or by hand

Retrieving a whole thread therefore needs either a cross-folder search or one search per folder. See [Searching threads](/docs/sending/threading/searching-threads).

### Provider Differences

- **Gmail**: a thread ID on every message (`X-GM-THRID` over IMAP, `threadId` in the Gmail API)
- **Microsoft 365**: a conversation ID over the Graph API, nothing over IMAP
- **Yahoo and AOL**: a thread ID through the `OBJECTID` extension
- **Other IMAP servers**: no thread ID unless the server implements `OBJECTID`

See [Provider support](/docs/sending/threading/provider-support) for the details.

### Subject Line Sensitivity

Some clients include the subject in their threading decision. Changing the subject text between messages, beyond adding `Re:` or `Fwd:`, can split a thread even when the headers are correct.

## EmailEngine's Threading Support

EmailEngine passes on the thread identifier whenever the backend provides one:

1. **Gmail API**: the Gmail API's `threadId`
2. **Microsoft Graph API**: the message's `conversationId`
3. **IMAP**: the value the server returns when it supports `OBJECTID` (RFC 8474) or Gmail's `X-GM-EXT-1` extension, which covers Gmail over IMAP and OBJECTID servers such as Yahoo and AOL

IMAP servers without either extension give EmailEngine nothing to pass on. For those accounts a thread has to be reconstructed from `Message-ID`, `In-Reply-To`, and `References`.

## Thread ID Format

The `threadId` property is a string whose format depends on the provider:

| Provider                        | Format                                            | Example                 |
| ------------------------------- | ------------------------------------------------- | ----------------------- |
| Gmail API                       | Long numeric string                               | `"1759349012996310407"` |
| Gmail over IMAP (X-GM-EXT-1)    | Long numeric string                               | `"1759349012996310407"` |
| Microsoft Graph API             | Graph `conversationId`, a long base64 string       | `"AAQkAGI2THY2..."`     |
| IMAP with OBJECTID (Yahoo, AOL) | Short numeric string                              | `"501"`                 |

Treat every one of them as an opaque string. [Provider support](/docs/sending/threading/provider-support) has a full Graph conversation ID in context.

A `threadId` is only meaningful within the account it came from. See [Message IDs](/docs/advanced/ids-explained) for how it relates to the other identifiers.

## Where Thread IDs Appear

`threadId` is present, when the backend provides one, in:

- Message listing responses
- Message detail responses
- Message search responses
- Webhook payloads that carry a message, such as `messageNew`

### Message Lists Are Not Thread Lists

EmailEngine does not group a listing by thread. Messages in the same thread are listed as separate entries, whatever the backend.

To get the messages of one thread, search with `threadId` in the `search` object. The `path` query parameter is required:

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

See [Searching threads](/docs/sending/threading/searching-threads) for which folder to search per provider.

**Example webhook** (abridged; a real `messageNew` payload carries the full message entry):

```json
{
  "account": "example",
  "date": "2025-10-10T14:30:00.000Z",
  "path": "INBOX",
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
