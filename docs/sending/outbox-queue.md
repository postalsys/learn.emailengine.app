---
title: Outbox Queue
sidebar_position: 7
description: Understanding EmailEngine's message queue system for reliable email delivery
---

# Outbox Queue

EmailEngine uses queues to process background tasks including email sending. Understanding the queue system helps you monitor delivery, troubleshoot issues, and optimize performance.

## Why Queues Matter

When you submit an email, EmailEngine doesn't send it immediately. Instead, it:

1. Validates your request
2. Adds the message to a queue
3. Returns immediately with a queue ID
4. Processes the queue asynchronously
5. Handles retries automatically
6. Notifies you via webhooks

This approach provides:
- **Reliability**: Automatic retries on failures
- **Scalability**: Handle high-volume sending
- **Monitoring**: Track delivery status
- **Resilience**: Survive crashes and restarts

## Queue Technology

EmailEngine uses [BullMQ](https://docs.bullmq.io/) for queue management, backed by Redis. BullMQ provides:

- Persistent job storage
- Automatic retry logic
- Priority queues
- Delayed jobs (scheduled sending)
- Job progress tracking

## Queue Types

EmailEngine maintains three queue types:

### 1. Submit Queue

Handles all email sending jobs.

- **Purpose**: Process outbound email
- **Jobs**: Individual send requests
- **Lifecycle**: Waiting -> Active -> Completed/Failed/Delayed

### 2. Notify Queue

Handles all webhook delivery jobs.

- **Purpose**: Send webhook notifications
- **Jobs**: Webhook HTTP requests
- **Retries**: Automatic retry on webhook failures

### 3. Documents Queue

Handles document indexing jobs.

- **Purpose**: Index emails for search functionality
- **Jobs**: Document indexing tasks
- **Used when**: Document Store/Elasticsearch integration is enabled

## Job Lifecycle

Jobs in the submit queue move through different states:

### 1. Waiting

**Description**: Jobs ready to be processed immediately.

**How jobs get here**:
- New submissions without `sendAt` property
- Delayed jobs whose `sendAt` time has been reached
- Jobs moved from *Paused* when queue is unpaused

**What happens**: Jobs are picked up one by one and moved to *Active*.

### 2. Active

**Description**: Jobs currently being processed.

**What happens**:
- EmailEngine connects to SMTP server
- Transmits the message
- Waits for SMTP response

**Outcomes**:
- **Success** - Moved to *Completed*
- **Temporary failure** - Moved to *Delayed* (will retry)
- **Permanent failure** - Moved to *Failed* (no retry, even if attempts remain)
- **Retries exhausted** - Moved to *Failed* after all attempts used

### 3. Completed

**Description**: Successfully delivered jobs.

**What happens**:
- SMTP server accepted the message (250 OK)
- `messageSent` webhook is emitted
- Message content is removed from Redis

By default, completed jobs are immediately removed from the queue. To keep them for debugging, configure the **Job History Limit** setting (see [Configuration](#keep-completedfailed-jobs) below). Even with retention enabled, the message content is no longer stored - only the job metadata remains in BullMQ.

### 4. Failed

**Description**: Jobs that will not be retried.

**How jobs get here**:
- All `deliveryAttempts` exhausted (default: 10) with retriable errors
- A permanent (non-retriable) error occurred, even on the first attempt

**What happens**:
- `messageFailed` webhook is emitted
- Message content is removed from Redis

By default, failed jobs are immediately removed from the queue, just like completed jobs. To keep them for debugging, configure the **Job History Limit** setting. Even with retention enabled, the message content is no longer stored - only the job metadata remains in BullMQ.

**Permanent (non-retriable) errors** cause immediate failure regardless of remaining attempts:

| Error Code | Meaning |
|---|---|
| `EAUTH` | SMTP authentication failed |
| `ENOAUTH` | No credentials provided |
| `EOAUTH2` | OAuth2 token failure |
| `ETLS` | TLS handshake failed |
| `EENVELOPE` | Invalid sender or recipients |
| `EMESSAGE` | Message content error |
| `EPROTOCOL` | SMTP protocol mismatch |

Additionally, any SMTP response with status code 500 or above (except 503, which is treated as transient) is considered permanent.

**Retriable errors** (temporary network issues, server timeouts, 4xx SMTP responses, etc.) move the job to *Delayed* for retry with exponential backoff.

### 5. Delayed

**Description**: Jobs waiting for future processing.

**How jobs get here**:
- New submissions with `sendAt` property (scheduled sending)
- Failed delivery attempts that will be retried (exponential backoff)

**What happens**:
- Job waits until the delay time
- Then moved to *Waiting*
- A `messageDeliveryError` webhook is emitted for every failed delivery attempt, including the final one

**Retry schedule** (exponential backoff with 5-second base delay and 20% jitter, computed as 2^(attemptsMade-1) x 5s):
- Attempt 1: Immediate
- Attempt 2: ~5 seconds (2^0 x 5s)
- Attempt 3: ~10 seconds (2^1 x 5s)
- Attempt 4: ~20 seconds (2^2 x 5s)
- Attempt 5: ~40 seconds (2^3 x 5s)
- Attempt 6: ~80 seconds (2^4 x 5s)
- And so on, doubling each time

The 20% jitter randomizes retry times slightly to prevent multiple failed jobs from retrying at exactly the same moment.

:::info nextAttempt estimate
The `nextAttempt` value reported by the outbox API is computed as 2^attempts x 5s, so it shows a later time than when the actual retry happens.
:::

### 6. Paused

**Description**: Jobs held when queue is paused.

**How to pause**: Use the Bull Board UI or API to pause the queue.

**What happens**:
- New jobs go to *Paused* instead of *Waiting*
- Active jobs finish processing
- When unpaused, jobs move to *Waiting*

**Use cases**:
- Maintenance windows
- Debugging issues
- Rate limit management

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

## Monitoring the Queue

### Bull Board UI

EmailEngine includes [Bull Board](https://github.com/felixmosh/bull-board), a web UI for BullMQ queues.

**Access**: Navigate to **System -> Queues** in EmailEngine UI, or go directly to `/admin/bull-board`.

**Features**:
- View job counts by state
- Inspect individual jobs
- Retry failed jobs
- Delete jobs
- Pause/resume queues
- View job logs

### Outbox API

The outbox API lists messages across all active queue states: waiting, active, delayed, paused, and failed.

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
      "created": "2025-05-14T10:00:00.000Z",
      "scheduled": "2025-05-14T10:00:00.000Z",
      "nextAttempt": "2025-05-14T10:10:35.465Z",
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

Use the `page` and `pageSize` query parameters for pagination:

```bash
curl "https://emailengine.example.com/v1/outbox?page=0&pageSize=10" \
  -H "Authorization: Bearer <token>"
```

The `progress` field tracks the delivery status of each message:

| Status | Meaning |
|---|---|
| `queued` | Waiting to be processed |
| `processing` | Currently being sent |
| `submitted` | Successfully delivered (includes SMTP `response`) |
| `error` | Last attempt failed (includes `error` details) |

The `nextAttempt` field shows when the next delivery attempt is scheduled. It is `false` when no more attempts remain.

:::info Completed and failed jobs
By default, completed and failed jobs are removed from the queue immediately. The list endpoint only queries jobs in the waiting, active, delayed, paused, and failed states - completed jobs never appear here. To keep failed jobs visible in the list, configure the [Job History Limit](#keep-completedfailed-jobs) setting. Retained completed jobs are only visible in Bull Board.
:::

#### Get a specific message

Retrieve details for a single queued message by its queue ID:

```bash
curl "https://emailengine.example.com/v1/outbox/4646ac53857fd2b2" \
  -H "Authorization: Bearer <token>"
```

**Response**:

```json
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
  "created": "2025-05-14T10:00:00.000Z",
  "scheduled": "2025-05-14T10:00:00.000Z",
  "nextAttempt": "2025-05-14T10:10:35.465Z",
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
```

:::warning
This endpoint only works for messages that are still queued (waiting, active, delayed, or paused). Once a message is completed or has permanently failed, its content is removed from Redis and this endpoint returns a 404 error - even if the job metadata is retained in BullMQ via the Job History Limit setting.
:::

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

Useful for:
- Removing stuck jobs
- Canceling scheduled sends
- Clearing failed jobs (if retention is enabled)

:::info Retrying Failed Jobs
To retry a failed job, you need to delete it from the queue and resubmit the message using the submit API. The outbox API does not support automatic retry of individual jobs. Alternatively, use the **Retry** button in Bull Board to retry failed jobs directly.
:::

## Configuration

### Delivery Attempts

Configure maximum retry attempts via the web UI or API:

**Via Web UI**:
1. Navigate to **Configuration -> Email Processing**
2. Set "Retry Attempts" under the Email Delivery card (default: 10)

**Via API when submitting a message**:
```json
{
  "to": [{"address": "recipient@example.com"}],
  "subject": "Test",
  "text": "Hello",
  "deliveryAttempts": 5
}
```

Default: 10 attempts

Note that [permanent errors](#4-failed) (such as authentication failures or invalid recipients) cause immediate failure regardless of this setting.

### Keep Completed/Failed Jobs

By default, completed and failed jobs are removed from the queue immediately to save Redis memory. When removed, they no longer appear in the outbox API or Bull Board.

**Enable retention**:

1. Navigate to **Configuration -> General**
2. Set **Job History Limit** (`queueKeep`) to the number of completed/failed jobs to keep
3. Example: Set to 100 to keep the last 100 completed and 100 failed jobs

Retained jobs are also automatically removed after 24 hours, whichever limit is reached first.

:::warning
This setting only affects new jobs created after the change. Existing jobs keep their original retention policy. Also note that even with retention enabled, the `GET /v1/outbox/{queueId}` endpoint returns 404 for completed and failed jobs because the message content is cleaned up on completion/failure. Retained failed jobs are visible through the `GET /v1/outbox` list endpoint and Bull Board; retained completed jobs are only visible in Bull Board.
:::

### SMTP Timeout

The SMTP socket timeout is set to 2 minutes (120 seconds). This is the maximum time allowed for SMTP operations before timing out.

## Webhook Events

The queue system triggers webhooks at key points:

### messageSent

Emitted when a job completes successfully:

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

Emitted on every failed delivery attempt - including the final one, where `messageFailed` is then also emitted. The `job.nextAttempt` field shows when the next retry is scheduled:

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
      "nextAttempt": "2025-05-14T10:10:35.465Z"
    },
    "envelope": {
      "from": "sender@example.com",
      "to": ["recipient@example.com"]
    }
  }
}
```

### messageFailed

Emitted when a job permanently fails (all retries exhausted or a non-retriable error):

```json
{
  "event": "messageFailed",
  "account": "example",
  "date": "2025-05-14T11:58:50.181Z",
  "data": {
    "messageId": "<message-id@example.com>",
    "queueId": "4646ac53857fd2b2",
    "error": "Error: Invalid login: 535 5.7.8 Error: authentication failed",
    "envelope": {
      "from": "sender@example.com",
      "to": ["recipient@example.com"]
    }
  }
}
```

## See Also

- [Queue Management](/docs/advanced/queue-management) - Detailed guide on BullMQ internals, Bull Board, and performance tuning
- [Basic Sending](/docs/sending/basic-sending) - How to submit emails via the API
- [Webhook Events](/docs/webhooks/overview) - Complete webhook event reference
