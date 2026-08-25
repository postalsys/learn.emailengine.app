---
title: Transactional Email Service
sidebar_position: 9
description: Use EmailEngine as a transactional email service with API and SMTP delivery, scheduling, bounce detection, and webhook notifications
keywords:
  - transactional email
  - email delivery
  - SMTP relay
  - email scheduling
  - bounce detection
  - email queue
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Transactional Email Service

EmailEngine can function as a self-hosted transactional email service, allowing you to convert any email account into a reliable email delivery system. You can submit messages for delivery, schedule future sends, track delivery status, and receive bounce notifications.

## Overview

EmailEngine provides transactional email capabilities through:

- **Multiple Submission Methods**: Send via REST API or SMTP
- **Message Queuing**: Reliable delivery with automatic retry
- **Scheduled Sending**: Delay delivery to a specific future time
- **Bounce Detection**: Automatic bounce tracking and webhooks
- **Sent Mail Tracking**: Automatic upload to "Sent Mail" folder
- **Reply Threading**: Automatic "Answered" flag on replied messages

## Delivery via REST API

### Submit Endpoint

Submit emails using the `/v1/account/{account}/submit` endpoint. EmailEngine converts your structured JSON into a valid RFC822 MIME message.

**Endpoint**: `POST /v1/account/{account}/submit`

**Benefits**:
- No MIME knowledge required
- Unicode strings and base64 attachments
- Automatic header generation
- Reply threading support

### Basic Example

```bash
curl -XPOST "https://emailengine.example.com/v1/account/example/submit" \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from": {
      "name": "Example Sender",
      "address": "sender@example.com"
    },
    "to": [{
      "name": "John Doe",
      "address": "john@example.com"
    }],
    "subject": "Hello from EmailEngine",
    "text": "Plain text message",
    "html": "<p>HTML message</p>",
    "attachments": [
      {
        "filename": "document.pdf",
        "content": "BASE64_ENCODED_CONTENT"
      }
    ]
  }'
```

**Response**:

```json
{
  "response": "Queued for delivery",
  "messageId": "<188db4df-3abb-806c-94c8-7a9303652c50@example.com>",
  "sendAt": "2025-10-15T10:30:00.000Z",
  "queueId": "24279fb3e0dff64e"
}
```

### Reply to Existing Message

When replying to a message, EmailEngine automatically handles threading headers:

```bash
curl -XPOST "https://emailengine.example.com/v1/account/example/submit" \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "reference": {
      "message": "AAAAAQAAP1w",
      "action": "reply"
    },
    "from": {
      "name": "Support Team",
      "address": "support@example.com"
    },
    "to": [{
      "name": "Customer",
      "address": "customer@example.com"
    }],
    "text": "Thank you for your message. We will review and get back to you.",
    "html": "<p>Thank you for your message. We will review and get back to you.</p>"
  }'
```

**Automatic Handling**:
- Subject derived from original (with "Re:" prefix)
- `In-Reply-To` header set correctly
- `References` header populated
- Original message marked as "Answered"

You can override the subject if needed:

```json
{
  "reference": {
    "message": "AAAAAQAAP1w",
    "action": "reply"
  },
  "subject": "Custom reply subject",
  "text": "Reply content"
}
```

### Attachments

Include attachments with base64-encoded content:

```json
{
  "from": { "address": "sender@example.com" },
  "to": [{ "address": "recipient@example.com" }],
  "subject": "File attached",
  "text": "Please find the file attached.",
  "attachments": [
    {
      "filename": "report.pdf",
      "content": "JVBERi0xLjQKJeLjz9MKMSAwIG9iago8PAovVHlwZSAvQ2F0YWxvZwovUGFnZXMgMiAwI...",
      "contentType": "application/pdf"
    },
    {
      "filename": "image.png",
      "content": "iVBORw0KGgoAAAANSUhEUgAAABAAAAAQAQMAAAAlPW0iAAAABlBMVEUAAAD///+l2Z...",
      "contentType": "image/png",
      "cid": "unique-cid-123"
    }
  ]
}
```

**Attachment Properties**:
- `filename`: Name of the file
- `content`: Base64-encoded file content
- `contentType` (optional): MIME type (auto-detected if omitted)
- `cid` (optional): Content-ID for inline images

### Inline Images

Reference inline images in HTML using CID:

