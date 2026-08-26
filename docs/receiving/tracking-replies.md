---
title: Tracking Email Replies
sidebar_position: 8
description: "Detect and track email replies using Message-ID and In-Reply-To headers with EmailEngine"
keywords:
  - email replies
  - tracking replies
  - message threading
  - in-reply-to
  - email tracking
---

# Tracking Email Replies

Reply tracking is how an integration learns that a recipient answered a message it sent. This guide shows how to send messages whose replies can be recognized, and how to match a reply in a `messageNew` webhook back to the message it answers.

## Why Track Replies?

**Lead Management**
- Identify leads who respond quickly
- Track engagement with sales emails
- Automate follow-ups based on responses

**Customer Support**
- Route replies to the correct ticket
- Track response times
- Close tickets automatically

**Email Campaigns**
- Measure actual engagement (not just opens)
- Identify interested prospects
- Segment by response behavior

**Workflow Automation**
- Trigger actions when replies arrive
- Update CRM records
- Send notifications to a team

## How Reply Tracking Works

Email threading uses standard headers:

**Message-ID** - Unique identifier of a message. You can set it when sending
```
Message-ID: <1697123456.k2x9p.eyJsZWFkSWQiOiJsZWFkLTQ1NiJ9@yourdomain.com>
```

**In-Reply-To** - The Message-ID of the message being replied to
```
In-Reply-To: <1697123456.k2x9p.eyJsZWFkSWQiOiJsZWFkLTQ1NiJ9@yourdomain.com>
```

**References** - The Message-IDs of the whole thread so far
```
References: <original@domain.com> <1697123456.k2x9p.eyJsZWFkSWQiOiJsZWFkLTQ1NiJ9@yourdomain.com>
```

When someone replies, their mail client sets `In-Reply-To` to your Message-ID and appends it to `References`. EmailEngine exposes the In-Reply-To value as `inReplyTo` on every message it reports, in the `messageNew` webhook and in the [get message](/docs/api/get-v-1-account-account-message-message) response, so a reply can be matched to the message it answers. The References header is available through `headers`.

## Sending Trackable Messages

### Step 1: Generate Unique Message-ID

Create a unique Message-ID when sending. The example packs tracking data into the local part, separated by dots; base64url output never contains a dot, so the parts can be split apart again:

```javascript
function generateMessageId(customData = {}) {
  const timestamp = Date.now();
  const random = Math.random().toString(36).substring(2, 8);

  // Include any data you want to track
  const data = Buffer.from(JSON.stringify(customData)).toString('base64url');

  return `<${timestamp}.${random}.${data}@yourdomain.com>`;
}

function extractMetadata(messageId) {
  const match = messageId.match(/^<[^.@]+\.[^.@]+\.([^@]+)@/);
  if (!match) {
    return {};
  }

  try {
    return JSON.parse(Buffer.from(match[1], 'base64url').toString());
  } catch (err) {
    return {};
  }
}

// Generate Message-ID for a sales email
const messageId = generateMessageId({
  campaignId: 'sales-2025-q4',
  leadId: 'lead-456',
  userId: 'user-789'
});

console.log(messageId);
// <1697123456789.k2x9p3.eyJjYW1wYWlnbklkIjoic2FsZXMtMjAyNS1xNCIsImxlYWRJZCI6ImxlYWQtNDU2IiwidXNlcklkIjoidXNlci03ODkifQ@yourdomain.com>

console.log(extractMetadata(messageId).leadId);
// lead-456
```

Use a domain you control after the `@`. Message-IDs are supposed to be globally unique, and the domain is what makes yours distinguishable from everyone else's.

### Step 2: Send with Message-ID

Send the email with your generated Message-ID through the [submit endpoint](/docs/api/post-v-1-account-account-submit). `messageId` sets the Message-ID header, and `headers` adds any further headers:

```javascript
async function sendTrackedEmail(accountId, recipient, subject, content, customData = {}) {
  // Generate unique Message-ID
  const messageId = generateMessageId({
    ...customData,
    recipient: recipient
  });

  // Send email
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/submit`,
    {
      method: 'POST',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        from: {
          name: 'Sales Team',
          address: 'sales@yourcompany.com'
        },
        to: [{
          address: recipient
        }],
        subject: subject,
        html: content,
        messageId: messageId,
        headers: {
          // Ask Exchange and Outlook not to send out-of-office replies
          'X-Auto-Response-Suppress': 'OOF, AutoReply'
        }
      })
    }
  );

  const result = await response.json();

  // Store Message-ID in your database
  await storeTrackedEmail({
    messageId: messageId,
    accountId: accountId,
    recipient: recipient,
    subject: subject,
    sentAt: new Date()
  });

  return {
    messageId,
    queueId: result.queueId
  };
}

// Send tracked email
const sent = await sendTrackedEmail(
  'example',
  'customer@example.com',
  'Special Offer for You',
  '<p>Hi, we have a special offer for you.</p>',
  { campaignId: 'sales-2025-q4', leadId: 'lead-456' }
);

console.log(`Sent with Message-ID: ${sent.messageId}`);
```

The response's `queueId` identifies the queued submission; `messageId` is what the reply will reference.

### Step 3: Store Message-IDs

Store Message-IDs in your database for matching:

```javascript
// In-memory stand-in for a database table keyed by Message-ID
const trackedEmails = new Map();

async function storeTrackedEmail(data) {
  trackedEmails.set(data.messageId, {
    ...data,
    replied: false,
    replyReceivedAt: null
  });
}

async function getTrackedEmail(messageId) {
  return trackedEmails.get(messageId);
}
```

## Detecting Replies

### Step 1: Configure Webhooks

Enable webhooks for new message events and include the threading headers in the payload:

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "webhooks": "https://your-app.com/webhooks/emailengine",
    "webhooksEnabled": true,
    "webhookEvents": ["messageNew"],
    "notifyHeaders": ["in-reply-to", "references", "message-id"]
  }'
```

`data.inReplyTo` is in the payload regardless of `notifyHeaders`. Listing the headers puts the raw values into `data.headers` as well, which is where `references` comes from. Header names in the payload are lower case.

### Step 2: Handle Webhook Events

Process incoming `messageNew` webhooks:

```javascript
const express = require('express');
const app = express();

app.use(express.json());

app.post('/webhooks/emailengine', async (req, res) => {
  const event = req.body;

  // Acknowledge immediately
  res.status(200).json({ success: true });

  // Process asynchronously
  if (event.event === 'messageNew') {
    await handleNewMessage(event);
  }
});

async function handleNewMessage(event) {
  // The folder path lives at the payload root, not inside the data object
  const { account, path, data } = event;

  // A reply references at least one earlier message
  const referenced = new Set();
  if (data.inReplyTo) {
    referenced.add(data.inReplyTo);
  }
  for (const value of (data.headers && data.headers.references) || []) {
    for (const id of value.split(/\s+/)) {
      if (id) {
        referenced.add(id);
      }
    }
  }

  if (referenced.size) {
    await handlePotentialReply(account, path, data, referenced);
  }
}

app.listen(3000);
```

### Step 3: Match Replies

Match the referenced Message-IDs to your stored ones:

