---
title: "messageDeliveryError"
sidebar_position: 8
description: "Webhook event triggered when EmailEngine fails to deliver an email to the SMTP server (may be retried)"
---

# messageDeliveryError

The `messageDeliveryError` webhook event is triggered when an SMTP delivery attempt for a queued email fails. It is sent for every failed attempt, including the last one. Whether the attempt is retried depends on the error; when EmailEngine gives up, a [`messageFailed`](/docs/webhooks/messagefailed) event follows.

## When This Event is Triggered

The `messageDeliveryError` event fires when an attempt to hand a queued message to an SMTP server fails, for example because:

- The SMTP server rejects the sender, a recipient or the message content
- The connection times out, or the TCP connection cannot be established
- TLS negotiation or certificate validation fails
- DNS resolution fails for the SMTP hostname
- Authentication with the SMTP server fails, or an OAuth2 access token cannot be obtained

This event covers SMTP submissions only, including messages routed through an [SMTP gateway](/docs/sending/transactional-service). A Gmail API or MS Graph account that submits through a gateway takes the SMTP path too, so it produces this event.

A submission handed to the Gmail API or the Microsoft Graph API does not produce it. Such a failure is retried by the queue in the same way, but only the final outcome is reported, as [`messageFailed`](/docs/webhooks/messagefailed).

## Common Use Cases

- **Delivery monitoring** - Track SMTP failures in real-time to identify connectivity issues
- **Alerting** - Trigger alerts when delivery errors exceed a threshold
- **Retry tracking** - Monitor how many attempts have been made for problematic messages
- **Diagnostics** - Log detailed error information for troubleshooting SMTP configuration
- **Failover logic** - Switch to backup SMTP servers when primary server errors occur
- **Rate limiting detection** - Identify when SMTP servers are rejecting messages due to sending limits

## Payload Schema

### Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `serviceUrl` | string or null | Yes | The configured EmailEngine service URL, `null` if not set |
| `account` | string | Yes | Account ID that attempted to send the message |
| `date` | string | Yes | ISO 8601 timestamp when the webhook was generated |
| `event` | string | Yes | Always `messageDeliveryError` |
| `data` | object | Yes | Event data object (see below) |

The event carries no `path` or `specialUse`. The unique event identifier is sent as the HTTP header `X-EE-Wh-Event-Id`, not in the JSON payload.

