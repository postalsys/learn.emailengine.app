---
title: Exporting Messages
sidebar_position: 11
description: "Bulk export email messages with configurable concurrency for efficient archival and analysis"
keywords:
  - export emails
  - bulk export
  - message archival
  - NDJSON export
  - email backup
---

# Exporting Messages

EmailEngine can export the messages of an account, in bulk, to a gzip-compressed NDJSON file. Exports run in the background on a dedicated worker, and the five endpoints of the [Export (Beta)](/docs/api/post-v-1-account-account-export) tag create, monitor, download, list and delete them.

:::info Beta
Export is labeled beta in the API. The endpoints and the file format described here are what ships today; check the changelog before relying on them across an upgrade.
:::

## Overview

The export feature:

- Creates gzip-compressed NDJSON files containing message data
- Processes exports asynchronously via a job queue
- Supports date range filtering and folder selection
- Optionally includes message text content and attachments
- Automatically encrypts export files when `EENGINE_SECRET` is configured
- Provides progress tracking and status monitoring

**Common use cases:**
- Email backup and archival
- Migration to other systems
- Compliance and legal discovery
- Data analysis and machine learning training
- Bulk message processing pipelines

## Creating an Export

Create a new export job using the [Create Export API endpoint](/docs/api/post-v-1-account-account-export):

```bash
curl -X POST "https://emailengine.example.com/v1/account/user123/export" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "startDate": "2024-01-01T00:00:00Z",
    "endDate": "2024-12-31T23:59:59Z",
    "folders": ["INBOX", "\\Sent"],
    "textType": "*",
    "maxBytes": 5242880,
    "includeAttachments": false
  }'
```

### Request Options

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `startDate` | ISO 8601 | Required | Export messages from this date |
| `endDate` | ISO 8601 | Required | Export messages until this date |
| `folders` | array | All Mail (Gmail/Outlook API); all folders except Junk and Trash (other accounts) | Folder paths or special-use flags such as `\Sent` to export |
| `textType` | string | `*` | Text content: `plain`, `html`, `*` (both) |
| `maxBytes` | number | 5242880 | Maximum bytes for text content per message (0 = unlimited) |
| `includeAttachments` | boolean | false | Include attachment content as base64 in each message's `attachments` array |

With `includeAttachments`, a message larger than 50 MB is skipped entirely and counted in `messagesSkipped`, and a single attachment larger than 50 MB is written with a `contentError` field instead of `content`. That limit is not configurable.

### Response

```json
{
  "exportId": "exp_abc123def456abc123def456",
  "status": "queued",
  "created": "2024-01-15T10:30:00.000Z"
}
```