```json
{
  "from": { "address": "sender@example.com" },
  "to": [{ "address": "recipient@example.com" }],
  "subject": "Image email",
  "html": "<p>Check out this image:</p><img src=\"cid:logo-image\" />",
  "attachments": [
    {
      "filename": "logo.png",
      "content": "BASE64_ENCODED_IMAGE",
      "contentType": "image/png",
      "cid": "logo-image"
    }
  ]
}
```

## Delivery via SMTP

EmailEngine includes an optional SMTP server for standard email client integration.

### Enable SMTP Server

1. Navigate to **Configuration → SMTP Server**
2. Check **Enable SMTP Server**
3. Configure port (default: 2525)
4. Set authentication password
5. Save settings

**Important Notes**:
- Implicit TLS support available via the `smtpServerTLSEnabled` setting. STARTTLS is not supported - when TLS is enabled the server listens for TLS connections directly
- Can enable HAProxy PROXY protocol support
- Authentication optional but recommended

### Authentication

SMTP uses PLAIN authentication. Generate auth string:

```bash
# Format: \0{account_id}\0{password}
echo -ne "\0example\0your_password" | base64
# Output: AGV4YW1wbGUAeW91cl9wYXNzd29yZA==
```

### Manual SMTP Session

Test SMTP with telnet or netcat:

```bash
# Connect
telnet localhost 2525
# or
nc -c localhost 2525
```

**SMTP Commands**:

```
EHLO client.example.com
AUTH PLAIN AGV4YW1wbGUAeW91cl9wYXNzd29yZA==
MAIL FROM:<sender@example.com>
RCPT TO:<recipient@example.com>
DATA
From: sender@example.com
To: recipient@example.com
Subject: Test Email
X-EE-Send-At: 2025-10-16T14:00:00.000Z

This is the email body.
.
QUIT
```

**Response**: `250 Message queued for delivery as {queueId} ({timestamp})`

### SMTP Headers

EmailEngine recognizes special headers:

#### X-EE-Send-At

Schedule delivery for future time:

```
X-EE-Send-At: 2025-10-16T14:00:00.000Z
```

This header is removed before delivery.

#### X-EE-Account

Specify account when authentication disabled:

```
X-EE-Account: example
```

Required only if SMTP authentication is disabled.

### SMTP Client Example

<Tabs groupId="language">
<TabItem value="nodejs" label="Node.js" default>

```javascript
const nodemailer = require('nodemailer');

const transporter = nodemailer.createTransport({
  host: 'localhost',
  port: 2525,
  secure: false, // No TLS
  auth: {
    user: 'example', // Account ID
    pass: 'your_password'
  }
});

const message = {
  from: 'sender@example.com',
  to: 'recipient@example.com',
  subject: 'Test Email',
  text: 'Plain text content',
  html: '<p>HTML content</p>',
  // Schedule for future delivery
  headers: {
    'X-EE-Send-At': '2025-10-16T14:00:00.000Z'
  }
};

const info = await transporter.sendMail(message);
console.log('Message queued:', info.messageId);
```

</TabItem>
<TabItem value="python" label="Python">

```python
import smtplib
from email.message import EmailMessage

msg = EmailMessage()
msg['From'] = 'sender@example.com'
msg['To'] = 'recipient@example.com'
msg['Subject'] = 'Test Email'
msg['X-EE-Send-At'] = '2025-10-16T14:00:00.000Z'
msg.set_content('Plain text content')

with smtplib.SMTP('localhost', 2525) as smtp:
    smtp.login('example', 'your_password')
    smtp.send_message(msg)
    print('Message queued')
```

</TabItem>
</Tabs>

### Important SMTP Notes

- **Recipient addresses**: Only addresses in `RCPT TO` commands receive email
- **Header addresses**: `To`, `Cc`, `Bcc` headers are informational only
- **Bcc header**: Automatically removed from messages
- **Mandatory headers**: `Message-ID`, `MIME-Version`, `Date` added if missing
- **TLS**: Enable implicit TLS via the `smtpServerTLSEnabled` setting (STARTTLS is not supported), or use HAProxy for TLS termination

## Scheduled Sending

Delay message delivery to a specific future time.

### API Scheduling

Use the `sendAt` property with ISO timestamp:

```bash
curl -XPOST "https://emailengine.example.com/v1/account/example/submit" \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from": {
      "address": "sender@example.com"
    },
    "to": [{
      "address": "recipient@example.com"
    }],
    "subject": "Scheduled Email",
    "text": "This email was scheduled for delivery.",
    "sendAt": "2025-10-18T08:00:00.000Z"
  }'
```

**Response includes scheduled time**:

```json
{
  "response": "Queued for delivery",
  "messageId": "<uuid@example.com>",
  "sendAt": "2025-10-18T08:00:00.000Z",
  "queueId": "abc123"
}
```

### SMTP Scheduling

Add `X-EE-Send-At` header:

```
From: sender@example.com
To: recipient@example.com
Subject: Scheduled Email
X-EE-Send-At: 2025-10-18T08:00:00.000Z

This email will be sent at the scheduled time.
```

### Time Format

Use ISO 8601 format with timezone:

```
2025-10-18T08:00:00.000Z          # UTC
2025-10-18T08:00:00+02:00         # UTC+2
2025-10-18T08:00:00-05:00         # UTC-5
```

### Scheduling Limits

- Maximum schedule time: Configurable (default: no limit)
- Minimum schedule time: None (if `sendAt` is in the past, message sends immediately)
- Queue retention: Messages remain queued until `sendAt` time

## Webhook Notifications

EmailEngine sends webhook notifications for delivery events.

### messageSent

Triggered when SMTP server accepts the message:

```json
{
  "account": "example",
  "date": "2025-10-15T10:30:05.000Z",
  "event": "messageSent",
  "data": {
    "messageId": "<188db4df-3abb-806c-94c8-7a9303652c50@example.com>",
    "response": "250 2.0.0 OK queued as 1234ABCD",
    "queueId": "24279fb3e0dff64e",
    "envelope": {
      "from": "sender@example.com",
      "to": ["recipient@example.com"]
    }
  }
}
```

### messageDeliveryError

Triggered when delivery fails temporarily (will retry):

```json
{
  "account": "example",
  "date": "2025-10-15T10:30:05.000Z",
  "event": "messageDeliveryError",
  "data": {
    "queueId": "24279fb3e0dff64e",
    "messageId": "<188db4df-3abb-806c-94c8-7a9303652c50@example.com>",
    "envelope": {
      "from": "sender@example.com",
      "to": ["recipient@example.com"]
    },
    "error": "Connection timeout",
    "errorCode": "ETIMEDOUT",
    "smtpResponse": "421 4.4.2 Connection timed out",
    "smtpResponseCode": 421,
    "job": {
      "attemptsMade": 1,
      "nextAttempt": "2025-10-15T10:30:15.000Z"
    }
  }
}
```

### messageFailed

Triggered when delivery permanently fails (no more retries):

```json
{
  "account": "example",
  "date": "2025-10-15T10:30:05.000Z",
  "event": "messageFailed",
  "data": {
    "queueId": "24279fb3e0dff64e",
    "messageId": "<188db4df-3abb-806c-94c8-7a9303652c50@example.com>",
    "error": "Recipient address rejected",
    "response": "550 5.1.1 User unknown"
  }
}
```

### messageBounce

Triggered when a bounce message is detected in the mailbox:

```json
{
  "account": "example",
  "date": "2025-10-15T11:00:00.000Z",
  "event": "messageBounce",
  "data": {
    "bounceMessage": "AAAAAgAAxxk",
    "recipient": "invalid@example.com",
    "action": "failed",
    "response": {
      "source": "smtp",
      "message": "550 5.1.1 No such user",
      "status": "5.1.1"
    },
    "mta": "mx.example.com (192.168.1.1)",
    "messageId": "<19f1157c-d72b-50eb-74d5-d30f9ec816d3@example.com>"
  }
}
```

**Note**: Bounce notification includes both `messageId` and `queueId`, which you can use to correlate bounces with sent messages.

## Bounce Detection

EmailEngine automatically monitors IMAP accounts for bounce messages and parses delivery status notifications (DSN).

### How It Works

1. Message submitted and sent to SMTP server
2. SMTP server accepts message (`messageSent` webhook)
3. If delivery later fails, MTA sends bounce email to sender
4. EmailEngine detects bounce message in IMAP account
5. EmailEngine parses bounce and extracts details
6. `messageBounce` webhook sent with failure information

