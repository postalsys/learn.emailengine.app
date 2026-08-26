---
title: "messageSent"
sidebar_position: 7
description: "Webhook event triggered when a queued email is successfully accepted by the SMTP server"
---

# messageSent

The `messageSent` webhook event is triggered when a queued email is accepted by the SMTP server, the Gmail API or the Microsoft Graph API. This event confirms that the message has been handed off to the provider for delivery.

## When This Event is Triggered

The `messageSent` event fires once per queued message when:

- The SMTP server accepts the message at the end of the `DATA` command
- The Gmail API accepts the message (Gmail API accounts)
- The Microsoft Graph API accepts the message (MS Graph accounts)

Which of those applies is decided per submission. A Gmail API or MS Graph account that submits through an [SMTP gateway](/docs/sending/transactional-service) takes the SMTP path, and its `messageSent` payload then carries `response` and `networkRouting` like any other SMTP submission.

This event indicates a successful handoff. It does not guarantee delivery to the recipient's inbox: the message may still bounce or be rejected by downstream servers, which surfaces as a [`messageBounce`](/docs/webhooks/messagebounce) event if the bounce lands in a monitored mailbox.

Messages queued through the [Submit API](/docs/api/post-v-1-account-account-submit) and through the [SMTP interface](/docs/sending/smtp-interface) both produce this event.

## Common Use Cases

- **Delivery confirmation** - Update your application when emails are successfully queued for delivery
- **Tracking correlation** - Associate the EmailEngine queue ID with the MTA's message ID
- **Audit logging** - Maintain a log of all successfully sent emails
- **Workflow triggers** - Initiate follow-up actions after email is sent
- **Analytics** - Track send volumes and success rates per account
- **CRM integration** - Update contact records with communication history

## Payload Schema

### Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `serviceUrl` | string or null | Yes | The configured EmailEngine service URL, `null` if not set |
| `account` | string | Yes | Account ID that sent the message |
| `date` | string | Yes | ISO 8601 timestamp when the webhook was generated |
| `event` | string | Yes | Always `messageSent` |
| `data` | object | Yes | Event data object (see below) |

The event carries no `path` or `specialUse`. The unique event identifier is sent as the HTTP header `X-EE-Wh-Event-Id`, not in the JSON payload.

### Data Object Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `messageId` | string | Yes | Message-ID of the sent email as the provider knows it (see below) |
| `originalMessageId` | string | No | The Message-ID EmailEngine generated for the message. SMTP submissions carry it only when the server assigned a different `messageId`; Gmail API and MS Graph submissions always carry it |
| `response` | string | SMTP only | The SMTP server's final response line. Absent for Gmail API and Graph submissions |
| `queueId` | string | Yes | EmailEngine's queue ID for this submission, the same value the Submit API returned |
| `envelope` | object | Yes | SMTP envelope with sender and recipients |
| `networkRouting` | object or null | SMTP only | Local address and proxy used for the SMTP connection, `null` when neither was configured. Absent for Gmail API and Graph submissions (see below) |

### Envelope Object Structure

| Field | Type | Description |
|-------|------|-------------|
| `from` | string | Envelope sender (`MAIL FROM` address) |
| `to` | array | Envelope recipient addresses (`RCPT TO`) |

### Network Routing Object Structure

Sent as `null` unless a [local address](/docs/advanced/local-addresses) or a proxy was used:

| Field | Type | Description |
|-------|------|-------------|
| `localAddress` | string | Local IP address used for the SMTP connection |
| `proxy` | string | Proxy URL used for the connection |
| `name` | string | Hostname used in the `EHLO` greeting |
| `requestedLocalAddress` | string | The `localAddress` requested in the submission, when a different one was used |

## Example Payload

