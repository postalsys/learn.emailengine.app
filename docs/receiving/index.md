---
title: Receiving Emails
sidebar_position: 1
description: "Receiving and processing email with EmailEngine: webhooks, real-time notifications, and mailbox operations"
keywords:
  - receiving emails
  - email webhooks
  - real-time processing
  - IMAP monitoring
  - mailbox operations
---

# Receiving Emails with EmailEngine

<!--
Source attribution:
- Original concept: EmailEngine documentation
- Enhanced with webhook and processing patterns
-->

EmailEngine watches connected mailboxes and reports what arrives, so a support desk, an analytics pipeline, or an automated workflow can act on new mail as it lands rather than polling for it.

## Why EmailEngine for Receiving Emails?

**Real-Time Processing**
- Instant webhook notifications for new messages
- No polling required - EmailEngine monitors mailboxes 24/7
- Process messages as they arrive, not minutes or hours later

**Multi-Provider Support**
- Works with any IMAP server
- Native Gmail API support with Cloud Pub/Sub
- Native Microsoft Graph API support with push notifications
- Unified API regardless of the underlying protocol

**Message Access**
- Full message content (headers, body, attachments)
- Message metadata (flags, labels, folder location)
- Threading information (message IDs, references)
- Attachment download capabilities

**Reliable Synchronization**
- Tracks message additions, updates, and deletions
- Handles folder changes and mailbox resets
- Recovers gracefully from connection issues
- Maintains state across restarts

## How EmailEngine Receives Messages

EmailEngine uses different mechanisms depending on the account type:

### IMAP Accounts

For IMAP accounts, EmailEngine:

1. **Establishes persistent connections** to the mail server
2. **Uses IDLE command** when supported for real-time notifications
3. **Falls back to polling** when IDLE is unavailable
4. **Tracks message UIDs** to detect new messages, updates, and deletions
5. **Emits webhooks** for all detected changes

### Gmail API Accounts

For Gmail accounts using the Gmail API:

1. **Subscribes to Cloud Pub/Sub** for push notifications
2. **Receives instant updates** from Google servers
3. **Fetches message details** when notified
4. **Emits webhooks** with complete message data

### Microsoft Graph API Accounts

For Outlook/Office 365 accounts using Graph API:

1. **Creates Graph API subscriptions** for mailbox events
2. **Receives push notifications** from Microsoft servers
3. **Fetches message details** when notified
4. **Emits webhooks** with complete message data

## Core Concepts

### Webhooks vs Polling

**Webhooks (Recommended)**
- EmailEngine pushes notifications to your application
- Real-time processing with minimal latency
- No need to repeatedly check for new messages
- Efficient and scalable

**API Polling (Alternative)**
- Your application periodically requests message lists
- Useful when webhooks cannot be configured
- Higher latency and less efficient
- Still fully supported via REST API

### Message States

Messages in EmailEngine have several properties you can track:

**Flags**
- `\Seen` - Message has been read
- `\Answered` - Reply has been sent
- `\Flagged` - Message is flagged/starred
- `\Draft` - Message is a draft
- `\Deleted` - Message is marked for deletion

**Labels (Gmail)**
- System labels: `\Inbox`, `\Sent`, `\Trash`, etc.
- Custom labels: User-created categories

**Folder Location**
- Messages can exist in multiple folders (some providers)
- Special-use folders detected automatically
- Custom folder hierarchies supported

## Common Use Cases

### Customer Support Systems

Receive support emails in real-time and:
- Create tickets automatically
- Assign to appropriate team members
- Track response times
- Monitor customer communication

### Email Analytics

Process all emails to:
- Extract sentiment and topics
- Generate summaries with AI
- Build searchable indexes
- Track conversation threads

### Automated Workflows

Trigger actions based on email content:
- Process order confirmations
- Extract invoice data
- Monitor shipping notifications
- Respond to specific keywords

### Backup and Archival

Continuously export emails to:
- External storage systems
- Vector databases for AI search
- Compliance archives
- Business intelligence tools

## Quick Start

### 1. Enable Webhooks

Configure EmailEngine to send webhooks to your application:

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "webhooks": "https://your-app.com/webhooks",
    "webhooksEnabled": true,
    "notifyWebSafeHtml": true
  }'
```

### 2. Listen for New Message Events

Set up an endpoint to receive webhooks:

```javascript
// Node.js with Express
app.post('/webhooks', async (req, res) => {
  const event = req.body;

  // Acknowledge receipt immediately
  res.json({ success: true });

  // Process the event asynchronously
  if (event.event === 'messageNew') {
    await processNewMessage(event);
  }
});

async function processNewMessage(event) {
  const { account, data } = event;
  const { id, subject, from, text } = data;

  console.log(`New message in ${account}:`);
  console.log(`From: ${from.address}`);
  console.log(`Subject: ${subject}`);
  console.log(`Message ID: ${id}`);

  // Text content is only included when webhook text payloads are enabled
  // (the notifyText setting); HTML is available at text.html
  if (text && text.html) {
    console.log(`HTML content: ${text.html.length} characters`);
  }

  // Your processing logic here
}
```

### 3. Fetch Full Message Details

If you need more information than what's in the webhook using the [Get Message API endpoint](/docs/api/get-v-1-account-account-message-message):

```javascript
async function fetchFullMessage(accountId, messageId) {
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/message/${messageId}`,
    {
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN'
      }
    }
  );

  const message = await response.json();
  return message;
}
```

### 4. Download Attachments

Process message attachments using the [Get Attachment API endpoint](/docs/api/get-v-1-account-account-attachment-attachment):

```javascript
async function downloadAttachment(accountId, attachmentId) {
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/attachment/${attachmentId}`,
    {
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN'
      }
    }
  );

  const buffer = await response.buffer();
  return buffer;
}
```

## Receiving Section Guide

This section covers all aspects of receiving and processing emails:

1. **[Webhooks](/docs/webhooks/overview)** - Setting up real-time notifications
2. **[Mailbox Operations](/docs/receiving/mailbox-operations)** - Working with folders and mailboxes
3. **[Message Operations](/docs/receiving/message-operations)** - Listing, fetching, and managing messages
4. **[Web-Safe HTML](/docs/receiving/web-safe-html)** - Displaying message HTML safely in a web page
5. **[Searching](/docs/receiving/searching)** - Finding messages with search queries
6. **[Attachments](/docs/receiving/attachments)** - Handling message attachments
7. **[Tracking Replies](/docs/receiving/tracking-replies)** - Detecting and handling reply emails
8. **[Tracking Deleted Messages](/docs/receiving/tracking-deleted)** - Monitoring message deletions
9. **[Continuous Processing](/docs/receiving/continuous-processing)** - Building real-time email processing pipelines
10. **[Exporting Messages](/docs/receiving/exporting)** - Bulk export with concurrency tuning


