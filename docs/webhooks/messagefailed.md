---
title: "messageFailed"
sidebar_position: 9
description: "Webhook event triggered when EmailEngine permanently fails to deliver an email after all retry attempts are exhausted"
---

# messageFailed

The `messageFailed` webhook event is triggered when EmailEngine gives up on a queued email. It is a terminal event: no further delivery attempts are made for this message.

## When This Event is Triggered

The `messageFailed` event fires when the delivery job for a queued message ends without success, because:

- All delivery attempts were used up: the `deliveryAttempts` of the submission, or the `deliveryAttempts` setting, or 10 by default
- An attempt failed permanently, so the remaining attempts were not spent. For SMTP submissions that is a `5xx` server reply other than `503`, or, without a server reply, an `EAUTH`, `ENOAUTH`, `EOAUTH2`, `ETLS`, `EENVELOPE`, `EMESSAGE` or `EPROTOCOL` error

The event is sent for every submission type: SMTP, Gmail API and Microsoft Graph API. For SMTP submissions each failed attempt has already produced a [`messageDeliveryError`](/docs/webhooks/messagedeliveryerror) event with the details of the error; for API submissions this event is the only report of the failure.

## Common Use Cases

- **Sender notification** - Notify the original sender that their email could not be delivered
- **Queue cleanup** - Remove the message from any pending send tracking
- **Audit logging** - Maintain a record of all permanently failed deliveries
- **Analytics** - Track failure rates and identify problematic patterns
- **Retry via alternative method** - Attempt delivery through a backup SMTP server or different channel
- **CRM updates** - Mark contacts as unreachable or flag delivery issues
- **Alerting** - Trigger alerts for high failure rates or critical email failures

## Payload Schema

### Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `serviceUrl` | string or null | Yes | The configured EmailEngine service URL, `null` if not set |
| `account` | string | Yes | Account ID that attempted to send the message |
| `date` | string | Yes | ISO 8601 timestamp when the webhook was generated |
| `event` | string | Yes | Always `messageFailed` |
| `data` | object | Yes | Event data object (see below) |

The event carries no `path` or `specialUse`. The unique event identifier is sent as the HTTP header `X-EE-Wh-Event-Id`, not in the JSON payload.

### Data Object Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `messageId` | string | Yes | Message-ID header of the email that failed to send |
| `queueId` | string | Yes | EmailEngine's queue ID for this submission, the same value the Submit API returned |
| `error` | string | Yes | First line of the stack trace recorded for the message's *first* failed attempt, so it starts with the error class name, as in `Error: <message>`. When a message fails the same way every time, which is the common case, that is also the final error |
| `networkRouting` | object or null | No | Local address and proxy used for the last SMTP attempt, `null` when neither was configured. Present for SMTP submissions, absent for submissions handed to the Gmail API or the Microsoft Graph API (see below) |

There is no `envelope` on this event. Correlate by `queueId` or `messageId` with the `messageSent` or `messageDeliveryError` events, or with your own record of the submission.

### Network Routing Object Structure

| Field | Type | Description |
|-------|------|-------------|
| `localAddress` | string | Local IP address used for the SMTP connection |
| `proxy` | string | Proxy URL used for the connection |
| `name` | string | Hostname used in the `EHLO` greeting |
| `requestedLocalAddress` | string | The `localAddress` requested in the submission, when a different one was used |

## Example Payload

### Authentication Failure

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T08:15:32.456Z",
  "event": "messageFailed",
  "data": {
    "messageId": "<305eabf4-9538-2747-acec-dc32cb651a0e@example.com>",
    "queueId": "183e4b18f0ffe977476",
    "error": "Error: Invalid login: 535 5.7.8 Error: authentication failed",
    "networkRouting": null
  }
}
```

### Connection Timeout

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T09:30:15.123Z",
  "event": "messageFailed",
  "data": {
    "messageId": "<abc123@example.com>",
    "queueId": "184a5c29e1aaf988567",
    "error": "Error: Connection timeout",
    "networkRouting": null
  }
}
```

