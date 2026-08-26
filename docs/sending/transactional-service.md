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

# Transactional Email Service

EmailEngine can serve as a self-hosted transactional email service in front of any email account it manages. You submit messages over the REST API or SMTP, schedule them, route them through an SMTP gateway, follow delivery through webhooks, and get bounces reported back as they arrive in the mailbox.

## Overview

The pieces involved:

- **Two submission methods**: the REST API or the built-in SMTP server
- **Queuing**: every submission goes through the submit queue, with retries on temporary failures
- **Scheduled sending**: hold a message until a given time
- **Gateways**: deliver through a dedicated SMTP relay instead of the account's own server
- **Bounce detection**: a delivery status notification arriving in the mailbox becomes a `messageBounce` webhook
- **Sent Mail copy**: an SMTP delivery is uploaded to the account's Sent Mail folder unless disabled. Gmail and Microsoft Graph file the sent message themselves, so the `copy` flag is a no-op on those accounts
- **Reply flags**: a reply sent with `reference` flags the original as `\Answered`

## Delivery via REST API

### Submit Endpoint

Submit a message with [`POST /v1/account/{account}/submit`](/docs/api/post-v-1-account-account-submit). EmailEngine turns the JSON into an RFC 822 message, so you do not build MIME yourself: strings are Unicode, attachments are base64, and the headers a message needs are generated.

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
        "content": "JVBERi0xLjQKJeLjz9MKMSAwIG9iago8PAovVHlwZSAvQ2F0YWxvZwo+PgplbmRvYmoK"
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

With `reference`, EmailEngine takes the subject, the recipients, and the threading headers from the stored message:

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
    "text": "Thank you for your message. We will review and get back to you.",
    "html": "<p>Thank you for your message. We will review and get back to you.</p>"
  }'
