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

Reply tracking is essential when building email integrations that need to know when recipients respond to your messages. This guide shows you how to send trackable emails and reliably detect replies using EmailEngine.

## Why Track Replies?

**Lead Management**
- Identify hot leads who respond quickly
- Track engagement with sales emails
- Automate follow-ups based on responses

**Customer Support**
- Route replies to correct ticket
- Track response times
- Close tickets automatically

**Email Campaigns**
- Measure actual engagement (not just opens)
- Identify interested prospects
- Segment by response behavior

**Workflow Automation**
- Trigger actions when replies arrive
- Update CRM records
- Send notifications to team

## How Reply Tracking Works

Email threading uses standard headers:

**Message-ID** - Unique identifier you assign when sending
```
Message-ID: <1697123456-account@yourdomain.com>
```

**In-Reply-To** - References the Message-ID being replied to
```
In-Reply-To: <1697123456-account@yourdomain.com>
```

**References** - Complete thread history
```
References: <original@domain.com> <1697123456-account@yourdomain.com>
```

When someone replies, their email client automatically sets `In-Reply-To` to your `Message-ID`. EmailEngine captures this in webhooks, allowing you to match replies to original messages.

## Sending Trackable Messages

### Step 1: Generate Unique Message-ID

Create a unique Message-ID when sending:

```javascript
function generateMessageId(accountId, customData = {}) {
  const timestamp = Date.now();
  const random = Math.random().toString(36).substring(7);

  // Include any data you want to track
  const data = Buffer.from(JSON.stringify(customData)).toString('base64url');

  return `<${timestamp}-${accountId}-${random}-${data}@yourdomain.com>`;
}

// Generate Message-ID for a sales email
const messageId = generateMessageId('account123', {
  campaignId: 'sales-2025-q4',
  leadId: 'lead-456',
  userId: 'user-789'
});

console.log(messageId);
// <1697123456-account123-x3k2p-eyJjYW1wYWlnbklkIjoic2FsZXMtMjAyNS1xNCJ9@yourdomain.com>
```

### Step 2: Send with Message-ID

Send the email with your generated Message-ID:

```javascript
async function sendTrackedEmail(accountId, recipient, subject, content) {
  // Generate unique Message-ID
  const messageId = generateMessageId(accountId, {
    recipient: recipient,
    sentAt: new Date().toISOString()
  });

  // Send email
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/submit`, // See: /docs/api/post-v-1-account-account-submit
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
          // Suppress out-of-office auto-replies
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
const result = await sendTrackedEmail(
  'account123',
  'customer@example.com',
  'Special Offer for You',
  '<p>Hi, we have a special offer...</p>'
);

console.log(`Sent with Message-ID: ${result.messageId}`);
```

### Step 3: Store Message-IDs

Store Message-IDs in your database for matching:

```javascript
// Database schema (example)
const trackedEmails = new Map();

async function storeTrackedEmail(data) {
  trackedEmails.set(data.messageId, {
    ...data,
    replied: false,
    replyReceivedAt: null
  });

  // In production, save to database:
  // await db.trackedEmails.insert(data);
}

async function getTrackedEmail(messageId) {
  return trackedEmails.get(messageId);

  // In production:
  // return await db.trackedEmails.findOne({ messageId });
}
```

## Detecting Replies

### Step 1: Configure Webhooks

Enable webhooks for new message events:

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "webhooks": "https://your-app.com/webhooks/emailengine",
    "webhooksEnabled": true,
    "webhookEvents": ["messageNew"],
    "notifyHeaders": ["In-Reply-To", "References", "Message-ID"]
  }'
```

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

  // Check if this is a reply
  if (data.inReplyTo) {
    await handlePotentialReply(account, path, data);
  }
}