```javascript
async function handlePotentialReply(accountId, path, message, referenced) {
  // Check if replying to one of our tracked messages. In-Reply-To is checked
  // first; References catches a reply to a reply, and clients that thread loosely
  let original = null;
  for (const messageId of referenced) {
    original = await getTrackedEmail(messageId);
    if (original) {
      break;
    }
  }

  if (!original) {
    // Not replying to a tracked message
    return;
  }

  // Ignore replies that landed in spam or trash
  const isInInbox = (
    path === 'INBOX' ||
    (message.labels && message.labels.includes('\\Inbox'))
  );

  if (!isInInbox) {
    console.log('Reply not in inbox, ignoring');
    return;
  }

  // Filter out auto-responses
  if (isAutoResponse(message)) {
    console.log('Auto-response detected, ignoring');
    return;
  }

  // Fetch the full message, including the body
  const fullMessage = await getMessage(accountId, message.id);

  // This is a genuine reply
  await handleReply(original, fullMessage);
}

async function getMessage(accountId, messageId) {
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/message/${messageId}?textType=*`,
    {
      headers: { 'Authorization': 'Bearer YOUR_ACCESS_TOKEN' }
    }
  );

  return await response.json();
}
```

## Filtering Auto-Responses

### Use EmailEngine's Detection

Every message EmailEngine reports carries `isAutoReply: true` when it looks like an automatic reply. The subject check matches a subject that starts with `Automatic reply`, `Auto reply`, `Autoreply`, `Auto response`, `Out of Office`, `Out of the Office`, `OOF:` or `OOO:`, and one that starts with `Auto:` when the message also carries an In-Reply-To header. The header check matches `Precedence` containing `auto_reply`, `auto-reply` or `autoreply`, `Auto-Submitted` containing `auto-replied`, and any value at all in `X-Auto-Response-Suppress`, `X-Autoresponder`, `X-Autorespond` or `X-Autoreply`. The flag is set on every account type. The get message response reads every header; for the `messageNew` payload EmailEngine always fetches `Precedence`, `Auto-Submitted`, `X-Autoreply` and `X-Autorespond`, while `X-Auto-Response-Suppress` and `X-Autoresponder` only count there when `notifyHeaders` includes them or is `["*"]`.

Bounces and mailing list traffic are not auto-replies in this sense, so check for those separately:

```javascript
function isAutoResponse(message) {
  if (message.isAutoReply) {
    return true; // Out of office, vacation responder, and similar
  }

  const headers = message.headers || {};

  // A null return path marks a bounce
  if (headers['return-path'] && headers['return-path'][0] === '<>') {
    return true;
  }

  // Mailing list traffic
  if (headers['list-id'] || headers['list-unsubscribe']) {
    return true;
  }

  // Bulk mail
  const precedence = headers['precedence'] && headers['precedence'][0];
  if (precedence && ['bulk', 'junk', 'list'].includes(precedence.toLowerCase())) {
    return true;
  }

  return false; // Appears to be a genuine reply
}
```

The header checks need those headers in the payload: add `return-path`, `list-id`, `list-unsubscribe` and `precedence` to `notifyHeaders`, or use `["*"]` to include every header. Bounces that EmailEngine itself recognizes also arrive as [`messageBounce`](/docs/webhooks/messagebounce) events.

### Check Sender

Verify the reply is from the expected recipient:

```javascript
function isValidReplySender(original, reply) {
  const originalRecipient = original.recipient.toLowerCase();
  const replyFrom = reply.from.address.toLowerCase();

  // Check if reply is from original recipient
  return replyFrom === originalRecipient;
}
```

Forwarded threads and shared mailboxes produce legitimate replies from other addresses, so treat a mismatch as a signal to review rather than a reason to drop the reply.

## Processing Replies

### Update Database

Mark message as replied:

```javascript
async function handleReply(original, reply) {
  // Update database
  await updateTrackedEmail(original.messageId, {
    replied: true,
    replyReceivedAt: new Date(),
    replyFrom: reply.from.address,
    replySubject: reply.subject,
    replyId: reply.id
  });

  console.log(`Reply received for ${original.messageId}`);
  console.log(`Original: ${original.subject}`);
  console.log(`Reply: ${reply.subject}`);

  // Trigger additional actions
  await onReplyReceived(original, reply);
}