### Recipient Unknown

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T10:45:00.000Z",
  "event": "messageFailed",
  "data": {
    "messageId": "<msg-456@company.com>",
    "queueId": "184b6d30f2bbg099678",
    "error": "Error: Can't send mail - all recipients were rejected: 550 5.1.1 The email account that you tried to reach does not exist",
    "networkRouting": null
  }
}
```

### With Network Routing Information

Every SMTP submission carries the field. It is `null` unless a [local address](/docs/advanced/local-addresses) or a proxy was used:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T11:22:33.789Z",
  "event": "messageFailed",
  "data": {
    "messageId": "<campaign-789@marketing.com>",
    "queueId": "184c7e41g3cch110789",
    "error": "Error: connect ECONNREFUSED 203.0.113.25:587",
    "networkRouting": {
      "localAddress": "192.168.1.100",
      "name": "mail.company.com"
    }
  }
}
```

## Field Details

### messageId

The Message-ID header from the original email. This is an RFC 5322 compliant identifier that can be used to:

- Correlate with the original submission request
- Match with any `messageDeliveryError` events that preceded this failure
- Track the message through your email pipeline

Format: `<unique-identifier@hostname.domain>`

### queueId

EmailEngine's queue identifier for this submission. Use this to:

- Correlate with the original [Submit API](/docs/api/post-v-1-account-account-submit) response
- Match with `messageDeliveryError` events during retry attempts
- Find the job in the **Failed** tab of **Submission Queue** under **System > Queues**

### error

The first line of the stack trace of the first failed attempt, so it starts with the error class name. A message whose attempts failed for different reasons therefore reports the earliest one here; the [`messageDeliveryError`](/docs/webhooks/messagedeliveryerror) events carry each attempt's own error, and the job's full stack traces are in Bull Board.

Common patterns:

| Error Pattern | Description |
|---------------|-------------|
| `Error: Invalid login: 535...` | SMTP authentication failure |
| `Error: Connection timeout` | Unable to connect to the SMTP server within the timeout |
| `Error: Can't send mail - all recipients were rejected: 550 5.1.1 ...` | Recipient mailbox does not exist |
| `Error: Can't send mail - all recipients were rejected: 550 5.7.1 ...` | Sender rejected by the recipient server |
| `Error: connect ECONNREFUSED ...` | SMTP server refused the connection |
| `Error: certificate has expired` | TLS certificate validation failed |

## Handling the Event

### Basic Handler

```javascript
async function handleMessageFailed(event) {
  const { account, data } = event;

  console.log(`Permanent delivery failure for account ${account}`);
  console.log(`  Queue ID: ${data.queueId}`);
  console.log(`  Message ID: ${data.messageId}`);
  console.log(`  Error: ${data.error}`);

  await db.emailLogs.update({
    queueId: data.queueId,
    status: 'failed',
    error: data.error,
    failedAt: event.date
  });
}
```

### Notify the Sender

```javascript
async function handleMessageFailed(event) {
  const { account, data, date } = event;

  const emailRecord = await db.emails.findOne({
    queueId: data.queueId
  });

  if (emailRecord && emailRecord.senderEmail) {
    await notificationService.sendEmail({
      to: emailRecord.senderEmail,
      subject: 'Email delivery failed',
      body: `Your email to ${emailRecord.recipientEmail} could not be delivered.\n\n` +
            `Error: ${data.error}\n\n` +
            `Message ID: ${data.messageId}`
    });
  }

  await auditLog.create({
    type: 'email_delivery_failed',
    account,
    queueId: data.queueId,
    error: data.error,
    timestamp: new Date(date)
  });
}
```

### Track Failure Patterns

```javascript
async function handleMessageFailed(event) {
  const { account, data } = event;

  let errorType = 'unknown';
  if (data.error.includes('authentication')) {
    errorType = 'auth_failure';
  } else if (data.error.includes('timeout')) {
    errorType = 'timeout';
  } else if (data.error.includes('5.1.1')) {
    errorType = 'invalid_recipient';
  } else if (data.error.includes('5.7.')) {
    errorType = 'policy_rejection';
  }

  await analytics.track({
    event: 'email_failed',
    properties: {
      account,
      errorType,
      queueId: data.queueId
    }
  });

  if (errorType === 'auth_failure') {
    await alerting.send({
      severity: 'high',
      title: 'SMTP Authentication Failure',
      message: `Account ${account} has authentication issues`,
      details: { error: data.error }
    });
  }
}
```

