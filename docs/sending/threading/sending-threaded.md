---
title: Sending Threaded Messages
sidebar_position: 4
description: How to send multiple emails in the same conversation thread
---

# Sending Threaded Messages

To keep a sequence of outgoing messages in one conversation, control the `Message-ID` and `References` headers. This page walks through a three-message sequence, then covers when to let EmailEngine build the headers instead, and how to find out when a receiving server rewrote a `Message-ID`.

## Why Manual Threading is Needed

Mail clients decide which messages belong together from the RFC 5322 `Message-ID` and `References` headers. If every message in a sequence gets a generated `Message-ID` and no `References`, each one lands as a separate conversation. Setting the headers yourself keeps every follow-up under the first message.

## Step-by-Step: Sending a Thread

### Step 1: Send the Initial Message

Send the first message with your own `messageId`, using the [Submit endpoint](/docs/api/post-v-1-account-account-submit):

```bash
curl -XPOST "https://emailengine.example.com/v1/account/demo/submit" \
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

Save the `messageId`; every follow-up needs it.

**Response:**

```json
{
  "response": "Queued for delivery",
  "messageId": "<56b3c6d2-f7c0-4272-8beb-e25fdb7c19f1@example.com>",
  "sendAt": "2025-10-15T10:30:00.000Z",
  "queueId": "1a2b3c4d5e6f7a8b"
}
```

If you leave `messageId` out, EmailEngine generates one and returns it here. Either way, the value to store is the one in the response.

### Step 2: Send the First Follow-Up

Send a follow-up with the first message's ID in the `References` header:

```bash
curl -XPOST "https://emailengine.example.com/v1/account/demo/submit" \
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

- A new, unique `messageId` for this message
- `references` carries the first message's ID
- The subject stays the same

### Step 3: Keep Extending References

Each further message appends every previous `Message-ID` to `References`:

```bash
curl -XPOST "https://emailengine.example.com/v1/account/demo/submit" \
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

- Space-separated list of `Message-ID`s
- Each ID wrapped in angle brackets `< >`
- Oldest first, newest last

EmailEngine sends a `references` header you supply exactly as written, with no reordering and no length limit of its own.

## Using the Reference API

For replies and forwards to a message EmailEngine can see, the `reference` field does the header work:

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

EmailEngine then:

- Builds `References` from the referenced message's `Message-ID`, `In-Reply-To`, and `References`, adding missing angle brackets and dropping duplicates
- Sets `In-Reply-To` to the referenced message's `Message-ID` for `reply` and `reply-all`
- Derives the subject with a `Re:` or `Fwd:` prefix unless you supply one
- Fills in the recipients of a reply
- Flags the referenced message `\Answered` once the new message is sent, adding `$Forwarded` as well when the action was `forward`

On a **Gmail API** account, `reference.threadId` attaches the outgoing message to a Gmail thread directly. It is the thread ID from the message listing, and it can stand on its own without `reference.message`; a `reference` object has to carry one or the other. When both are given, `threadId` wins over the thread of the referenced message. IMAP and Microsoft Graph accounts ignore it, because those backends thread on the headers.

**When to use the reference API:**

- Replying to a received message
- Forwarding a stored message
- You have the EmailEngine message ID (the `id` field, for example `"AAAADQAABl0"`, not the `Message-ID` header)

**When to build the headers yourself:**

- Sending a sequence of follow-ups from scratch
- Drip campaigns and scheduled sequences
- There is no stored message to reference

See [Replies and forwards](/docs/sending/replies-forwards) for every `reference` option.

## Common Pitfalls

### Missing Angle Brackets

**Problem:** IDs in `References` without `< >`.

**Result:** Many clients fail to match them.

**Solution:** Store IDs with the brackets and keep them when building `References`. EmailEngine adds missing brackets only on the `reference` path, not to headers you set yourself.

### Subject Drift

**Problem:** Changing the subject text between messages in a thread.

**Result:** Some clients split the thread despite correct headers.

**Solution:** Keep the subject text the same. A `Re:` or `Fwd:` prefix is fine (EmailEngine adds one on the `reference` path); changing the wording, for example "Meeting on Monday" to "Meeting on Tuesday", is not.

### Not Persisting Message IDs

**Problem:** Sending without storing the `messageId` from the response.

**Result:** No way to build `References` for the follow-up.

**Solution:** Store every `messageId` as soon as the submit response arrives, indexed by thread.

### Message-ID Rewriting by Mail Servers

**Problem:** Some servers replace the `Message-ID` you set. The recipient then sees a different ID from the one you stored, and a follow-up that references the stored one does not thread.

**Detection:** the `messageSent` webhook carries the final `Message-ID` as `messageId` and the one EmailEngine sent as `originalMessageId`. The two differ when the receiving server rewrote it:

```json
{
  "account": "user@example.com",
  "date": "2025-01-15T10:30:00.000Z",
  "event": "messageSent",
  "data": {
    "messageId": "<rewritten-id@mailserver.com>",
    "originalMessageId": "<56b3c6d2-f7c0-4272-8beb-e25fdb7c19f1@example.com>",
    "response": "250 2.0.0 OK",
    "queueId": "1a2b3c4d5e6f7a8b",
    "envelope": {
      "from": "sender@example.com",
      "to": ["recipient@example.com"]
    }
  }
}
```

EmailEngine learns the final ID in these cases:

- **Outlook and Microsoft 365 SMTP**: the `250 2.0.0 OK` reply names the new ID when it ends in `.prod.outlook.com`, and EmailEngine reads it from there
- **Amazon SES SMTP**: the `250 Ok <uuid>` reply carries the SES message ID, from which EmailEngine builds `<uuid@<region>.amazonses.com>`. Recognized when the SMTP host ends in `.amazonaws.com` or `.awsapps.com`
- **Gmail API**: EmailEngine reads the sent message back to get the `Message-ID` Gmail stored

**Solution:** Handle `messageSent`, and whenever `messageId` differs from `originalMessageId`, replace the stored ID with `messageId`. Use that value in every later `References` header. On an SMTP delivery `originalMessageId` is present only when the ID was rewritten; on Gmail API and Microsoft Graph accounts it is always present, so compare the two rather than testing for its presence.

**Limitation:** a **Gmail API account with the send-only scope** (`gmail.send`) cannot read the sent message back, so `messageId` in the webhook is the ID you set, and a rewrite goes unnoticed. Register Gmail API accounts with the full scope if you rely on this detection. Any other server that rewrites the ID without reporting it in its SMTP reply is likewise not detected.

## See Also

- [Threading overview](/docs/sending/threading/overview) - What each header does
- [Replies and forwards](/docs/sending/replies-forwards) - The automatic alternative to building headers
- [messageSent webhook](/docs/webhooks/messagesent) - The full payload, including `originalMessageId`
- [Mail merge](/docs/sending/mail-merge) - Sending a sequence to many recipients
- [Basic sending](/docs/sending/basic-sending) - The submit fields used here