The request is refused with `429` and the message `Maximum concurrent exports reached` when the account already has `exportMaxConcurrent` exports running, or the instance has `exportMaxGlobalConcurrent`; see [Export Limits](#export-limits).

## Monitoring Export Progress

Check export status using the [Get Export Status API endpoint](/docs/api/get-v-1-account-account-export-exportid):

```bash
curl "https://emailengine.example.com/v1/account/user123/export/exp_abc123def456abc123def456" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

### Export States

Exports progress through these states:

| Status | Phase | Description |
|--------|-------|-------------|
| `queued` | - | Export is waiting in the queue (no `phase` field is set) |
| `processing` | `indexing` | Scanning folders and queuing messages |
| `processing` | `exporting` | Fetching and writing messages to file |
| `completed` | `complete` | Export finished successfully |
| `failed` | - | Export encountered an error |
| `cancelled` | - | Export was cancelled before completion |

### Progress Fields

The response includes the request parameters, progress counters and the file's expiry:

```json
{
  "exportId": "exp_abc123def456abc123def456",
  "status": "processing",
  "phase": "exporting",
  "folders": ["INBOX", "\\Sent"],
  "startDate": "2024-01-01T00:00:00.000Z",
  "endDate": "2024-12-31T23:59:59.000Z",
  "isEncrypted": false,
  "progress": {
    "foldersScanned": 2,
    "foldersTotal": 3,
    "messagesQueued": 1500,
    "messagesExported": 750,
    "messagesSkipped": 5,
    "bytesWritten": 52428800
  },
  "created": "2024-01-15T10:30:00.000Z",
  "expiresAt": "2024-01-16T10:30:00.000Z",
  "error": null
}
```

| Field | Description |
|-------|-------------|
| `foldersScanned` | Number of folders indexed so far |
| `foldersTotal` | Total folders to index |
| `messagesQueued` | Messages found and queued for export |
| `messagesExported` | Messages successfully written to file |
| `messagesSkipped` | Messages skipped (deleted or inaccessible) |
| `bytesWritten` | Total bytes written to export file |

`bytesWritten` counts uncompressed NDJSON bytes, not the size of the file on disk. A top-level `truncated: true` appears when the export was cut short by `exportMaxMessages` or `exportMaxSize` (see [Export Limits](#export-limits)); it is absent otherwise. The export still completes, but the file does not contain every matching message. `error` carries the failure reason for a `failed` export and is `null` otherwise.

## Downloading Export Files

Download a completed export using the [Download Export API endpoint](/docs/api/get-v-1-account-account-export-exportid-download):

```bash
curl "https://emailengine.example.com/v1/account/user123/export/exp_abc123def456abc123def456/download" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -o export.ndjson.gz
```

The download is only available once `status` is `completed`; before that the endpoint answers `400`, and after the file has expired or been deleted it answers `404`. The response is a gzip-compressed NDJSON file, served with `Content-Disposition: attachment` and the filename `<exportId>.ndjson.gz`. Each line is one message in the same shape as [`GET /v1/account/{account}/message/{message}`](/docs/api/get-v-1-account-account-message-message) returns, plus a `path` field naming the folder it came from:

```text
{"id":"AAAAAQAACnA","uid":12345,"path":"INBOX","emailId":"1789473523904663830","subject":"Hello","from":{"name":"Sender","address":"sender@example.com"},"to":[{"name":"","address":"recipient@example.com"}],"date":"2024-01-15T10:30:00.000Z","messageId":"<a1b2c3@example.com>","flags":["\\Seen"],"text":{"id":"AAAAAQAACnAAAAAB","encodedSize":{"plain":42},"plain":"Meeting notes attached, see you Tuesday."},"attachments":[]}
{"id":"AAAAAQAACnB","uid":12346,"path":"INBOX","emailId":"1789473523904663831","subject":"Re: Hello","from":{"name":"Reply","address":"reply@example.com"},"to":[{"name":"Sender","address":"sender@example.com"}],"date":"2024-01-15T11:00:00.000Z","messageId":"<d4e5f6@example.com>","inReplyTo":"<a1b2c3@example.com>","flags":[],"text":{"id":"AAAAAQAACnBAAAAB","encodedSize":{"plain":19},"plain":"Tuesday works for me."},"attachments":[]}
```

If the export was encrypted (when `EENGINE_SECRET` is set), decryption happens automatically during download.

## Concurrency Tuning

Export jobs are processed by dedicated worker threads. You can tune concurrency based on your system resources.

### Configuration Options

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `EENGINE_WORKERS_EXPORT` | env | 1 | Export worker threads |
| `EENGINE_EXPORT_QC` | env | 1 | Concurrent jobs per worker |
| `exportMaxConcurrent` | setting | 2 | Max concurrent exports per account |
| `exportMaxGlobalConcurrent` | setting | 8 | Max concurrent exports system-wide |

### Calculating Total Concurrency

The maximum number of exports that can run simultaneously is:

```
MAX_CONCURRENT = EENGINE_WORKERS_EXPORT x EENGINE_EXPORT_QC
```

This is further capped by `exportMaxGlobalConcurrent` to prevent system overload.

**Example**: With `EENGINE_WORKERS_EXPORT=2` and `EENGINE_EXPORT_QC=2`, you can have up to 4 concurrent exports. If `exportMaxGlobalConcurrent=8`, the global limit won't be a factor. But if you set `exportMaxGlobalConcurrent=3`, only 3 exports will run concurrently even though the worker configuration allows 4.

### What an Export Costs

Each running export holds one connection to the mail server for the account, keeps a queue of message references in Redis while it runs, and streams messages through gzip (and, when encrypted, AES-256-GCM) to one file. Concurrency multiplies all three. There are no published memory figures; watch the instance under a representative export before raising the defaults.

Every request the export makes to the mail server uses the timeout in `EENGINE_EXPORT_TIMEOUT` (default 5 minutes), which is separate from the API's `EENGINE_TIMEOUT`.

### Provider-Specific Batch Sizes

EmailEngine fetches messages in batches during export. The batch size can be tuned per email provider to optimize throughput and avoid rate limits.

Configure via the [Settings API](/docs/api/post-v-1-settings):

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "gmailExportBatchSize": 10,
    "outlookExportBatchSize": 20
  }'
```

| Setting | Default | Range | Description |
|---------|---------|-------|-------------|
| `gmailExportBatchSize` | 10 | 1-50 | Messages fetched in parallel from Gmail API per batch |
| `outlookExportBatchSize` | 20 | 1-20 | Messages fetched in parallel from Microsoft Graph API per batch |

**Gmail:** Supports up to 50 parallel fetches. Higher values speed up exports but increase memory usage and may trigger rate limits on accounts with heavy concurrent usage.

**Outlook:** Microsoft Graph API limits batch requests to 20 items per batch. Setting a higher value has no effect.

**IMAP accounts:** Batch size is not configurable for IMAP - messages are fetched one at a time over the account's connection.

### Export Limits

Additional settings control maximum export sizes:

| Setting | Default | Description |
|---------|---------|-------------|
| `exportMaxMessages` | 500,000 | Maximum messages per export job. Indexing stops here and the export is marked `truncated` |
| `exportMaxSize` | 10 GB | Maximum uncompressed NDJSON bytes per export. Writing stops here and the export is marked `truncated` |
| `exportMaxConcurrent` | 2 | Max concurrent exports per account. A request over this is refused with `429` |
| `exportMaxGlobalConcurrent` | 8 | Max concurrent exports system-wide. A request over this is refused with `429` |

### Tuning Considerations

1. **Memory**: Each export batch loads message data into memory, and `includeAttachments` loads every attachment up to the 50 MB limit. Reduce concurrency if you see memory pressure.

2. **Disk I/O**: Multiple concurrent gzip streams write to `EENGINE_EXPORT_PATH` at the same time.

3. **Email Provider Limits**: A rate-limited batch is retried up to five times with exponential backoff starting at 5 seconds before the messages in it are skipped. Watch for those retries in the logs before raising the provider batch sizes.

4. **Redis**: The index of messages to export lives in Redis for the life of the export and expires with it.

**Tuning tips:**
- Start with conservative settings and increase gradually
- Monitor memory usage with `docker stats` or `top`
- Check logs for rate limiting errors from email providers
- Use `exportMaxGlobalConcurrent` to cap total system load regardless of worker configuration

## File Storage

### Configuration

File storage is configured with environment variables (these options are not available through the settings API or UI):

| Environment Variable | Default | Description |
|----------------------|---------|-------------|
| `EENGINE_EXPORT_PATH` | OS temp dir | Directory for export files. Created if missing |
| `EENGINE_EXPORT_MAX_AGE` | 24 hours | File retention time in milliseconds. Sets `expiresAt` on each export; the record and the file are removed after it |
| `EENGINE_EXPORT_TIMEOUT` | 5 minutes | Timeout for each request the export makes to the mail server |

### Encryption

When `EENGINE_SECRET` is configured, export files are automatically encrypted using AES-256-GCM:

- Encrypted files have `.ndjson.gz.enc` extension
- Unencrypted files have `.ndjson.gz` extension
- Downloads are automatically decrypted by EmailEngine

This ensures exported data is protected at rest without requiring separate encryption handling.

## Webhooks

Export completion triggers webhook notifications. Both need to be in the `webhookEvents` allowlist (or `*`) to be delivered:

| Event | Description |
|-------|-------------|
| `exportCompleted` | Export finished, with the counters and the file's `expiresAt` |
| `exportFailed` | Export failed. Not sent when an export was cancelled through the API or its account was deleted |

Example webhook payload for `exportCompleted`:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2024-01-15T10:42:11.000Z",
  "event": "exportCompleted",
  "data": {
    "exportId": "exp_abc123def456abc123def456",
    "folders": ["INBOX", "\\Sent"],
    "startDate": "2024-01-01T00:00:00.000Z",
    "endDate": "2024-12-31T23:59:59.000Z",
    "messagesExported": 1495,
    "messagesSkipped": 5,
    "bytesWritten": 104857600,
    "duration": 731000,
    "expiresAt": "2024-01-16T10:30:00.000Z"
  }
}
```

The `exportFailed` payload carries `exportId`, `error`, `errorCode`, the `phase` the export was in, `messagesExported` and `messagesQueued`. Both events are documented on the [exportCompleted and exportFailed](/docs/webhooks/exportcompleted) reference.

## Managing Exports

### List Exports

Get all exports for an account using the [List Exports API endpoint](/docs/api/get-v-1-account-account-exports):

```bash
curl "https://emailengine.example.com/v1/account/user123/exports" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

