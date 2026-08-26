---
title: Sending Replies and Forwards
sidebar_position: 3
description: Learn how to properly reply to and forward emails with correct threading headers and IMAP flags
---

# Sending Replies and Forwards

A `reference` object in a submit call turns the new message into a reply to, or a forward of, a message already in the mailbox. EmailEngine reads the referenced message and fills in what a mail client would: the recipients, the subject prefix, the `In-Reply-To` and `References` headers, optionally the quoted original and its attachments, and the flags on the original once the new message is sent. This guide covers each reference mode and what it derives.

## Why It Matters

A reply that threads correctly needs `In-Reply-To` and `References` built from the original's headers, a `Re:` or `Fwd:` prefix that is not doubled, the quoted text in the form clients expect, and the `\Answered` flag set on the original afterwards. The `reference` object does all of that from one message ID, so the caller supplies only the new content.

## How It Works

When you include a `reference` object in your submission payload, EmailEngine:

1. Fetches the referenced message from the mailbox (over IMAP, or the Gmail or MS Graph API)
2. Extracts the headers and metadata it needs
3. Builds the threading headers (`In-Reply-To`, `References`)
4. Adds the subject prefix (`Re:` or `Fwd:`) unless the subject already carries it
5. Queues the message
6. Flags the original message once the new one has been sent

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
  "sendAt": "2025-05-14T10:02:27.135Z",
  "reference": {
    "message": "AAAADQAABl0",
    "success": true
  }
}
```

The `reference` object in the response reports whether the referenced message was found. With `ignoreMissing` set and the message gone, `success` is `false` and `error` says why, and the message is sent without the derived fields.

**What EmailEngine does automatically:**

- Sets `from` to your account email
- Sets `to` to the referenced message's `Reply-To` address, or to its sender when it names none, unless you supply `to` yourself
- Adds `Re:` prefix to subject (if not already present)
- Sets `In-Reply-To` header to original Message-ID
- Builds `References` header with the thread history
- Marks original message with the `\Answered` flag once the reply is sent

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

The original message appears below your content with a quote header. The HTML part carries the header and the original HTML in a `blockquote`; when the reply also has a `text` part, the original is quoted there like this:

```text
Your reply content

On Wed, May 14, 2025 10:30 AM, Original Sender <sender@example.com> wrote:
>
> Original message content here
```

The date is formatted for the `locale` and `tz` of the submission, falling back to the account's and then the instance's settings. Images the original embeds by Content-ID are carried over so the quote renders. When `inline` is `false`, only your new content is included.

### Complete reference object fields

| Field | Type | Description |
|-------|------|-------------|
| `message` | string | EmailEngine message ID to reply to or forward. Required unless `threadId` is supplied. |
| `action` | string | `reply` (default), `reply-all`, or `forward`. |
| `inline` | boolean | Include the original message as quoted text (default `false`). |
| `forwardAttachments` | boolean | Include original attachments when forwarding (only valid when `action` is `forward`; default `false`). |
| `ignoreMissing` | boolean | Continue sending even if the referenced message cannot be found (default `false`). |
| `messageId` | string | Verify the referenced email's `Message-ID` matches this value before proceeding. |
| `threadId` | string | Gmail thread ID to attach the outgoing message to. Used only by Gmail API accounts; ignored for IMAP and Microsoft Graph accounts, which thread via the RFC `In-Reply-To`/`References` headers. |

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
    "html": "<p>FYI, see below</p>"
  }'
```

**Important:** Unlike a reply, a forward has no implied recipient, so set `to` yourself. A forward submitted without one is accepted into the queue and then fails at the first delivery attempt with `EENVELOPE` (`No recipients defined`), which is a permanent error, so it is reported through `messageFailed` rather than retried.

**What EmailEngine does automatically:**

- Adds `Fwd:` prefix to subject
- Prepends original message with forwarding header (with `inline`)
- Optionally copies attachments (with `forwardAttachments`)
- Marks original message with the `\Answered` and `$Forwarded` flags once the forward is sent

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

When `true`: EmailEngine downloads every attachment of the original message and includes it in the forward.

When `false`: No attachments are included, apart from images the quoted HTML embeds by Content-ID when `inline` is also set.

To forward some attachments but not others, leave `forwardAttachments` off and list the ones you want in `attachments` by their `reference` ID instead (see [attachments](./basic-sending.md#attachments)).

**Warning:** Large attachments can exceed the size the account's SMTP server accepts, in which case the delivery is rejected and reported as a delivery error. Consider `forwardAttachments: false` or referencing only the attachments that matter.

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

The original message appears below your content with a forwarding header. The HTML part carries the header lines styled the way Apple Mail formats a forward; when the forward also has a `text` part, it looks like this:

```text
Check this out:

Begin forwarded message:

From: Original Sender <sender@example.com>
Subject: Original Subject
Date: Wed, May 14, 2025 10:30 AM
To: original-recipient@example.com
>
> Original message content
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
      "content": "JVBERi0xLjQKJSBtaW5pbWFsIGV4YW1wbGUK",
      "contentType": "application/pdf"
    }
  ]
}
```

### Forward Without Original Content

Keep the threading headers of a forward but supply the whole body yourself:

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
  "html": "<p>Following up on the conversation with the customer from last week.</p>"
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

**Result:** The submit call succeeds, and the delivery attempt fails with `EENVELOPE` (`No recipients defined`), reported through the `messageFailed` webhook.

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

**Problem:** With `forwardAttachments`, EmailEngine downloads the original's attachments and builds a new message carrying them. If the total size exceeds what the account's SMTP server accepts, the delivery is rejected.

**Solution:**

- Use `forwardAttachments: false`, or reference only the attachments that matter
- Download attachments separately and host externally
- Inform users of size limits

### Timeout Issues

**Problem:** Building a forward with large attachments means downloading them from the mailbox first, and some hosting platforms cut connections they consider idle before that finishes.

**Solutions:**

- Raise the request timeout for the submit call with the `x-ee-timeout` header (milliseconds)
- Move EmailEngine off the constrained host

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
    "queueId": "24279fb3e0dff64e",
    "response": "250 2.0.0 Ok: queued",
    "envelope": {
      "from": "andris@example.com",
      "to": ["sender@example.com"]
    }
  }
}
```

The payload does not repeat the `reference`; correlate by `queueId` or `messageId` from the submit response.

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

### Skip the Quote When It Is Not Needed

Every reference fetches the original's headers. `inline` and `forwardAttachments` additionally fetch its body and attachments, so leave them off for an automated reply that only has to thread correctly.

## See Also

- [Threading](/docs/sending/threading) - The headers that keep a conversation together
- [Basic sending](/docs/sending/basic-sending) - Composing a message rather than answering one
- [Message operations](/docs/receiving/message-operations) - Finding the message ID a reference needs
- [Sending API](/docs/api-reference/sending-api) - The reference block in the endpoint reference
- [messageFailed](/docs/webhooks/messagefailed) - The event a forward without recipients ends in
