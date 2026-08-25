---
title: Sending Replies and Forwards
sidebar_position: 3
description: Learn how to properly reply to and forward emails with correct threading headers and IMAP flags
---

<!--
SOURCE ATTRIBUTION:
- Primary: blog/2025-02-07-sending-reply-and-forward-emails.md
-->

# Sending Replies and Forwards

EmailEngine makes it easy to reply to or forward any message in your customer's mailbox with just one API call. This guide covers how to use the reference modes to maintain proper email threading.

## Why It Matters

Writing a raw RFC 822 email that threads correctly is deceptively hard - `In-Reply-To`, `References`, prefixes like **Re:**/**Fwd:**, attachment handling, the IMAP `\Answered` flag... and that's _before_ you juggle every provider's SMTP quirks. EmailEngine eliminates that boilerplate so you can reply or forward with one POST request.

## How It Works

When you include a `reference` object in your submission payload, EmailEngine:

1. Fetches the original message from IMAP
2. Extracts necessary headers and metadata
3. Builds proper threading headers (`In-Reply-To`, `References`)
4. Adds appropriate subject prefix (`Re:` or `Fwd:`)
5. Marks the original message with IMAP flags
6. Sends the message with all correct attributes

## Replying to Emails

### Simple Reply

Reply to the sender of the original message using the [submit API](/docs/api/post-v-1-account-account-submit):

```bash
curl -XPOST "https://emailengine.example.com/v1/account/example/submit" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "reference": {
      "message": "AAAADQAABl0",
      "action": "reply",
      "inline": true
    },
    "html": "<p>Hello from myself!</p>"
  }'
```

**Response:**

```json
{
  "response": "Queued for delivery",
  "messageId": "<reply-id@example.com>",
  "queueId": "24279fb3e0dff64e",
  "sendAt": "2025-05-14T10:02:27.135Z"
}
```

**What EmailEngine does automatically:**

- Sets `from` to your account email
- Sets `to` to the referenced message's `Reply-To` address, or to its sender when it names none, unless you supply `to` yourself
- Adds `Re:` prefix to subject (if not already present)
- Sets `In-Reply-To` header to original Message-ID
- Builds `References` header with the thread history
- Marks original message with `\Answered` flag in IMAP

### Reply All

Reply to all recipients (sender + all CC recipients):

```bash
curl -XPOST "https://emailengine.example.com/v1/account/example/submit" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "reference": {
      "message": "AAAADQAABl0",
      "action": "reply-all",
      "inline": true
    },
    "html": "<p>Reply to everyone</p>"
  }'
```

EmailEngine automatically:

- Puts the referenced message's `To` recipients in `to`, next to the address a plain reply would go to
- Puts its `Cc` recipients in `cc`
- Drops your own address, and any address that would otherwise appear twice
- Leaves its `Bcc` out. Those recipients are invisible to everyone else on the thread, and the header only survived filing. A `bcc` you set yourself is still sent

Recipients you supply are added to the derived ones rather than replacing them.

### Reference Options for Replies

#### inline (boolean)

When `true`, EmailEngine includes the original message content in your reply:

```json
{
  "reference": {
    "message": "AAAADQAABl0",
    "action": "reply",
    "inline": true
  },
  "html": "<p>Your reply</p>"
}
```

The original message appears below your content with a quote header:

```
Your reply content

On May 14, 2025, at 10:30 AM, Original Sender wrote:
> Original message content here
```

When `false`, only your new content is included.

### Complete reference object fields

| Field | Type | Description |
|-------|------|-------------|
| `message` | string | EmailEngine message ID to reply to or forward. Required unless `threadId` is supplied. |
| `action` | string | `reply` (default), `reply-all`, or `forward`. |
| `inline` | boolean | Include the original message as quoted text (default `false`). |
| `forwardAttachments` | boolean | Include original attachments when forwarding (only valid when `action` is `forward`; default `false`). |
| `ignoreMissing` | boolean | Continue sending even if the referenced message cannot be found (default `false`). |
| `messageId` | string | Verify the referenced email's `Message-ID` matches this value before proceeding. |
| `threadId` | string | Gmail thread ID to attach the outgoing message to. Used only by Gmail-API accounts; ignored for IMAP and Microsoft Graph accounts, which thread via the RFC `In-Reply-To`/`References` headers. When both `message` and `threadId` are provided, the supplied `threadId` takes precedence over the value derived from the referenced message. |

At least one of `message` or `threadId` must be present.

### Overriding Auto-Generated Fields

You can override any automatically set fields:

```json
{
  "reference": {
    "message": "AAAADQAABl0",
    "action": "reply"
  },
  "to": [{ "address": "different-recipient@example.com" }],
  "subject": "Custom subject instead of Re:",
  "html": "<p>Reply with overrides</p>"
}
```

Be careful: overriding recipients or subject may break email threading.

## Forwarding Emails

### Basic Forward

Forward a message to new recipients:

```bash
curl -XPOST "https://emailengine.example.com/v1/account/example/submit" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "reference": {
      "message": "AAAADQAABl0",
      "action": "forward",
      "inline": true,
      "forwardAttachments": true
    },
    "to": {
      "name": "Andris Reinman",
      "address": "andris@ethereal.email"
    },
    "html": "<p>FYI  -  see below</p>"
  }'
```

**Important:** Unlike replies, forwards require you to set the `to` field. Omit it and EmailEngine returns **400 Bad Request**.

**What EmailEngine does automatically:**

- Adds `Fwd:` prefix to subject
- Prepends original message with forwarding header
- Optionally copies attachments
- Marks original message with `\Answered` and `$Forwarded` flags

### Forward with Attachments

Control attachment handling with `forwardAttachments`:

```json
{
  "reference": {
    "message": "AAAADQAABl0",
    "action": "forward",
    "forwardAttachments": true
  },
  "to": { "address": "recipient@example.com" },
  "html": "<p>See attached files from original email</p>"
}
```

When `true`: EmailEngine streams all attachments from the original message and includes them in the forward.

When `false`: No attachments are included.

**Warning:** Huge attachments can breach the recipient's size limit. If the total size exceeds the mailbox's send limit, the SMTP server will bounce the message. Consider using `forwardAttachments: false` or filtering specific attachments.

### Forward Inline Content

The `inline` option works the same way as with replies:

```json
{
  "reference": {
    "message": "AAAADQAABl0",
    "action": "forward",
    "inline": true
  },
  "to": { "address": "recipient@example.com" },
  "html": "<p>Check this out:</p>"
}
```

The original message appears below your content with a forwarding header:

```
Check this out:

---------- Forwarded message ----------
From: Original Sender <sender@example.com>
Date: May 14, 2025 at 10:30 AM
Subject: Original Subject
To: original-recipient@example.com

Original message content...
```

### Multiple Recipients

Forward to multiple recipients:

```json
{
  "reference": {
    "message": "AAAADQAABl0",
    "action": "forward",
    "inline": true
  },
  "to": [
    { "name": "Alice", "address": "alice@example.com" },
    { "name": "Bob", "address": "bob@example.com" }
  ],
  "cc": [{ "address": "manager@example.com" }]
}
```

## Advanced Scenarios

### Add Commentary to Forward

Provide context when forwarding:

```json
{
  "reference": {
    "message": "AAAADQAABl0",
    "action": "forward",
    "inline": true
  },
  "to": { "address": "team@example.com" },
  "html": "<p>Team,</p><p>Please review the email below and provide your thoughts.</p><p>Thanks,<br>Manager</p>"
}
```

Your HTML content appears before the original message.

### Reply with Attachments

Add new attachments to a reply:

```json
{
  "reference": {
    "message": "AAAADQAABl0",
    "action": "reply"
  },
  "html": "<p>Here are the requested files</p>",
  "attachments": [
    {
      "filename": "document.pdf",
      "content": "base64-content",
      "contentType": "application/pdf"
    }
  ]
}
```

### Forward Without Original Content

Forward just as a reference, with your own message:

```json
{
  "reference": {
    "message": "AAAADQAABl0",
    "action": "forward",
    "inline": false,
    "forwardAttachments": false
  },
  "to": { "address": "recipient@example.com" },
  "subject": "Follow-up on customer inquiry",
  "html": "<p>Following up on the conversation with the customer...</p>"
}
```

This maintains threading but doesn't include original content.

## Getting Message IDs

You need the EmailEngine message ID (e.g., `AAAADQAABl0`) to reply or forward. Get it from:

### 1. Message List API

Use the [list messages API](/docs/api/get-v-1-account-account-messages):

```bash
curl "https://emailengine.example.com/v1/account/example/messages?path=INBOX" \
  -H "Authorization: Bearer <token>"
```

Response includes message IDs:

```json
{
  "messages": [
    {
      "id": "AAAADQAABl0",
      "uid": 1234,
      "subject": "Original message"
    }
  ]
}
```

### 2. Webhooks

When EmailEngine syncs new mail, it sends `messageNew` webhooks containing the message ID:

```json
{
  "event": "messageNew",
  "data": {
    "id": "AAAADQAABl0",
    "subject": "New message"
  }
}
```

Store these IDs in your application for later use.

### 3. Search API

Search for specific messages using the [search messages API](/docs/api/post-v-1-account-account-search):

```bash
curl -XPOST "https://emailengine.example.com/v1/account/example/search?path=INBOX" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "search": {
      "subject": "customer inquiry"
    }
  }'
```

## Common Pitfalls

### Missing 'to' on Forward

**Problem:** Forgot to set the `to` field when forwarding.

```json
{
  "reference": {
    "message": "AAAADQAABl0",
    "action": "forward"
  },
  "html": "<p>FYI</p>"
}
```

**Error:** `400 Bad Request: Missing recipient`

**Solution:** Always specify `to` for forwards:

```json
{
  "reference": {
    "message": "AAAADQAABl0",
    "action": "forward"
  },
  "to": { "address": "recipient@example.com" },
  "html": "<p>FYI</p>"
}
```

### Huge Attachments

**Problem:** EmailEngine streams attachments from IMAP to SMTP. If the total size breaches the mailbox's send limit, the SMTP server will bounce the message.

**Solution:**

- Use `forwardAttachments: false`
- Download attachments separately and host externally
- Inform users of size limits

### Timeout Issues

**Problem:** Some PaaS providers kill idle sockets during long IMAP/SMTP operations.

**Solutions:**

- Increase `smtpTimeout` configuration
- Scale your dynos/instances
- Move EmailEngine off the constrained host
- Use faster document store retrieval

### Wrong Message ID Format

**Problem:** Using wrong ID format (IMAP UID instead of EmailEngine ID).

**Incorrect:**

```json
{
  "reference": {
    "message": "1234"
  }
}
```

**Correct:**

```json
{
  "reference": {
    "message": "AAAADQAABl0"
  }
}
```

Use the base64-encoded ID from EmailEngine, not the numeric IMAP UID.

### Breaking Threads

**Problem:** Overriding subject or recipients breaks threading.

**Avoid:**

```json
{
  "reference": {
    "message": "AAAADQAABl0",
    "action": "reply"
  },
  "subject": "Completely different subject"
}
```

**Better:** Let EmailEngine handle the subject automatically, or only make minor changes.

## Webhook Notifications

Replies and forwards trigger the same webhook events as regular sending:

### messageSent

```json
{
  "event": "messageSent",
  "data": {
    "messageId": "<reply-id@example.com>",
    "response": "250 2.0.0 Ok: queued",
    "reference": {
      "message": "AAAADQAABl0",
      "action": "reply"
    }
  }
}
```

### messageDeliveryError

```json
{
  "event": "messageDeliveryError",
  "data": {
    "queueId": "abc123",
    "error": "Connection timeout",
    "job": {
      "attemptsMade": 1,
      "attempts": 10,
      "nextAttempt": "2025-05-14T15:07:45.465Z"
    }
  }
}
```

### messageFailed

```json
{
  "event": "messageFailed",
  "data": {
    "messageId": "<reply-id@example.com>",
    "queueId": "abc123",
    "error": "Max retries exceeded"
  }
}
```

## Testing Replies and Forwards

### Test with Ethereal Email

1. Create an Ethereal test account at [ethereal.email](https://ethereal.email/)
2. Send a test email to your EmailEngine account
3. Get the message ID from the API or webhooks
4. Send a reply or forward
5. Check the Ethereal inbox for the result

### Verify IMAP Flags

Check that the original message was flagged:

```bash
curl "https://emailengine.example.com/v1/account/example/message/AAAADQAABl0" \
  -H "Authorization: Bearer <token>"
```

Look for `\Answered` in the `flags` array:

```json
{
  "id": "AAAADQAABl0",
  "flags": ["\\Seen", "\\Answered"]
}
```

## Performance Considerations

### Limit Forwarded Attachments

For large forwards, consider:

- Setting `forwardAttachments: false`
- Downloading and re-uploading only specific attachments
- Using external file storage with links

### Cache Original Messages

If you need to reply multiple times to the same message:

- Fetch the original message once via API
- Cache the necessary fields in your application
- Build the reply headers yourself
