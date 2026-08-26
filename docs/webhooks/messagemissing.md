---
title: "messageMissing"
sidebar_position: 6
description: "Webhook event triggered when a message that should exist is not found, indicating a synchronization error"
---

# messageMissing

The `messageMissing` webhook event is triggered when EmailEngine learns about a new message but cannot fetch it from the mail server. It is sent in place of the [`messageNew`](/docs/webhooks/messagenew) event that the message would otherwise have produced.

## When This Event is Triggered

The `messageMissing` event fires when:

- **IMAP**: a new UID appeared in the folder listing, but the server returned nothing when EmailEngine fetched the message, and three retries did not change that. Servers with replication lag between the listing and the fetch, and messages deleted or moved by a filter right after arrival, are the usual causes
- **Gmail API**: the history reported a new message, but fetching it by ID returned nothing
- **MS Graph**: a change notification reported a new message, but fetching it by ID returned nothing

On IMAP accounts the retries are spaced 1.7<sup>n</sup> seconds apart: 1000 ms, then 1700 ms, then 2890 ms, so the event is sent about 5.6 seconds after the first failed fetch. Gmail API and MS Graph accounts do not retry.

## Common Use Cases

- **Sync error monitoring** - Track and alert on message retrieval failures
- **Debugging mail server issues** - Identify replication lag or server problems
- **Retry scheduling** - Implement custom retry logic for critical accounts
- **Analytics** - Monitor synchronization health across accounts
- **Audit logging** - Track cases where expected messages could not be retrieved
- **Alert systems** - Notify administrators of persistent sync issues

## Payload Schema

### Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `serviceUrl` | string or null | Yes | The configured EmailEngine service URL, `null` if not set |
| `account` | string | Yes | Account ID where the missing message was detected |
| `date` | string | Yes | ISO 8601 timestamp when the webhook was generated |
| `path` | string | Yes | Folder the message should have been in. IMAP accounts report the folder path (for example `INBOX`). Gmail API and MS Graph accounts report `\All` |
| `specialUse` | string | No | Special use flag of the folder, for example `\Inbox`. Gmail API and MS Graph accounts report `\All` |
| `event` | string | Yes | Always `messageMissing` |
| `data` | object | Yes | Message identification and retry data (see below) |

The unique event identifier is sent as the HTTP header `X-EE-Wh-Event-Id`, not in the JSON payload.

### Message Data Fields (`data` object)

#### IMAP Accounts

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | EmailEngine message ID (base64url-packed folder and UID) |
| `uid` | number | Yes | IMAP UID of the missing message within the folder |
| `missingRetries` | number | Yes | Number of retries made before giving up, always `3` |
| `missingDelay` | number | Yes | Total time in milliseconds spent waiting between the attempts, always `5590` |

A message that the server did return on one of the retries produces a normal `messageNew` event instead, carrying the same two fields with the counts it took.

#### Gmail API Accounts

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Gmail message ID that could not be retrieved |

#### Microsoft Graph (Outlook) Accounts

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Graph message ID that could not be retrieved |

## Example Payloads

### IMAP Account

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T06:44:14.660Z",
  "path": "INBOX",
  "specialUse": "\\Inbox",
  "event": "messageMissing",
  "data": {
    "id": "AAAADAAABy4",
    "uid": 1838,
    "missingRetries": 3,
    "missingDelay": 5590
  }
}
```

### Gmail API Account

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "gmail-user",
  "date": "2025-10-17T08:15:22.123Z",
  "path": "\\All",
  "specialUse": "\\All",
  "event": "messageMissing",
  "data": {
    "id": "18b5c7d8e9f01234"
  }
}
```