### Standard SMTP Submission

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T06:46:26.954Z",
  "event": "messageSent",
  "data": {
    "messageId": "<305eabf4-9538-2747-acec-dc32cb651a0e@example.com>",
    "response": "250 2.0.0 Ok: queued as 9441D8220E",
    "queueId": "183e4b18f0ffe977476",
    "envelope": {
      "from": "sender@example.com",
      "to": ["recipient@destination.com"]
    },
    "networkRouting": null
  }
}
```

### With Network Routing Information

When EmailEngine uses a local address for the SMTP connection:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T08:30:15.123Z",
  "event": "messageSent",
  "data": {
    "messageId": "<abc123@example.com>",
    "response": "250 2.0.0 Ok: queued as XYZ789",
    "queueId": "184a5c29e1aaf988567",
    "envelope": {
      "from": "marketing@company.com",
      "to": ["customer1@email.com", "customer2@email.com"]
    },
    "networkRouting": {
      "localAddress": "192.168.1.100",
      "name": "mail.company.com"
    }
  }
}
```

### Gmail API Submission

Submitted through the Gmail API rather than a gateway, so there is no `response` and no `networkRouting`. `messageId` is the Message-ID header read back from the stored message, so it reflects any rewrite Gmail made, and `originalMessageId` is the value EmailEngine submitted:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "gmail-user",
  "date": "2025-10-17T09:15:42.000Z",
  "event": "messageSent",
  "data": {
    "messageId": "<CABcd123@mail.gmail.com>",
    "originalMessageId": "<local-id-456@example.com>",
    "queueId": "184b6d30f2bbg099678",
    "envelope": {
      "from": "user@gmail.com",
      "to": ["recipient@example.com"]
    }
  }
}
```

On a send-only Gmail account (no read scope) the stored message cannot be read back, so `messageId` equals `originalMessageId`.

### Microsoft Graph API Submission

Submitted through the Graph API rather than a gateway, so there is no `response` and no `networkRouting`. Graph does not report a Message-ID, so `messageId` and `originalMessageId` both carry the value EmailEngine submitted:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "outlook-user",
  "date": "2025-10-17T10:22:33.456Z",
  "event": "messageSent",
  "data": {
    "messageId": "<local-id-789@example.com>",
    "originalMessageId": "<local-id-789@example.com>",
    "queueId": "184c7e41g3cch110789",
    "envelope": {
      "from": "user@outlook.com",
      "to": ["recipient@example.com"]
    }
  }
}
```

### With MTA Message-ID Override