### Retry via Alternative Method

```javascript
async function handleMessageFailed(event) {
  const { data } = event;

  const originalMessage = await db.outbox.findOne({
    queueId: data.queueId
  });

  if (originalMessage && originalMessage.retryCount < 1) {
    try {
      await emailService.sendViaBackup({
        to: originalMessage.to,
        subject: originalMessage.subject,
        body: originalMessage.body,
        originalQueueId: data.queueId
      });

      await db.outbox.update({
        queueId: data.queueId,
        retryCount: originalMessage.retryCount + 1,
        lastRetryMethod: 'backup_smtp'
      });
    } catch (backupError) {
      console.error('Backup delivery also failed:', backupError);
    }
  }
}
```

## Retry Behavior

Before `messageFailed` is sent, the queue retries transient failures with an exponential backoff: 5 seconds after the first failure, doubling each time, with a random reduction of up to 20 percent, for up to 10 attempts by default. The schedule, the `deliveryAttempts` setting and the distinction between permanent and transient failures are described on the [Outbox queue](/docs/sending/outbox-queue#delivery-attempts) page.

The failed job stays in the submit queue for inspection: the last 500 failed jobs are kept for 7 days by default, see [Keep Completed and Failed Jobs](/docs/sending/outbox-queue#keep-completed-and-failed-jobs). The message itself is no longer queued and cannot be resumed; submit it again to retry.

## Relationship to Other Events

The `messageFailed` event is the terminal failure state in the email delivery lifecycle:

| Event | Description |
|-------|-------------|
| `messageSent` | Successful handoff to the provider |
| `messageDeliveryError` | A failed SMTP attempt, retried or not |
| `messageFailed` | EmailEngine has given up (this event) |
| `messageBounce` | Bounce notification received after a successful handoff |

## Difference from messageDeliveryError

| Aspect | messageDeliveryError | messageFailed |
|--------|---------------------|---------------|
| When triggered | Each failed SMTP attempt | Once, when the job ends without success |
| Submission types | SMTP only | SMTP, Gmail API, MS Graph |
| Retries remaining | Usually | None |
| Action required | Monitor, may self-resolve | Intervention needed |
| Payload detail | Error code, server response, envelope, retry info | Message and queue IDs, one error line, network routing |

## Best Practices

1. **Always handle this event** - These are permanent failures that require attention
2. **Notify senders** - Let users know their emails failed to deliver
3. **Track error patterns** - Identify systemic issues (bad credentials, blocked IPs)
4. **Clean up references** - Remove failed messages from pending queues
5. **Alert on spikes** - Monitor for unusual increases in failure rates
6. **Log for compliance** - Maintain records of delivery failures for audit purposes
7. **Consider alternatives** - Implement fallback delivery methods for critical emails
8. **Process quickly** - Return 2xx before the 30 second delivery timeout, then do the work asynchronously

## Related Events

- [messageSent](/docs/webhooks/messagesent) - Successful handoff to the provider
- [messageDeliveryError](/docs/webhooks/messagedeliveryerror) - A failed SMTP attempt, with the error details
- [messageBounce](/docs/webhooks/messagebounce) - Bounce message received after delivery

## See Also

- [Webhooks Overview](/docs/webhooks/overview) - Delivery, retries, headers and signing
- [Outbox queue](/docs/sending/outbox-queue) - Retry schedule, permanent versus transient failures, and what the queue keeps
- [Submit API](/docs/api/post-v-1-account-account-submit) - Send emails via EmailEngine
- [Queue management](/docs/advanced/queue-management) - Inspecting failed jobs in Bull Board
- [Sending Emails](/docs/sending) - Email sending guide