```

**What EmailEngine fills in**:

- Subject from the original, with a `Re:` prefix
- `In-Reply-To` and `References`
- The recipients of a reply
- The `\Answered` flag on the original once the reply is sent

Any field you supply, such as `subject` or `to`, wins over the derived value:

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

See [Replies and forwards](/docs/sending/replies-forwards) for every `reference` option.

### Attachments

Attachments are base64-encoded in `content`:

```json
{
  "from": { "address": "sender@example.com" },
  "to": [{ "address": "recipient@example.com" }],
  "subject": "File attached",
  "text": "Please find the file attached.",
  "attachments": [
    {
      "filename": "report.pdf",
      "content": "JVBERi0xLjQKJeLjz9MKMSAwIG9iago8PAovVHlwZSAvQ2F0YWxvZwo+PgplbmRvYmoK",
      "contentType": "application/pdf"
    },
    {
      "filename": "image.png",
      "content": "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNkYPhfDwAChwGA60e6kgAAAABJRU5ErkJggg==",
      "contentType": "image/png",
      "cid": "unique-cid-123"
    }
  ]
}
```

**Attachment properties**:

- `filename`: the file name shown to the recipient
- `content`: the base64-encoded file content
- `contentType` (optional): MIME type, derived from `filename` if omitted
- `cid` (optional): Content-ID, for images referenced from the HTML

### Inline Images

Reference an inline image from the HTML through its `cid`:

```json
{
  "from": { "address": "sender@example.com" },
  "to": [{ "address": "recipient@example.com" }],
  "subject": "Image email",
  "html": "<p>Check out this image:</p><img src=\"cid:logo-image\" />",
  "attachments": [
    {
      "filename": "logo.png",
      "content": "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNkYPhfDwAChwGA60e6kgAAAABJRU5ErkJggg==",
      "contentType": "image/png",
      "cid": "logo-image"
    }
  ]
}
```

## Delivery via SMTP

EmailEngine includes an SMTP submission server, off by default, for software that speaks SMTP rather than HTTP. [SMTP server](/docs/sending/smtp-interface) is the reference for it; this section covers what a transactional setup needs.

### Enable the SMTP Server

1. Open **Configuration > SMTP Server** in the admin interface
2. Check **Enable SMTP Server**
3. Set **Port** (`2525` by default) and, if needed, **Listen Address**
4. Check **Require Authentication** and set a **Global Password**
5. Save

The same settings are `smtpServerEnabled`, `smtpServerPort`, `smtpServerHost`, `smtpServerAuthEnabled`, and `smtpServerPassword` on `POST /v1/settings`. `smtpServerTLSEnabled` switches the listener to implicit TLS; `STARTTLS` is never offered, in either mode. `smtpServerProxy` enables the HAProxy PROXY protocol for a load balancer in front.

The listener starts on `127.0.0.1:2525`, so it is reachable from the EmailEngine host only until **Listen Address** is widened.

### Authentication

The server accepts `AUTH PLAIN` and `AUTH LOGIN`. The username is the **account ID**, and the password is either the global password from the settings or an [access token](/docs/api-reference/access-tokens) with the `smtp` scope. To build the `PLAIN` credential by hand:

```bash
echo -ne "\0example\0your_password" | base64
```

With authentication switched off, the message has to name the account instead, in an `X-EE-Account` header that EmailEngine strips before delivery.

### Manual SMTP Session

Run this on the EmailEngine host, since the listener is bound to the loopback address by default:

```bash
nc localhost 2525
```

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

The server answers `DATA` with `250 Message queued for delivery as <queueId> (<sendAt>)`, where `queueId` is the same value the REST API returns.

### Control Headers

Submit options that a message can carry as headers are listed in full in [EmailEngine options as headers](/docs/sending/smtp-interface#emailengine-options-as-headers). For transactional sending the useful ones are:

- `X-EE-Send-At`: an ISO 8601 timestamp; the message is held until then
- `X-EE-Delivery-Attempts`: overrides the retry count for this message
- `X-EE-Gateway`: a gateway ID, see [Delivery through a gateway](#delivery-through-a-gateway)
- `X-EE-Tracking-Enabled`: turns open and click tracking on or off for this message
- `X-EE-Idempotency-Key`: collapses a repeated submission of the same message into one delivery
- `X-EE-Account`: the account to send through, only when authentication is disabled

Every control header is removed before the message goes out.

### What the SMTP Server Does with a Message

- Only the `RCPT TO` addresses receive the message; `To`, `Cc`, and `Bcc` are informational
- A `Bcc` header is removed before delivery
- `Message-ID`, `Date`, and `MIME-Version` are added if missing
- With an `X-EE-Send-At` in the future, the `Date` header is rewritten to the scheduled time

## Delivery through a Gateway

By default a message goes out through the account's own SMTP server or provider API. An SMTP gateway is a separate relay registered with EmailEngine, for bulk or transactional mail that should not count against the mailbox's own sending limits.

Register one with [`POST /v1/gateway`](/docs/api/post-v-1-gateway). It takes `gateway` (the ID you will refer to it by), `name`, `host` and `port` as required fields, plus the optional `user`, `pass` and `secure` (`true` for implicit TLS, usually on port 465). Four more endpoints manage the registry:

| Endpoint | Purpose |
|----------|---------|
| [`GET /v1/gateways`](/docs/api/get-v-1-gateways) | List the registered gateways |
| [`GET /v1/gateway/{gateway}`](/docs/api/get-v-1-gateway-gateway) | Read one gateway, including its last use and last error |
| [`PUT /v1/gateway/edit/{gateway}`](/docs/api/put-v-1-gateway-edit-gateway) | Change host, port, credentials or name |
| [`DELETE /v1/gateway/{gateway}`](/docs/api/delete-v-1-gateway-gateway) | Remove it |

Name the gateway on submit to route a message through it:

```json
{
  "from": { "address": "sender@example.com" },
  "to": [{ "address": "recipient@example.com" }],
  "subject": "Order confirmation",
  "text": "Your order has shipped.",
  "gateway": "transactional-relay"
}
```

Over SMTP the equivalent is the `X-EE-Gateway` header. The message still belongs to the account: its Sent Mail copy, bounce detection, and webhooks work as for any other submission.

## Scheduled Sending

### API Scheduling

Set `sendAt` to an ISO 8601 timestamp:

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

The response echoes the time:

```json
{
  "response": "Queued for delivery",
  "messageId": "<3b1d4c2a-7f0e-4d55-9a1c-2c9e7f8b6a10@example.com>",
  "sendAt": "2025-10-18T08:00:00.000Z",
  "queueId": "24279fb3e0dff64e"
}
```

### SMTP Scheduling

Add an `X-EE-Send-At` header:

```
From: sender@example.com
To: recipient@example.com
Subject: Scheduled Email
X-EE-Send-At: 2025-10-18T08:00:00.000Z