### Microsoft Outlook Account

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "outlook-user",
  "date": "2025-10-17T09:30:45.789Z",
  "path": "\\All",
  "specialUse": "\\All",
  "event": "messageMissing",
  "data": {
    "id": "AAMkADI2NGVhZTVlLTI1OGItNDUwZS05ZDVkLWQzN2E2MDUyYzc3YQBGAAAAAAI"
  }
}
```

## Handling the Event

### Basic Handler

```javascript
async function handleMessageMissing(event) {
  const { account, path, data } = event;

  console.log(`Missing message detected for ${account}:`);
  console.log(`  Message ID: ${data.id}`);
  console.log(`  Folder: ${path}`);
  if (data.uid) {
    console.log(`  UID: ${data.uid}`);
  }
  if (data.missingRetries) {
    console.log(`  Retry attempts: ${data.missingRetries}`);
    console.log(`  Total delay: ${data.missingDelay}ms`);
  }

  await logSyncIssue(account, data.id, 'message_missing');
}
```

### Monitoring and Alerting

The event identifier is in the `X-EE-Wh-Event-Id` request header, so a record that keeps it needs both the header and the body:

```javascript
async function handleMessageMissing(req) {
  const eventId = req.headers['x-ee-wh-event-id'];
  const { account, path, date, data } = req.body;

  await db.syncIssues.create({
    data: {
      eventId,
      timestamp: new Date(date),
      account,
      folder: path,
      issueType: 'message_missing',
      messageId: data.id,
      uid: data.uid || null,
      retryAttempts: data.missingRetries || 0,
      totalDelay: data.missingDelay || 0
    }
  });

  const recentIssues = await db.syncIssues.count({
    where: {
      account,
      issueType: 'message_missing',
      timestamp: {
        gte: new Date(Date.now() - 3600000)
      }
    }
  });

  if (recentIssues >= 5) {
    await alerting.send({
      level: 'warning',
      title: 'Multiple missing messages detected',
      message: `Account ${account} has ${recentIssues} missing messages in the last hour`,
      metadata: { account, recentIssues }
    });
  }
}
```

### Fetching the Message Later

The `id` stays valid for as long as the message exists in that folder, so a handler can try again through the API after a delay:

```javascript
async function handleMessageMissing(event) {
  const { account, data } = event;

  await jobQueue.add(
    'retry-fetch-message',
    {
      account,
      messageId: data.id,
      attempt: 1
    },
    {
      delay: 300000
    }
  );
}

async function retryFetchMessage(job) {
  const { account, messageId } = job.data;

  const response = await fetch(
    `https://emailengine.example.com/v1/account/${account}/message/${messageId}`,
    {
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN'
      }
    }
  );

  if (response.status === 404) {
    console.log(`Message ${messageId} is gone for good`);
    return;
  }

  const message = await response.json();
  await processNewEmail(account, message);
}
```

## Important Considerations

### This Event is Not Always an Error

A `messageMissing` event does not necessarily indicate a problem. It also occurs when:

- A user deletes a message right after it arrives
- A server-side rule or spam filter moves or deletes the message before EmailEngine fetches it
- The IMAP server's listing runs ahead of its message store, which some clustered servers do briefly

Consider the frequency of these events when deciding how to handle them.

### Correlation with Other Events

On IMAP accounts the UID was added to EmailEngine's index before the fetch was attempted, so under the full [indexer](/docs/accounts/imap-indexers) the server's later expunge of it produces a [`messageDeleted`](/docs/webhooks/messagedeleted) event carrying the same `id` and `uid`. Track the `id` to correlate the two.

## Comparing to messageDeleted

| Aspect | messageMissing | messageDeleted |
|--------|----------------|----------------|
| **Trigger** | A new message could not be fetched | A tracked message was removed |
| **Content accessed** | Never successfully fetched | Was fetched at least once |
| **Common cause** | Timing, filters, server problems | User action, filter rules, API deletion |
| **Includes retry stats** | Yes (IMAP only) | No |
| **Indicates error** | Potentially | No |

## Related Events

- [messageNew](/docs/webhooks/messagenew) - Triggered when a new message is fetched successfully
- [messageDeleted](/docs/webhooks/messagedeleted) - Triggered when a tracked message is removed
- [connectError](/docs/webhooks/connecterror) - Triggered when connection to email server fails
- [authenticationError](/docs/webhooks/authenticationerror) - Triggered when authentication fails

## See Also

- [Webhooks Overview](/docs/webhooks/overview) - Delivery, retries, headers and signing
- [Message API](/docs/api/get-v-1-account-account-message-message) - Fetching a message by `id` later
- [IMAP indexers](/docs/accounts/imap-indexers) - How EmailEngine tracks what is in a folder
- [Troubleshooting](/docs/troubleshooting) - Diagnosing sync issues
- [Settings API](/docs/api/post-v-1-settings) - Configure webhook settings