### Bounce Types

**Hard Bounce** (`action: "failed"`):
- Permanent delivery failure
- Invalid email address
- Domain doesn't exist
- Mailbox disabled

**Soft Bounce** (`action: "delayed"`):
- Temporary failure
- Mailbox full
- Server temporarily unavailable
- Will be retried by MTA

### Bounce Information

Bounce webhooks include:

- **recipient**: Failed recipient address
- **action**: `failed` (permanent) or `delayed` (temporary)
- **response.status**: SMTP status code (e.g., "5.1.1")
- **response.message**: Error message from server
- **mta**: Mail server that reported failure
- **messageId**: Original message ID
- **bounceMessage**: EmailEngine ID of the bounce email

### Tracking Bounces

To correlate bounces with sent messages:

**1. Store messageId when sending**:

```javascript
const response = await fetch('https://emailengine.example.com/v1/account/example/submit', {
  method: 'POST',
  headers: {
    'Authorization': 'Bearer TOKEN',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    from: { address: 'sender@example.com' },
    to: [{ address: 'recipient@example.com' }],
    subject: 'Test',
    text: 'Content'
  })
});

const data = await response.json();

// Store in database
await db.messages.insert({
  queueId: data.queueId,
  messageId: data.messageId,
  recipient: 'recipient@example.com',
  status: 'queued'
});
```

**2. Match bounce webhook to original**:

```javascript
// Webhook handler
app.post('/webhooks', async (req, res) => {
  const event = req.body;

  if (event.event === 'messageBounce') {
    const messageId = event.data.messageId;

    // Find original message
    const original = await db.messages.findOne({ messageId });

    if (original) {
      // Update status
      await db.messages.update(
        { messageId },
        {
          status: 'bounced',
          bounceReason: event.data.response.message,
          bounceStatus: event.data.response.status
        }
      );

      // Handle bounce (unsubscribe, notify, etc.)
      await handleBounce(original, event.data);
    }
  }

  res.json({ success: true });
});
```

## Queue Management

EmailEngine uses BullMQ for reliable message queuing.

### Queue Monitoring

View queue status in Bull Board:

1. Navigate to **System → Queues**
2. Select **Submit** queue
3. View job statuses:
   - **Waiting**: Ready to send immediately
   - **Delayed**: Scheduled for future or retry after failure
   - **Active**: Currently being sent
   - **Completed**: Successfully delivered
   - **Failed**: Permanently failed

### Retry Behavior

**Default Retry Strategy**:
EmailEngine uses exponential backoff with a base delay of 5 seconds. The delay before the next attempt is `2^(attempts made) x 5s`, so the interval doubles after each failure:

- Initial attempt: Immediate
- Retry 1: ~10 seconds later (2^1 x 5s)
- Retry 2: ~20 seconds later (2^2 x 5s)
- Retry 3: ~40 seconds later (2^3 x 5s)
- Retry 4: ~80 seconds later (2^4 x 5s)
- And so on, doubling each time

A small amount of jitter (up to 20%) is applied to each delay to avoid a thundering herd. Default maximum attempts: 10

Configure retry attempts in **Configuration → Email Processing → Retry Attempts**.

### Manual Queue Management

**Retry a failed job**:
1. Go to Bull Board → Submit queue → Failed
2. Find the job
3. Click **Retry**

**Remove a job**:
1. Go to Bull Board → Submit queue
2. Find the job in any status
3. Click **Delete**

**Pause queue**:
1. Go to Bull Board → Submit queue
2. Click **Pause**
3. All new jobs go to "Paused" status
4. Click **Resume** to continue

### Queue Performance

For high-volume sending:

- Monitor **Waiting** queue size
- If growing, increase worker concurrency
- Check SMTP server rate limits
- Review delivery errors in **Failed** tab

## See Also

- [SMTP server](/docs/sending/smtp-interface) - The submission interface this page relays through
- [Outbox queue](/docs/sending/outbox-queue) - Retry behavior and how to watch a backlog
- [Bounces](/docs/advanced/bounces) - Recognizing a rejection that arrives as mail
- [Blocklists](/docs/advanced/blocklists) - Suppressing addresses that have already bounced
- [Delivery testing](/docs/advanced/email-authentication-testing) - Checking SPF, DKIM, and DMARC before a campaign