This email will be sent at the scheduled time.
```

### Time Format

ISO 8601 with a timezone designator:

```
2025-10-18T08:00:00.000Z
2025-10-18T08:00:00+02:00
2025-10-18T08:00:00-05:00
```

### How Scheduling Behaves

- A `sendAt` in the past sends immediately
- There is no upper bound on how far ahead a message can be scheduled
- Until `sendAt`, the message sits in the submit queue as a delayed job and is listed by the [outbox API](/docs/sending/outbox-queue#outbox-api), which is also where it can be deleted

## Webhook Notifications

Delivery is reported through four events. Each has a reference page with the full payload; the fields that matter for a transactional integration are:

| Event | When | Fields to correlate on |
|-------|------|------------------------|
| [`messageSent`](/docs/webhooks/messagesent) | The receiving server accepted the message | `messageId`, `queueId`, `response`, and `originalMessageId`, which differs from `messageId` when the server rewrote it |
| [`messageDeliveryError`](/docs/webhooks/messagedeliveryerror) | A delivery attempt failed; another is scheduled unless it was the last | `queueId`, `messageId`, `error`, `errorCode`, `smtpResponse`, `smtpResponseCode`, and `job.nextAttempt` |
| [`messageFailed`](/docs/webhooks/messagefailed) | The job is over: the attempts ran out, or a permanent rejection ended it early | `queueId`, `messageId`, `error` |
| [`messageBounce`](/docs/webhooks/messagebounce) | A delivery status notification arrived in the mailbox | `messageId` of the bounced message, `recipient`, `action`, `response.status`, `bounceMessage` |

A `messageSent` example:

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

`messageDeliveryError` is emitted on every failed attempt, including the final one, which is then followed by `messageFailed`. The `messageBounce` payload identifies the original message by its `messageId`; its `queueId` field, when present, is the queue ID reported by the bouncing MTA, not EmailEngine's.

## Bounce Detection

EmailEngine watches the account's mailbox for delivery status notifications and reports them as `messageBounce` webhooks.

### How It Works

1. The message is submitted and handed to the SMTP server or provider API
2. The server accepts it, and `messageSent` is emitted
3. Delivery to the final recipient fails later, and the responsible MTA sends a bounce to the sender address
4. The bounce arrives in the monitored mailbox
5. EmailEngine parses it and emits `messageBounce` with the failed recipient, the status, and the `Message-ID` of the original message

### Bounce Types

**Hard bounce** (`action: "failed"`):
- Permanent failure
- Unknown or invalid address
- Domain does not exist
- Mailbox disabled

**Soft bounce** (`action: "delayed"`):
- Temporary failure
- Mailbox full
- Server temporarily unavailable
- The MTA keeps retrying

### Bounce Information

A `messageBounce` payload carries:

- **recipient**: the address that failed
- **action**: `failed` (permanent) or `delayed` (temporary)
- **response.status**: the enhanced status code, for example `5.1.1`
- **response.message**: the server's diagnostic text
- **mta**: the server that reported the failure
- **messageId**: the `Message-ID` of the original message
- **bounceMessage**: the EmailEngine ID of the bounce email itself

The webhook is sent only when the report yields `action`, `recipient` and `messageId` together; a bounce that cannot be tied to a sent message is logged instead. With classification enabled, `response` also carries `category`, `recommendedAction`, `blocklist` and `retryAfter`. [Bounces](/docs/advanced/bounces) has the full field list and the categories.

### Tracking Bounces

To correlate a bounce with the message it is about:

**1. Store the `messageId` when sending**:

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

await db.messages.insert({
  queueId: data.queueId,
  messageId: data.messageId,
  recipient: 'recipient@example.com',
  status: 'queued'
});
```