### Data Object Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `queueId` | string | Yes | EmailEngine's queue ID for this submission, the same value the Submit API returned |
| `envelope` | object | Yes | SMTP envelope with sender and recipients |
| `messageId` | string | Yes | Message-ID header of the email being sent |
| `error` | string | Yes | Error message from the SMTP client |
| `errorCode` | string | No | Error code (see [Error Codes Reference](#error-codes-reference)) |
| `smtpResponse` | string | No | The server's response line, when the server answered |
| `smtpResponseCode` | number | No | SMTP reply code from that response (for example `421`, `450`, `550`) |
| `smtpCommand` | string | No | SMTP command the server rejected (for example `RCPT TO`, `DATA`, `AUTH PLAIN`) |
| `networkRouting` | object or null | Yes | Local address and proxy used for the SMTP connection, `null` when neither was configured (see below) |
| `job` | object | Yes | Queue job information including retry details (see below) |

Fields whose value is not known (`errorCode`, `smtpResponse`, `smtpResponseCode`, `smtpCommand`) are omitted from the JSON.

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

### Job Object Structure

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Queue job ID. The submit job is created with the queue ID as its job ID, so this is the same value as `queueId` |
| `attemptsMade` | number | Attempts completed before this one. `0` on the first attempt |
| `attempts` | number | Maximum number of attempts for this message: the `deliveryAttempts` of the submission, or the `deliveryAttempts` setting, or `10` |
| `nextAttempt` | string or boolean | ISO 8601 time of the next attempt if this one is retried, or `false` when this was the last attempt |

`nextAttempt` is computed before the attempt is made, from the attempt count alone. A permanent failure (see [Retry Behavior](#retry-behavior)) ends the job even when `nextAttempt` carries a time; in that case `messageFailed` follows immediately.

## Example Payload

### Connection Timeout

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T08:15:32.456Z",
  "event": "messageDeliveryError",
  "data": {
    "queueId": "183e4b18f0ffe977476",
    "envelope": {
      "from": "sender@example.com",
      "to": ["recipient@destination.com"]
    },
    "messageId": "<305eabf4-9538-2747-acec-dc32cb651a0e@example.com>",
    "error": "Connection timeout",
    "errorCode": "ETIMEDOUT",
    "smtpCommand": "CONN",
    "networkRouting": null,
    "job": {
      "id": "183e4b18f0ffe977476",
      "attemptsMade": 0,
      "attempts": 10,
      "nextAttempt": "2025-10-17T08:15:37.456Z"
    }
  }
}
```

### Authentication Failure

A `535` reply is permanent, so the message is not retried even though `nextAttempt` carries a time:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T09:30:15.123Z",
  "event": "messageDeliveryError",
  "data": {
    "queueId": "184a5c29e1aaf988567",
    "envelope": {
      "from": "sender@example.com",
      "to": ["recipient@destination.com"]
    },
    "messageId": "<abc123@example.com>",
    "error": "Invalid login: 535 5.7.8 Authentication credentials invalid",
    "errorCode": "EAUTH",
    "smtpResponse": "535 5.7.8 Authentication credentials invalid",
    "smtpResponseCode": 535,
    "smtpCommand": "AUTH PLAIN",
    "networkRouting": null,
    "job": {
      "id": "184a5c29e1aaf988567",
      "attemptsMade": 0,
      "attempts": 10,
      "nextAttempt": "2025-10-17T09:30:20.123Z"
    }
  }
}
```

### TLS Certificate Error

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T10:45:00.000Z",
  "event": "messageDeliveryError",
  "data": {
    "queueId": "184b6d30f2bbg099678",
    "envelope": {
      "from": "sender@company.com",
      "to": ["customer@external.com"]
    },
    "messageId": "<msg-456@company.com>",
    "error": "certificate has expired",
    "errorCode": "ESOCKET",
    "smtpCommand": "CONN",
    "networkRouting": null,
    "job": {
      "id": "184b6d30f2bbg099678",
      "attemptsMade": 1,
      "attempts": 10,
      "nextAttempt": "2025-10-17T10:45:10.000Z"
    }
  }
}
```

### Temporary Recipient Rejection

A `450` reply is transient, so the message is retried:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T12:00:00.000Z",
  "event": "messageDeliveryError",
  "data": {
    "queueId": "184d8f52h4ddi221890",
    "envelope": {
      "from": "sender@example.com",
      "to": ["greylisted@strict-server.com"]
    },
    "messageId": "<rejected-msg@example.com>",
    "error": "Can't send mail - all recipients were rejected: 450 4.7.1 <greylisted@strict-server.com>: Recipient address rejected: Greylisted, see http://postgrey.schweikert.ch/help/strict-server.com.html",
    "errorCode": "EENVELOPE",
    "smtpResponse": "450 4.7.1 <greylisted@strict-server.com>: Recipient address rejected: Greylisted, see http://postgrey.schweikert.ch/help/strict-server.com.html",
    "smtpResponseCode": 450,
    "smtpCommand": "RCPT TO",
    "networkRouting": null,
    "job": {
      "id": "184d8f52h4ddi221890",
      "attemptsMade": 2,
      "attempts": 10,
      "nextAttempt": "2025-10-17T12:00:20.000Z"
    }
  }
}
```

### With Network Routing Information

When EmailEngine uses a local address and a proxy for the SMTP connection:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T13:15:45.678Z",
  "event": "messageDeliveryError",
  "data": {
    "queueId": "184e9g63i5eej332901",
    "envelope": {
      "from": "marketing@company.com",
      "to": ["customer@email.com"]
    },
    "messageId": "<campaign-123@company.com>",
    "error": "connect ECONNREFUSED 203.0.113.25:587",
    "errorCode": "ESOCKET",
    "smtpCommand": "CONN",
    "networkRouting": {
      "localAddress": "192.168.1.100",
      "proxy": "socks5://proxy.company.com:1080",
      "name": "mail.company.com"
    },
    "job": {
      "id": "184e9g63i5eej332901",
      "attemptsMade": 9,
      "attempts": 10,
      "nextAttempt": false
    }
  }
}
```

## Error Codes Reference

`errorCode` is the code set by the SMTP client library:

| Error Code | Description |
|------------|-------------|
| `ETIMEDOUT` | Connection or command timed out |
| `ECONNECTION` | Failed to establish a TCP connection to the SMTP server |
| `ESOCKET` | Socket-level error, including refused connections and TLS certificate failures |
| `EDNS` | DNS resolution failed for the SMTP hostname |
| `ETLS` | TLS negotiation failed |
| `EPROTOCOL` | Unexpected response from the SMTP server |
| `EAUTH` | SMTP authentication failed |
| `ENOAUTH` | The server requires authentication and no credentials were configured |
| `EOAUTH2` | An OAuth2 access token could not be obtained or was rejected |
| `EENVELOPE` | The server rejected the sender or the recipients |
| `EMESSAGE` | The server rejected the message content |
| `ESTREAM` | The message stream failed during transmission |

## Handling the Event

### Basic Handler

```javascript
async function handleMessageDeliveryError(event) {
  const { account, data } = event;

  console.log(`Delivery error for account ${account}`);
  console.log(`  Queue ID: ${data.queueId}`);
  console.log(`  Error: ${data.error}`);
  console.log(`  Error Code: ${data.errorCode}`);
  console.log(`  Attempt: ${data.job.attemptsMade + 1}/${data.job.attempts}`);

  if (data.job.nextAttempt) {
    console.log(`  Next retry: ${data.job.nextAttempt}`);
  } else {
    console.log(`  No more retries scheduled`);
  }

  await monitoring.logDeliveryError({
    account,
    queueId: data.queueId,
    error: data.error,
    errorCode: data.errorCode,
    attempt: data.job.attemptsMade + 1,
    timestamp: event.date
  });
}
```

### Alert on Repeated Failures

```javascript
async function handleMessageDeliveryError(event) {
  const { account, data } = event;

  if (data.job.attemptsMade >= 5) {
    await alerting.send({
      severity: 'warning',
      title: 'Email delivery struggling',
      message: `Message ${data.queueId} has failed ${data.job.attemptsMade + 1} times`,
      details: {
        account,
        error: data.error,
        errorCode: data.errorCode,
        recipients: data.envelope.to
      }
    });
  }

  await analytics.trackError({
    type: 'smtp_delivery_error',
    account,
    errorCode: data.errorCode,
    smtpCode: data.smtpResponseCode
  });
}
```

### Detect Authentication Issues

```javascript
async function handleMessageDeliveryError(event) {
  const { account, data } = event;

  if (data.errorCode === 'EAUTH') {
    await notifyAdmin({
      title: 'SMTP Authentication Failed',
      message: `Account ${account} failed to authenticate with SMTP server`,
      action: 'Check SMTP credentials in account configuration'
    });
  }
}
```

## Retry Behavior

Whether a failed attempt is retried depends on the error:

- A server reply of `5xx` is permanent, except `503`. The job ends and `messageFailed` follows
- A server reply of `4xx` (or `503`) is transient and the attempt is retried
- Without a server reply, `EAUTH`, `ENOAUTH`, `EOAUTH2`, `ETLS`, `EENVELOPE`, `EMESSAGE` and `EPROTOCOL` are permanent; every other error is retried

Retries follow an exponential backoff: 5 seconds after the first failure, doubling each time, with a random reduction of up to 20 percent. When the attempts are used up, `messageFailed` is sent. The full schedule, the `deliveryAttempts` setting and what the queue keeps afterwards are described on the [Outbox queue](/docs/sending/outbox-queue#delivery-attempts) page.

## Relationship to Other Events

The `messageDeliveryError` event is part of the email delivery lifecycle:

1. **Submit API call** - Email is queued for sending
2. **messageDeliveryError** - A delivery attempt failed (this event, may occur multiple times)
3. **messageSent** - Email accepted by the SMTP server (success path)
4. **messageFailed** - EmailEngine has given up (failure path)

```text
Submit API
    |
    v
+------------------+
| Delivery attempt |
+------------------+
    |               |
    | success       | error
    v               v
messageSent    messageDeliveryError
                    |
                    | retry?
              +-----+-----+
              | yes       | no
              v           v
         next attempt  messageFailed
```

## Best Practices

1. **Monitor error patterns** - Track error codes to identify systemic issues (DNS, TLS, auth)
2. **Set up alerts** - Notify operators when error rates spike or specific errors occur
3. **Log for debugging** - Store full webhook payloads to diagnose delivery problems
4. **Handle auth errors specially** - Authentication failures are permanent and usually mean a configuration problem
5. **Track retry counts** - Know which messages are struggling to deliver
6. **Process quickly** - Return 2xx before the 30 second delivery timeout, then do the work asynchronously

## Related Events

- [messageSent](/docs/webhooks/messagesent) - Successful delivery to the SMTP server
- [messageFailed](/docs/webhooks/messagefailed) - EmailEngine has given up on the message
- [messageBounce](/docs/webhooks/messagebounce) - Bounce message received

## See Also

- [Webhooks Overview](/docs/webhooks/overview) - Delivery, retries, headers and signing
- [Outbox queue](/docs/sending/outbox-queue) - Retry schedule, permanent versus transient failures, and what the queue keeps
- [Submit API](/docs/api/post-v-1-account-account-submit) - Send emails via EmailEngine
- [Outbox API](/docs/api/get-v-1-outbox-queueid) - Check the state of a queued message by `queueId`
- [Local addresses](/docs/advanced/local-addresses) - Where `networkRouting` comes from
