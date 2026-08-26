---
title: Email Threading
sidebar_position: 1
description: How EmailEngine groups related messages into conversations, and where to read about thread IDs, provider support, thread search, and threaded sending
---

# Email Threading

Email threading groups related messages into a conversation. Mail clients decide what belongs together from the `Message-ID`, `In-Reply-To`, and `References` headers, and some mail servers additionally assign a thread identifier that EmailEngine exposes as `threadId`. This page is the entry point; the four pages under it each own one part of the subject.

## Quick Start

Use the `reference` field of the submit API and EmailEngine sets `In-Reply-To` and `References` from the referenced message:

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

That covers replies and forwards to a message EmailEngine can already see. A sequence that starts with no stored message to reference has to carry its own `messageId` and threading headers; [Sending threaded messages](/docs/sending/threading/sending-threaded) walks through one.

## Threading Documentation

### 1. [Threading Overview](/docs/sending/threading/overview)

The fundamentals:

- What `Message-ID`, `In-Reply-To`, and `References` do
- Why the subject line matters
- Which backends assign a `threadId`, and what one looks like per provider
- Where `threadId` appears in API responses and webhook payloads

### 2. [Provider-Specific Threading](/docs/sending/threading/provider-support)

How each backend behaves:

- Gmail over IMAP and the Gmail API: thread IDs and the `\All` folder
- Microsoft 365 over the Graph API: conversation IDs and the `\All` folder
- Microsoft 365 over IMAP: no thread IDs
- Yahoo, AOL, and other OBJECTID servers: thread IDs, but no `\All` folder
- Other IMAP servers: no thread IDs

### 3. [Searching Thread Messages](/docs/sending/threading/searching-threads)

Retrieving every message in a conversation:

- One search against `\All` where the backend has it
- One search per folder where it does not
- Building a thread from headers when the server assigns no `threadId`

### 4. [Sending Threaded Messages](/docs/sending/threading/sending-threaded)

Keeping a sequence you send in one conversation:

- Setting `messageId` and extending `References` with each message
- When to use `reference` instead
- Detecting a `Message-ID` the receiving server rewrote

## See Also

- [Replies and forwards](/docs/sending/replies-forwards) - Letting EmailEngine build the threading headers for you
- [Searching messages](/docs/receiving/searching) - The search terms the thread queries are built from
- [Message IDs](/docs/advanced/ids-explained) - What a `threadId` is, and why it is not portable between providers
- [Messages API](/docs/api-reference/messages-api) - Where `threadId` appears in a message payload