If the sending server rewrites Message-IDs, update the stored value from the `messageSent` webhook whenever its `messageId` differs from its `originalMessageId`, or the bounce will name an ID you never stored. [Message-ID rewriting](/docs/sending/threading/sending-threaded#message-id-rewriting-by-mail-servers) lists the servers EmailEngine can detect this for.

**2. Match the bounce webhook to the original**:

```javascript
app.post('/webhooks', async (req, res) => {
  const event = req.body;

  if (event.event === 'messageBounce') {
    const messageId = event.data.messageId;

    const original = await db.messages.findOne({ messageId });

    if (original) {
      await db.messages.update(
        { messageId },
        {
          status: 'bounced',
          bounceReason: event.data.response.message,
          bounceStatus: event.data.response.status
        }
      );

      await handleBounce(original, event.data);
    }
  }

  res.json({ success: true });
});
```

## Queue Management

Submissions are BullMQ jobs in the `submit` queue. [Outbox queue](/docs/sending/outbox-queue) is the reference for the queue itself; the parts a transactional integration touches are below.

### Queue Monitoring

Open **System > Queues** in the admin interface for the Bull Board dashboard and select the **submit** queue. A job is in one of:

- **Waiting**: ready to send
- **Delayed**: scheduled for later, or waiting out a retry delay
- **Active**: being sent
- **Completed**: delivered
- **Failed**: no attempts left

### Retry Behavior

A temporary failure is retried with exponential backoff: 5 seconds after the first failed attempt, then 10, 20, 40 seconds and so on, with jitter shortening each delay by up to 20% so that a batch of failures does not retry in lockstep. The default is 10 attempts, set globally as **Retry Attempts** under **Configuration > Email Processing** (`deliveryAttempts` on `POST /v1/settings`) and per message with `deliveryAttempts` on submit or the `X-EE-Delivery-Attempts` header. A permanent rejection, an SMTP 5xx other than 503, ends the job before the attempts run out. [Outbox queue](/docs/sending/outbox-queue) has the full classification.

### Manual Queue Management

From Bull Board, a failed job can be retried and a job in any state can be removed. Pausing the queue holds every waiting job until it is resumed; the same is available as `PUT /v1/settings/queue/submit` with `{"paused": true}` or `{"paused": false}`. A queued message can also be removed with `DELETE /v1/outbox/{queueId}`.

### Queue Performance

For high-volume sending:

- Watch the **Waiting** count; a growing backlog means submissions outpace delivery
- Check the receiving server's rate limits before raising concurrency; see [Performance tuning](/docs/advanced/performance-tuning)
- Review the **Failed** tab for the rejections behind repeated `messageDeliveryError` events

## See Also

- [SMTP server](/docs/sending/smtp-interface) - The submission interface this page relays through
- [Outbox queue](/docs/sending/outbox-queue) - Retry behavior and how to watch a backlog
- [Bounces](/docs/advanced/bounces) - Recognizing a rejection that arrives as mail
- [Blocklists](/docs/advanced/blocklists) - Suppressing addresses that have already bounced
- [Delivery testing](/docs/advanced/email-authentication-testing) - Checking SPF, DKIM, and DMARC before a campaign