Some SMTP servers assign their own Message-ID and report it in the response. EmailEngine recognizes Outlook.com and Amazon SES responses:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T11:45:00.000Z",
  "event": "messageSent",
  "data": {
    "messageId": "<0102018b9c8d7e6f-a1b2c3d4-5678-90ab-cdef-123456789012-000000@eu-west-1.amazonses.com>",
    "originalMessageId": "<local-message-id@example.com>",
    "response": "250 Ok 0102018b9c8d7e6f-a1b2c3d4-5678-90ab-cdef-123456789012-000000",
    "queueId": "184d8f52h4ddi221890",
    "envelope": {
      "from": "sender@example.com",
      "to": ["recipient@destination.com"]
    },
    "networkRouting": null
  }
}
```

## Field Details

### messageId vs originalMessageId

- **`messageId`**: the Message-ID to use for tracking and correlation. This is the value the provider knows the message by
- **`originalMessageId`**: the Message-ID EmailEngine generated when the message was queued, which is also the `messageId` the Submit API returned

For SMTP submissions `originalMessageId` is present only when the server rewrote the Message-ID:
- **Amazon SES**: the `250 Ok <id>` response is turned into `<id@region.amazonses.com>`, with the region read from the SMTP hostname. The `us-east-1` region is written as `email.amazonses.com`
- **Outlook.com SMTP**: the `250 2.0.0 OK <...@...prod.outlook.com>` response carries the new Message-ID

Other servers that rewrite the Message-ID are not detected, and `messageId` keeps the submitted value.

### response Field

The `response` field contains the SMTP server's final response line. Common formats:

- **Postfix**: `250 2.0.0 Ok: queued as 9441D8220E`
- **Amazon SES**: `250 Ok 0102018b9c8d7e6f-...`
- **Gmail SMTP**: `250 2.0.0 OK 1234567890 abc123.456 - gsmtp`

This field is not present for a submission handed to the Gmail API or the Graph API. A submission from such an account that is routed through an SMTP gateway does carry it.

### queueId Field

The `queueId` is EmailEngine's identifier for the submission job. Use it to:

- Correlate with the original [Submit API](/docs/api/post-v-1-account-account-submit) response
- Look the job up through the [Outbox API](/docs/api/get-v-1-outbox-queueid) while it is still queued
- Match the [`messageDeliveryError`](/docs/webhooks/messagedeliveryerror) and [`messageFailed`](/docs/webhooks/messagefailed) events for the same message

## Handling the Event

### Basic Handler

```javascript
async function handleMessageSent(event) {
  const { account, data } = event;

  console.log(`Email sent successfully for account ${account}`);
  console.log(`  Queue ID: ${data.queueId}`);
  console.log(`  Message ID: ${data.messageId}`);
  console.log(`  Recipients: ${data.envelope.to.join(', ')}`);

  if (data.response) {
    console.log(`  Server Response: ${data.response}`);
  }

  await db.emailLogs.update({
    queueId: data.queueId,
    status: 'sent',
    messageId: data.messageId,
    sentAt: event.date
  });
}
```

### Tracking with Original Message-ID

```javascript
async function handleMessageSent(event) {
  const { data } = event;

  const trackingId = data.originalMessageId || data.messageId;

  await db.emails.update(
    { messageId: trackingId },
    {
      finalMessageId: data.messageId,
      status: 'delivered_to_mta',
      mtaResponse: data.response
    }
  );
}
```

### With Error Handling

```javascript
async function handleMessageSent(event) {
  try {
    const { account, data, date } = event;

    await auditLog.create({
      type: 'email_sent',
      account,
      queueId: data.queueId,
      messageId: data.messageId,
      recipients: data.envelope.to,
      timestamp: new Date(date)
    });

    await notifyWebhookSubscribers('email.sent', {
      messageId: data.messageId,
      recipients: data.envelope.to
    });

  } catch (error) {
    console.error('Failed to process messageSent webhook:', error);
    throw error;
  }
}
```

## Relationship to Other Events

The `messageSent` event is part of the email delivery lifecycle:

1. **Submit API call** - Email is queued for sending
2. **messageSent** - Email accepted by the provider (this event)
3. **messageDeliveryError** - A delivery attempt failed and may be retried
4. **messageFailed** - EmailEngine has given up on the message
5. **messageBounce** - A bounce notification arrived in a monitored mailbox

A successful `messageSent` event means the provider accepted the message, but delivery issues may still occur:

- The recipient server may later bounce the message
- The message may be filtered as spam
- The recipient mailbox may be full

Monitor `messageBounce`, `messageDeliveryError`, and `messageFailed` events for complete delivery tracking.

## Version History

- **v2.52.4**: `envelope` is included for Gmail API and MS Graph submissions
- **v2.40.2**: `networkRouting` added

## Best Practices

1. **Store the queue ID** - Save `queueId` when calling the Submit API to correlate with this webhook
2. **Handle Message-ID changes** - Check for `originalMessageId` to maintain tracking when providers rewrite IDs
3. **Don't assume delivery** - This event confirms acceptance by the provider, not inbox delivery
4. **Process quickly** - Return 2xx before the 30 second delivery timeout, then do the work asynchronously
5. **Use for audit trails** - Log all sent emails for compliance and debugging
6. **Correlate with bounces** - Match `messageId` with bounce notifications for delivery verification

## Related Events

- [messageDeliveryError](/docs/webhooks/messagedeliveryerror) - A delivery attempt failed and may be retried
- [messageFailed](/docs/webhooks/messagefailed) - EmailEngine has given up on the message
- [messageBounce](/docs/webhooks/messagebounce) - Bounce message received

## See Also

- [Webhooks Overview](/docs/webhooks/overview) - Delivery, retries, headers and signing
- [Submit API](/docs/api/post-v-1-account-account-submit) - Send emails via EmailEngine
- [Outbox queue](/docs/sending/outbox-queue) - What happens to a message between submission and this event
- [Outbox API](/docs/api/get-v-1-outbox) - Check queued message status
- [Local addresses](/docs/advanced/local-addresses) - Where `networkRouting` comes from