Response:

```json
{
  "total": 3,
  "page": 0,
  "pages": 1,
  "exports": [
    {
      "exportId": "exp_abc123def456abc123def456",
      "status": "completed",
      "created": "2024-01-15T10:30:00.000Z",
      "expiresAt": "2024-01-16T10:30:00.000Z"
    }
  ]
}
```

### Delete Export

Cancel a pending export or delete a completed export file using the [Delete Export API endpoint](/docs/api/delete-v-1-account-account-export-exportid):

```bash
curl -X DELETE "https://emailengine.example.com/v1/account/user123/export/exp_abc123def456abc123def456" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

```json
{
  "deleted": true
}
```

This will:
- Cancel the export if it's still queued or processing. A running export is marked `cancelled` and the worker removes its partial file and record when it notices, which is within one batch
- Delete the export file from disk
- Remove the export record from the system

### Handling Failed Exports

Failed exports cannot be resumed. If an export ends up in the `failed` state, check the `error` field in the status response for details, then delete the failed export and create a new one (using a narrower date range if the failure was caused by size or rate limits).

## Best Practices

### Large Exports

For very large exports (millions of messages):

1. **Use date range filtering** - Split large exports into smaller date ranges
2. **Monitor progress** - Poll the status endpoint to track completion
3. **Handle failures gracefully** - Check the `error` field if status is `failed`; delete the failed export and create a new one (exports cannot be resumed)
4. **Download promptly** - Files expire after the retention period set by `EENGINE_EXPORT_MAX_AGE` (default 24 hours)

### Production Usage

1. **Configure storage path** - Set the `EENGINE_EXPORT_PATH` environment variable to a dedicated volume with sufficient space
2. **Set appropriate retention** - Adjust the `EENGINE_EXPORT_MAX_AGE` environment variable (in milliseconds) based on your download SLA
3. **Monitor disk space** - Large exports can consume significant disk space
4. **Use webhooks** - Set up webhook handlers for `exportCompleted` and `exportFailed` events instead of polling

### Processing Export Files

NDJSON format allows streaming processing without loading the entire file into memory:

```javascript
const readline = require('readline');
const zlib = require('zlib');
const fs = require('fs');

const gunzip = zlib.createGunzip();
const input = fs.createReadStream('export.ndjson.gz').pipe(gunzip);

const rl = readline.createInterface({ input });

rl.on('line', (line) => {
  const message = JSON.parse(line);
  // Process each message
  console.log(`Processing: ${message.subject}`);
});
```

## See Also

- [exportCompleted and exportFailed](/docs/webhooks/exportcompleted) - Knowing when an export finished
- [Continuous processing](/docs/receiving/continuous-processing) - Keeping up with new mail once history is exported
- [Performance tuning](/docs/advanced/performance-tuning) - What a large export costs the instance
- [Export API](/docs/api/post-v-1-account-account-export) - The endpoint reference
