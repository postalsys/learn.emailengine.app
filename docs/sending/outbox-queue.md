---
title: Outbox Queue
sidebar_position: 7
description: Understanding EmailEngine's message queue system for reliable email delivery
---

# Outbox Queue

A submitted message is not sent inside the API call. EmailEngine stores it, answers with a queue ID, and a separate worker delivers it. This page covers what happens between "queued" and "sent": the states a message passes through, how retries are timed, what the outbox API shows, and which settings control it.

## Why Queues Matter

When you submit an email, EmailEngine:

1. Validates your request
2. Stores the message and adds a job to the submit queue
3. Returns immediately with a queue ID
4. Delivers the message from a worker, independently of the API request
5. Retries a failed delivery on a schedule
6. Reports the outcome through webhooks

Because the job lives in Redis rather than in the API process, a scheduled or retrying message survives an EmailEngine restart.

## Queue Technology

EmailEngine uses [BullMQ](https://docs.bullmq.io/) for its queues, backed by Redis. Outbound mail is the `submit` queue; webhooks, exports, and the deprecated Document Store have queues of their own. The [queue management](/docs/advanced/queue-management) page describes the queues as a set, along with Bull Board and worker concurrency. This page stays with the submit queue.

## Job Lifecycle

A job in the submit queue moves through these states:

### 1. Waiting

**Description**: Jobs ready to be processed.

**How jobs get here**:
- New submissions without a `sendAt` in the future
- Delayed jobs whose time has come

**What happens**: A submit worker picks the job up and moves it to *Active*.

### 2. Active

**Description**: Jobs currently being processed.

**What happens**:
- EmailEngine connects to the account's SMTP server (or the gateway named in the submission)
- Transmits the message
- Waits for the SMTP response

**Outcomes**:
- **Success** - Moved to *Completed*
- **Temporary failure** - Moved to *Delayed* and retried
- **Permanent failure** - Moved to *Failed*, even if attempts remain
- **Retries exhausted** - Moved to *Failed* after the last attempt

### 3. Completed

**Description**: The SMTP server accepted the message.

**What happens**:
- `messageSent` webhook is emitted
- The stored message content is removed from Redis

By default a completed job is removed from the queue as soon as it finishes. To keep completed jobs for debugging, set the **Job History Limit** described under [Configuration](#keep-completed-and-failed-jobs). Even then only the job metadata remains; the message content is gone.

### 4. Failed

**Description**: Jobs that will not be retried.

**How jobs get here**:
- Every attempt allowed by `deliveryAttempts` (default 10) failed with a retriable error
- A permanent error occurred, on any attempt

**What happens**:
- `messageFailed` webhook is emitted
- The stored message content is removed from Redis

Failed jobs are kept, unlike completed ones: a failure is the only record that a delivery was given up on. By default the last 500 failed entries are retained for 7 days; `EENGINE_QUEUE_KEEP_FAILED` and `EENGINE_QUEUE_KEEP_FAILED_AGE` (seconds) change those bounds. The retained entry holds the job metadata and the last error, not the message itself.

**How EmailEngine decides that an error is permanent:**

1. When the SMTP server replied with a status code, that code decides: a 5xx reply is permanent, except 503, which is retried. A 4xx reply is retried
2. When there was no SMTP reply (the connection or the handshake failed), the error code decides. These codes are permanent:

| Error Code | Meaning |
|---|---|
| `EAUTH` | SMTP authentication failed |
| `ENOAUTH` | No credentials provided |
| `EOAUTH2` | OAuth2 token failure |
| `ETLS` | TLS handshake failed |
| `EENVELOPE` | Invalid sender or recipients |
| `EMESSAGE` | Message content error |
| `EPROTOCOL` | SMTP protocol mismatch |

Everything else (network timeouts, connection resets, a server that closed the connection) is retried.

### 5. Delayed

**Description**: Jobs waiting for a future time.

**How jobs get here**:
- New submissions with a `sendAt` in the future
- Failed delivery attempts that will be retried

**What happens**:
- The job waits until its time
- Then it moves to *Waiting*
- A `messageDeliveryError` webhook is emitted for every failed delivery attempt, including the final one

**Retry schedule**: BullMQ's exponential backoff with a 5 second base. The delay before retry *n* is 2^(n-1) x 5 s, reduced by a random amount of up to 20% (the jitter), so each retry lands between 80% and 100% of the nominal delay:

| Attempt | Nominal delay after the previous failure |
|---|---|
| 1 | Immediate |
| 2 | 5 s |
| 3 | 10 s |
| 4 | 20 s |
| 5 | 40 s |
| 6 | 80 s |
| 10 | about 21 minutes |

With the default of 10 attempts, a message that keeps failing with a retriable error is given up on roughly 43 minutes after the first attempt. The jitter keeps a batch of messages that failed together from retrying at the same instant.

:::info nextAttempt is the nominal time
The `nextAttempt` value in the outbox API and in the `messageDeliveryError` payload is computed from the nominal delay without the jitter, so it is the latest moment the retry can happen; the actual retry is up to 20% earlier. Before v2.70.0 the outbox API doubled the delay once too often and reported a time later than the nominal one.
:::

### Pausing the queue

BullMQ 6 (EmailEngine v2.79.0 and later) has no separate "paused" job state. Pausing the queue stops workers from taking new jobs; the jobs that are already waiting stay in *Waiting*, an active job finishes, and new submissions join *Waiting* too. Resuming lets the workers continue.

Pause and resume from Bull Board, or through the [queue settings API](/docs/api/put-v-1-settings-queue-queue):

```bash
# Pause queue
curl -XPUT "https://emailengine.example.com/v1/settings/queue/submit" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"paused": true}'

# Resume queue
curl -XPUT "https://emailengine.example.com/v1/settings/queue/submit" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"paused": false}'
```

Use it for maintenance windows, for debugging, or to hold outgoing mail while a provider rate limit clears.

## Monitoring the Queue

### Bull Board UI

EmailEngine includes [Bull Board](https://github.com/felixmosh/bull-board), a web UI for BullMQ queues.

**Access**: **System > Queues** in the admin menu, or `/admin/bull-board` directly.

**Features**:
- View job counts by state
- Inspect individual jobs
- Retry failed jobs
- Delete jobs
- Pause and resume queues
- View job logs

### Outbox API

The [outbox API](/docs/api/get-v-1-outbox) lists the jobs in the delayed, waiting, active, and failed states, in that order. Completed jobs never appear here, even when the Job History Limit retains them; they are visible in Bull Board only.

#### List queued messages

```bash
curl "https://emailengine.example.com/v1/outbox" \
  -H "Authorization: Bearer <token>"
```

**Response**:

```json
{
  "total": 120,
  "page": 0,
  "pages": 6,
  "messages": [
    {
      "queueId": "4646ac53857fd2b2",
      "account": "example",
      "source": "api",
      "messageId": "<test123@example.com>",
      "envelope": {
        "from": "sender@example.com",
        "to": ["recipient@example.com"]
      },
      "subject": "Test message",
      "gateway": null,
      "proxy": null,
      "localAddress": null,
      "created": "2025-05-14T10:00:00.000Z",
      "scheduled": "2025-05-14T10:00:00.000Z",
      "nextAttempt": "2025-05-14T10:00:15.465Z",
      "attemptsMade": 2,
      "attempts": 10,
      "progress": {
        "status": "error",
        "error": {
          "message": "Connection timeout",
          "code": "ETIMEDOUT",
          "statusCode": null
        }
      }
    }
  ]
}
```

Use the `page` and `pageSize` query parameters for pagination (`pageSize` defaults to 20 and goes up to 1000):

```bash
curl "https://emailengine.example.com/v1/outbox?page=0&pageSize=10" \
  -H "Authorization: Bearer <token>"
```

`source` says how the message entered the queue: `api`, `smtp` (the [SMTP server](./smtp-interface.md)), `ui` (a test message from the admin interface), or `test` (a [delivery test](/docs/advanced/inbox-placement-testing)).

The `progress` field tracks the delivery status of each message:

| Status | Meaning |
|---|---|
| `queued` | Accepted and waiting for a delivery attempt |
| `processing` | A submit worker has picked the message up |
| `smtp-starting` | Opening the SMTP connection |
| `smtp-completed` | The SMTP transaction finished and the message was handed over (includes `response` and `messageId`, plus `originalMessageId` when the server assigned a new Message-ID) |
| `submitted` | Delivered to the receiving server (includes the SMTP `response`) |
| `error` | The last attempt failed (includes `error` with `message`, `code`, and the SMTP `statusCode`) |

`nextAttempt` is when the next delivery attempt is due, or `false` when no attempts remain. `attemptsMade` against `attempts` shows how much of the retry budget is left.

#### Get a specific message

Retrieve a single queued message by its queue ID with the [get outbox entry API](/docs/api/get-v-1-outbox-queueid):

```bash
curl "https://emailengine.example.com/v1/outbox/4646ac53857fd2b2" \
  -H "Authorization: Bearer <token>"
```

The response is one entry in the same shape as the list. The endpoint works while the message content is still stored, which means the waiting, active, and delayed states. Once a message has completed or failed for good, its content is removed and the endpoint returns 404, even when the Job History Limit keeps the job's metadata in Bull Board.

## Managing Queue Jobs

### Delete a Job

Use the [delete outbox entry API](/docs/api/delete-v-1-outbox-queueid):

```bash
curl -XDELETE "https://emailengine.example.com/v1/outbox/4646ac53857fd2b2" \
  -H "Authorization: Bearer <token>"
```

**Response**:

```json
{
  "deleted": true
}
```

This removes both the stored message and the job. It works for waiting, delayed, and failed jobs; a job that a worker holds at that moment (active) cannot be removed and the response carries `"deleted": false`, as does an unknown queue ID.

Useful for:
- Cancelling a scheduled send
- Clearing a retained failed job

:::info Retrying Failed Jobs
The outbox API has no retry call. To send a failed message again, resubmit it through the submit API, or use the **Retry** button in Bull Board, which puts the same job back into *Waiting*.
:::

## Configuration

### Delivery Attempts

The maximum number of delivery attempts is set per instance, and can be overridden per message.

**Instance default**:
1. Open **Configuration > Email Processing**
2. Set **Retry Attempts** in the **Email Delivery** card (default: 10)

The same value is the `deliveryAttempts` key in the [Settings API](/docs/api/post-v-1-settings).

**Per message**, in the submit payload:

```json
{
  "to": [{"address": "recipient@example.com"}],
  "subject": "Test",
  "text": "Hello",
  "deliveryAttempts": 5
}
```

Permanent errors (see [Failed](#4-failed)) end the job immediately regardless of this setting.

### Keep Completed and Failed Jobs

Completed jobs are removed as soon as they finish, to save Redis memory. To keep them:

1. Open **Configuration > General**
2. Set **Job History Limit** (`queueKeep`) to the number of completed jobs to keep. 0 disables the history

Retained completed jobs are also dropped after 24 hours, whichever limit is reached first. Failed jobs have their own floor, described under [Failed](#4-failed): at least 500 entries for 7 days, or the Job History Limit when that is higher.

:::warning
The retention policy is attached to a job when it is created, so a change applies to jobs queued after the change. Retained jobs are visible in Bull Board, and retained failed jobs also in `GET /v1/outbox`; `GET /v1/outbox/{queueId}` returns 404 for a finished job either way, because the message content is removed on completion or failure.
:::

### SMTP Timeout

The SMTP socket timeout during delivery is 2 minutes. A server that stays silent longer than that fails the attempt with a retriable error.

## Webhook Events

The queue emits three events. Each has a page of its own with the full payload; the examples here show the fields that relate to the queue.

### messageSent

Emitted when a job completes:

```json
{
  "event": "messageSent",
  "account": "example",
  "date": "2025-05-14T10:32:39.499Z",
  "data": {
    "messageId": "<message-id@example.com>",
    "queueId": "4646ac53857fd2b2",
    "response": "250 2.0.0 Ok: queued as 5755482356",
    "envelope": {
      "from": "sender@example.com",
      "to": ["recipient@example.com"]
    }
  }
}
```

### messageDeliveryError

Emitted on every failed delivery attempt, including the final one, where `messageFailed` follows. `job.nextAttempt` is the nominal time of the next retry, or `false` after the last attempt:

```json
{
  "event": "messageDeliveryError",
  "account": "example",
  "date": "2025-05-14T10:05:35.832Z",
  "data": {
    "queueId": "4646ac53857fd2b2",
    "messageId": "<message-id@example.com>",
    "error": "Connection timeout",
    "errorCode": "ETIMEDOUT",
    "smtpResponseCode": null,
    "job": {
      "attemptsMade": 2,
      "attempts": 10,
      "nextAttempt": "2025-05-14T10:05:55.465Z"
    },
    "envelope": {
      "from": "sender@example.com",
      "to": ["recipient@example.com"]
    }
  }
}
```

### messageFailed

Emitted when a job is finished for good, after the last attempt or on a permanent error:

```json
{
  "event": "messageFailed",
  "account": "example",
  "date": "2025-05-14T11:58:50.181Z",
  "data": {
    "messageId": "<message-id@example.com>",
    "queueId": "4646ac53857fd2b2",
    "error": "Error: Invalid login: 535 5.7.8 Error: authentication failed"
  }
}
```

## See Also

- [Queue Management](/docs/advanced/queue-management) - Every queue, Bull Board, worker concurrency, and Redis tuning
- [Basic Sending](/docs/sending/basic-sending) - How a message enters the queue
- [messageDeliveryError](/docs/webhooks/messagedeliveryerror) and [messageFailed](/docs/webhooks/messagefailed) - The full payloads of the failure events
- [Sending API](/docs/api-reference/sending-api) - The outbox endpoints in the API reference
- [Bounces](/docs/advanced/bounces) - A failure that arrives by mail after the queue reported success