app.listen(3000);
```

### Step 3: Match Replies

Match the `In-Reply-To` header to your stored Message-IDs:

```javascript
async function handlePotentialReply(accountId, path, message) {
  const inReplyTo = message.inReplyTo;

  // Check if replying to one of our tracked messages
  const original = await getTrackedEmail(inReplyTo);

  if (!original) {
    // Not replying to a tracked message
    return;
  }

  // Verify it's in inbox (not spam/trash)
  const isInInbox = (
    path === 'INBOX' ||
    (message.labels && message.labels.includes('\\Inbox'))
  );

  if (!isInInbox) {
    console.log('Reply not in inbox, ignoring');
    return;
  }

  // Fetch full message details
  const fullMessage = await getMessage(accountId, message.id);

  // Filter out auto-responses
  if (await isAutoResponse(fullMessage)) {
    console.log('Auto-response detected, ignoring');
    return;
  }

  // This is a genuine reply!
  await handleReply(original, fullMessage);
}
```

## Filtering Auto-Responses

### Check Auto-Response Headers

Filter out automated messages:

```javascript
async function isAutoResponse(message) {
  const headers = message.headers || {};

  // Check Return-Path for bounces
  if (headers['return-path']?.[0] === '<>') {
    return true; // Bounce message
  }

  // Check Auto-Submitted header
  const autoSubmitted = headers['auto-submitted']?.[0];
  if (autoSubmitted && autoSubmitted.toLowerCase() !== 'no') {
    return true; // Auto-generated
  }

  // Check for out-of-office in subject
  const subject = (message.subject || '').toLowerCase();
  if (
    subject.includes('out of office') ||
    subject.includes('automatic reply') ||
    subject.includes('auto:') ||
    subject.includes('away:')
  ) {
    return true;
  }

  // Check for mailing list headers
  if (headers['list-id'] || headers['list-unsubscribe']) {
    return true; // Mailing list
  }

  // Check precedence header
  const precedence = headers['precedence']?.[0];
  if (precedence && ['bulk', 'junk', 'list'].includes(precedence.toLowerCase())) {
    return true;
  }

  return false; // Appears to be genuine reply
}
```

:::info Header Availability
Full message headers are not available for Microsoft Graph API (Outlook) accounts, so this header-based filtering pattern applies to IMAP and Gmail accounts. For MS Graph accounts, rely on subject-based checks and sender verification instead.
:::

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

  // In production:
  // await db.trackedEmails.update({ messageId }, { $set: updates });
}
```

### Trigger Actions

Perform actions when replies are received:

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
    content: reply.text,
    html: reply.html,
    sentiment: await analyzeSentiment(reply.text)
  });
}

function extractMetadata(messageId) {
  // Extract data from Message-ID
  const match = messageId.match(/<[\d-]+-([\w-]+)-([^@]+)@/);
  if (match && match[2]) {
    const data = Buffer.from(match[2], 'base64url').toString();
    return JSON.parse(data);
  }
  return {};
}
```

## Putting It Together

The three steps above are the whole mechanism. What ties them together is a single stored row per sent message:

| Column | Written when | Used for |
|--------|--------------|----------|
| `messageId` | Sending | The join key. Matched against `inReplyTo` and `references` on incoming mail |
| `recipient`, `subject`, `sentAt` | Sending | Reporting, and calculating response time |
| `repliedAt`, `replyFrom` | On a matching `messageNew` | Marking the thread answered |

Two details decide whether this works in production:

- **Index on `messageId`.** Every inbound message triggers a lookup, so a table scan per webhook does not survive contact with a busy mailbox.
- **Check `references` as well as `inReplyTo`.** A reply from a client that threads loosely, or a reply to a reply, may carry your Message-ID only in `references`. Matching on `inReplyTo` alone silently misses those.

Filter auto-responses before recording a reply, or vacation autoresponders will close out threads nobody read. See [Filtering Auto-Responses](#filtering-auto-responses) above.

## Advanced Patterns

### Track Multiple Recipients

Handle emails sent to multiple recipients:

```javascript
async function sendTrackedToMultiple(accountId, recipients, subject, html) {
  const messageIds = [];

  for (const recipient of recipients) {
    const messageId = await sendTrackedEmail(
      accountId,
      recipient,
      subject,
      html,
      { recipient }
    );

    messageIds.push({ recipient, messageId });
  }

  return messageIds;
}

// Track who replied
const tracking = await sendTrackedToMultiple('account123', [
  'customer1@example.com',
  'customer2@example.com',
  'customer3@example.com'
], 'Product Update', '<p>New features...</p>');

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

// Track average response times
await analytics.trackResponseTime({
  campaignId: original.metadata.campaignId,
  responseTime: responseTime.milliseconds
});
```

### Thread Multiple Replies

Track conversation threads:

```javascript
const threads = new Map();

async function handleReply(original, reply) {
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