async function updateTrackedEmail(messageId, updates) {
  const existing = trackedEmails.get(messageId);
  if (existing) {
    trackedEmails.set(messageId, { ...existing, ...updates });
  }
}
```

### Trigger Actions

Perform actions when replies are received. The fetched message carries its bodies under `text.plain` and `text.html`:

```javascript
async function onReplyReceived(original, reply) {
  // Extract metadata from Message-ID
  const metadata = extractMetadata(original.messageId);

  if (metadata.campaignId === 'sales-2025-q4') {
    // Update CRM: Mark lead as engaged
    await updateCRM(metadata.leadId, {
      status: 'hot',
      lastEngagement: new Date(),
      responseTime: Date.now() - original.sentAt.getTime()
    });

    // Notify sales team
    await sendNotification({
      type: 'lead-replied',
      lead: metadata.leadId,
      from: reply.from.address,
      subject: reply.subject
    });
  }

  // Store reply content for analysis
  await storeReplyContent({
    originalId: original.messageId,
    replyId: reply.id,
    content: reply.text && reply.text.plain,
    html: reply.text && reply.text.html
  });
}
```

`updateCRM`, `sendNotification` and `storeReplyContent` stand for your own application's functions.

## Putting It Together

The three steps above are the whole mechanism. What ties them together is a single stored row per sent message:

| Column | Written when | Used for |
|--------|--------------|----------|
| `messageId` | Sending | The join key. Matched against `inReplyTo` and the `references` header on incoming mail |
| `recipient`, `subject`, `sentAt` | Sending | Reporting, and calculating response time |
| `repliedAt`, `replyFrom` | On a matching `messageNew` | Marking the thread answered |

Two details decide whether this works in production:

- **Index on `messageId`.** Every inbound message triggers a lookup, so a table scan per webhook does not survive contact with a busy mailbox.
- **Check `references` as well as `inReplyTo`.** A reply from a client that threads loosely, or a reply to a reply, may carry your Message-ID only in `references`. Matching on `inReplyTo` alone silently misses those. `references` is a header, so it has to be requested with `notifyHeaders`.

Filter auto-responses before recording a reply, or vacation autoresponders will close out threads nobody read. See [Filtering Auto-Responses](#filtering-auto-responses) above.

## Advanced Patterns

### Track Multiple Recipients

Send one message per recipient so each gets its own Message-ID:

```javascript
async function sendTrackedToMultiple(accountId, recipients, subject, html) {
  const messageIds = [];

  for (const recipient of recipients) {
    const { messageId } = await sendTrackedEmail(
      accountId,
      recipient,
      subject,
      html
    );

    messageIds.push({ recipient, messageId });
  }

  return messageIds;
}

// Track who replied
const tracking = await sendTrackedToMultiple('example', [
  'customer1@example.com',
  'customer2@example.com',
  'customer3@example.com'
], 'Product Update', '<p>New features are available.</p>');

// Later, check replies
for (const { recipient, messageId } of tracking) {
  const tracked = trackedEmails.get(messageId);
  console.log(`${recipient}: ${tracked.replied ? 'REPLIED' : 'No reply yet'}`);
}
```

### Calculate Response Time

Track how quickly recipients respond:

```javascript
function calculateResponseTime(original, reply) {
  const sentTime = new Date(original.sentAt).getTime();
  const replyTime = new Date(reply.date).getTime();
  const responseTimeMs = replyTime - sentTime;

  const hours = Math.floor(responseTimeMs / (1000 * 60 * 60));
  const minutes = Math.floor((responseTimeMs % (1000 * 60 * 60)) / (1000 * 60));

  return {
    milliseconds: responseTimeMs,
    hours,
    minutes,
    formatted: `${hours}h ${minutes}m`
  };
}

// When reply received
const responseTime = calculateResponseTime(original, reply);
console.log(`Response time: ${responseTime.formatted}`);
```

`reply.date` is the time the mail server received the reply, not the time the sender's client stamped on it.

### Thread Multiple Replies

Track conversation threads:

```javascript
const threads = new Map();

async function recordInThread(original, reply) {
  let thread = threads.get(original.messageId);

  if (!thread) {
    thread = {
      originalId: original.messageId,
      originalSubject: original.subject,
      messages: [
        { id: original.messageId, type: 'sent', date: original.sentAt }
      ]
    };
    threads.set(original.messageId, thread);
  }

  thread.messages.push({
    id: reply.id,
    type: 'reply',
    from: reply.from.address,
    date: reply.date,
    subject: reply.subject
  });

  console.log(`Thread has ${thread.messages.length} messages`);
}
```

Where the server assigns thread IDs (Gmail, Microsoft Graph, and IMAP servers with the OBJECTID extension), `threadId` on the reply lets you fetch the whole conversation in one search instead of assembling it yourself. See [Searching threads](/docs/sending/threading/searching-threads).

## See Also

- [Threading](/docs/sending/threading) - The headers that make a reply a reply
- [messageNew](/docs/webhooks/messagenew) - The event a reply arrives on
- [Replies and forwards](/docs/sending/replies-forwards) - Answering from your own application
- [Searching messages](/docs/receiving/searching) - Finding the original a reply refers to
