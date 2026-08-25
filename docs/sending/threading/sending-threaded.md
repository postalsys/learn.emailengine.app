---
title: Sending Threaded Messages
sidebar_position: 4
description: How to send multiple emails in the same conversation thread
---

# Sending Threaded Messages

To maintain a conversation thread when sending follow-up emails, you need to control the Message-ID and References headers. This guide shows you how to send multiple related emails that appear as a single conversation.

## Why Manual Threading is Needed

Email clients rely on RFC 5322 `Message-ID` and `References` headers to decide which messages belong together. If you let EmailEngine autogenerate those values, your perfectly timed sequence may scatter across the inbox. By controlling them yourself, every follow-up lands exactly where the user expects.

## Step-by-Step: Sending a Thread

### Step 1: Send the Initial Message

Send the first message with a custom Message-ID using the [Submit Email API endpoint](/docs/api/post-v-1-account-account-submit):

```bash
curl -XPOST "http://127.0.0.1:3000/v1/account/demo/submit" \
  -H "Authorization: Bearer $EE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from": { "address": "sender@example.com" },
    "to": { "address": "recipient@example.com" },
    "subject": "Product inquiry",
    "html": "<p>First message in thread!</p>",
    "messageId": "<56b3c6d2-f7c0-4272-8beb-e25fdb7c19f1@example.com>"
  }'
```

**Important:** Save the `messageId` value - you'll need it for every follow-up.

**Response:**

```json
{
  "response": "Queued for delivery",
  "messageId": "<56b3c6d2-f7c0-4272-8beb-e25fdb7c19f1@example.com>",
  "queueId": "abc123"
}
```

### Step 2: Send the First Follow-Up

Send a follow-up with the original Message-ID in the References header:

```bash
curl -XPOST "http://127.0.0.1:3000/v1/account/demo/submit" \
  -H "Authorization: Bearer $EE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from": { "address": "sender@example.com" },
    "to": { "address": "recipient@example.com" },
    "subject": "Product inquiry",
    "html": "<p>Second message in thread!</p>",
    "messageId": "<77a7c383-cc1a-44c6-9866-96b2873e3322@example.com>",
    "headers": {
      "references": "<56b3c6d2-f7c0-4272-8beb-e25fdb7c19f1@example.com>"
    }
  }'
```

**Key points:**

- New unique `messageId` for this message
- `references` header contains the first message's ID
- Subject stays the same (important!)

### Step 3: Keep Extending References

Each subsequent message appends all previous Message-IDs to the References header:

```bash
curl -XPOST "http://127.0.0.1:3000/v1/account/demo/submit" \
  -H "Authorization: Bearer $EE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from": { "address": "sender@example.com" },
    "to": { "address": "recipient@example.com" },
    "subject": "Product inquiry",
    "html": "<p>Third message in thread!</p>",
    "messageId": "<8c9d1234-e5f6-7890-abcd-ef1234567890@example.com>",
    "headers": {
      "references": "<56b3c6d2-f7c0-4272-8beb-e25fdb7c19f1@example.com> <77a7c383-cc1a-44c6-9866-96b2873e3322@example.com>"
    }
  }'
```

**References header format:**

- Space-separated list of Message-IDs
- Each ID wrapped in angle brackets `< >`
- Oldest message first, newest last
- Build up the chain with each new message

## Using the Reference API

For replies and forwards, EmailEngine can handle threading automatically using the `reference` parameter:

```bash
curl -XPOST "http://127.0.0.1:3000/v1/account/example/submit" \
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

EmailEngine automatically:

- Extracts the original Message-ID
- Builds the References header
- Sets In-Reply-To header
- Adds Re: prefix to subject
- Maintains the thread

**When to use Reference API:**

- Replying to received emails
- Forwarding existing messages
- You have the internal message ID from EmailEngine (the `id` field, e.g., `"AAAADQAABl0"`, not the Message-ID header)

**When to use manual threading:**

- Sending multiple follow-ups from scratch
- Building drip campaigns
- Scheduled message sequences
- You don't have an original message to reference

See [Replies & Forwards](/docs/sending/replies-forwards) for details on the Reference API.

## Common Pitfalls

### Missing Angle Brackets

**Problem:** Message IDs in References header without `< >`.

**Result:** Threading breaks in many email clients.

**Solution:** Always wrap IDs in angle brackets, both in storage and when building References.

### Subject Drift

**Problem:** Changing the actual subject text between messages in a thread.

**Result:** Thread breaks despite correct Message-ID headers.

**Solution:** Keep the subject text consistent. Adding Re: or Fwd: prefixes is fine and recommended (EmailEngine does this automatically when using the Reference API), but don't change the actual subject content (e.g., "Meeting on Monday" → "Meeting on Tuesday").

### Not Persisting Message IDs

**Problem:** Not storing Message-IDs after generation.

**Result:** Cannot build References header for follow-up messages.

**Solution:** Store all Message-IDs in your database immediately after generation, indexed by thread.

### Message-ID Rewriting by Mail Servers

**Problem:** Some mail servers override the Message-ID header you set, causing your locally stored Message-ID to become invalid because the recipient receives a different Message-ID.

**Detection:** EmailEngine can detect Message-ID changes in most cases and notifies you via the `messageSent` webhook event:

```json
{
  "account": "user@example.com",
  "date": "2025-01-15T10:30:00.000Z",
  "event": "messageSent",
  "data": {
    "messageId": "<rewritten-id@mailserver.com>",
    "originalMessageId": "<56b3c6d2-f7c0-4272-8beb-e25fdb7c19f1@example.com>",
    "response": "250 2.0.0 OK",
    "envelope": {
      "from": "sender@example.com",
      "to": ["recipient@example.com"]
    }
  }
}
```

**Solution:** Listen for `messageSent` webhooks and update your stored Message-ID with the `messageId` value (the rewritten ID) if `originalMessageId` is present. Use the rewritten ID in future References headers to maintain threading.

**Important limitations** - Message-ID rewriting cannot be detected in these cases:

1. **AWS WorkMail**: The Message-ID is rewritten during delivery after the message has been passed from EmailEngine to the mail server, so EmailEngine never sees the final Message-ID.

2. **Gmail API with send-only scope** (`gmail.send`): Gmail API modifies the Message-ID, but EmailEngine cannot resolve the updated Message-ID because it lacks read access to the mail storage. You need both send and read OAuth2 scopes for Message-ID detection to work.

**Recommendation:**

- Always implement webhook handlers to track Message-ID changes
- Use full OAuth2 scopes for Gmail (not just `gmail.send`)
- Be aware that AWS WorkMail requires alternative threading strategies
- Test your threading implementation with your specific mail server configuration
