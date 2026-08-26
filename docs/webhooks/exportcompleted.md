---
title: "exportCompleted"
sidebar_position: 25
description: "Webhook event triggered when a bulk email export job finishes successfully"
---

# exportCompleted

The `exportCompleted` webhook event is triggered when EmailEngine finishes a bulk message export. It carries the export's statistics and the time the file will be deleted, and it means the file is ready for download.

## When This Event is Triggered

The `exportCompleted` event fires when:

- Every queued message has been written or skipped
- The export file has been closed
- The export has been marked `completed`

The event confirms that the file is available through the [Download Export API](/docs/api/get-v-1-account-account-export-exportid-download). An export that was cut short by the message or size limit still completes and sends this event; the [export status](/docs/receiving/exporting#progress-fields) then reports `truncated: true`.

## Common Use Cases

- **Download automation** - Trigger automatic download of completed exports
- **Notification systems** - Alert users when their export is ready
- **Workflow triggers** - Initiate downstream processing pipelines
- **Audit logging** - Track export completion for compliance purposes
- **Cleanup scheduling** - Delete the export once it has been downloaded

## Payload Schema

### Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `serviceUrl` | string or null | Yes | The configured EmailEngine service URL. `null` when the `serviceUrl` setting is empty |
| `account` | string | Yes | Account ID the export belongs to |
| `date` | string | Yes | ISO 8601 timestamp when the webhook was generated |
| `event` | string | Yes | Always `exportCompleted` |
| `data` | object | Yes | Export details (see below) |

### Data Object Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `exportId` | string | Yes | Identifier of the export job |
| `folders` | array | Yes | The `folders` value the export was created with: folder paths or special-use flags. An empty array when none were given, which exports the [default folder set](/docs/receiving/exporting#request-options) |
| `startDate` | string | Yes | ISO 8601 start of the exported date range |
| `endDate` | string | Yes | ISO 8601 end of the exported date range |
| `messagesExported` | number | Yes | Messages written to the export file |
| `messagesSkipped` | number | Yes | Messages that were queued but could not be fetched, see below |
| `bytesWritten` | number | Yes | Bytes of NDJSON written, before compression |
| `duration` | number | Yes | Milliseconds from the moment the worker picked the job up to completion |
| `expiresAt` | string | Yes | ISO 8601 time after which the export is deleted |

There is no event ID in the body. EmailEngine sends it in the `X-EE-Wh-Event-Id` request header. See [Delivery and Retries](/docs/webhooks/overview#delivery-and-retries).

:::warning Download before `expiresAt`
The export file is deleted when it expires, and cannot be recovered afterwards. Treat this event as the start of a deadline, not only as a completion notice. The retention window is the `exportMaxAge` setting, or the `EENGINE_EXPORT_MAX_AGE` environment variable when the setting is empty, and 24 hours by default. It is counted from completion, so a long-running export gets the full window.
:::

## Example Payload

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-01-15T14:30:00.000Z",
  "event": "exportCompleted",
  "data": {
    "exportId": "exp_abc123def456abc123def456",
    "folders": ["INBOX", "\\Sent"],
    "startDate": "2024-01-01T00:00:00.000Z",
    "endDate": "2025-01-01T00:00:00.000Z",
    "messagesExported": 1495,
    "messagesSkipped": 5,
    "bytesWritten": 104857600,
    "duration": 125000,
    "expiresAt": "2025-01-16T14:30:00.000Z"
  }
}
```

### Default folder set

An export created without `folders` reports an empty array:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "archive-user",
  "date": "2025-01-15T18:45:00.000Z",
  "event": "exportCompleted",
  "data": {
    "exportId": "exp_xyz789ghi012xyz789ghi012",
    "folders": [],
    "startDate": "2015-01-01T00:00:00.000Z",
    "endDate": "2025-01-01T00:00:00.000Z",
    "messagesExported": 45230,
    "messagesSkipped": 127,
    "bytesWritten": 2147483648,
    "duration": 3600000,
    "expiresAt": "2025-01-16T18:45:00.000Z"
  }
}
```

## Field Details

### messagesExported vs messagesSkipped

- **`messagesExported`**: Messages fetched and written to the export file
- **`messagesSkipped`**: Messages that were listed while indexing but could not be fetched afterwards: the message was deleted or moved between indexing and export, the server answered 404 for it, or EmailEngine could not generate an ID for it

A skipped message is dropped, not retried. Transient fetch errors are retried and do not count as skipped; if the retries run out, the export fails instead and sends [`exportFailed`](/docs/webhooks/exportfailed). A folder that disappears mid-export is treated as empty rather than skipped.

A high `messagesSkipped` count on a busy mailbox usually means messages were deleted or moved while the export ran.

### bytesWritten

`bytesWritten` counts the NDJSON lines as written, before gzip compression, so it is larger than the downloaded file. It is the same figure the [export status](/docs/receiving/exporting#progress-fields) reports.

### duration

Time in milliseconds from the worker picking the job up to completion. Queue time before the job started is not included. Use it to estimate completion times for similar exports and to track throughput.

## Handling the Event

### Basic Handler

```javascript
async function handleExportCompleted(event) {
  const { account, data } = event;

  console.log(`Export completed for account ${account}`);
  console.log(`  Export ID: ${data.exportId}`);
  console.log(`  Messages: ${data.messagesExported} exported, ${data.messagesSkipped} skipped`);
  console.log(`  Written: ${(data.bytesWritten / 1024 / 1024).toFixed(2)} MB before compression`);
  console.log(`  Expires: ${data.expiresAt}`);

  // Update your database
  await db.exports.update({
    exportId: data.exportId,
    status: 'completed',
    messagesExported: data.messagesExported,
    completedAt: event.date,
    expiresAt: data.expiresAt
  });

  // Notify the user
  await notifyUser(account, {
    type: 'export_ready',
    exportId: data.exportId,
    messageCount: data.messagesExported
  });
}
```

### Automatic Download

```javascript
async function handleExportCompleted(event) {
  const { account, data } = event;

  // Download the export file
  const response = await fetch(
    `${EMAILENGINE_URL}/v1/account/${account}/export/${data.exportId}/download`,
    {
      headers: { 'Authorization': `Bearer ${API_TOKEN}` }
    }
  );

  // Save to storage
  const exportPath = `/exports/${account}/${data.exportId}.ndjson.gz`;
  await saveToStorage(exportPath, response.body);

  // Clean up the export from EmailEngine
  await fetch(
    `${EMAILENGINE_URL}/v1/account/${account}/export/${data.exportId}`,
    {
      method: 'DELETE',
      headers: { 'Authorization': `Bearer ${API_TOKEN}` }
    }
  );

  console.log(`Export ${data.exportId} downloaded and cleaned up`);
}
```

The download is encrypted when EmailEngine runs with a service secret. See [Encryption](/docs/receiving/exporting#encryption) for the file format.

### With Error Handling

```javascript
async function handleExportCompleted(event) {
  try {
    const { account, data, date } = event;

    // Log the completion
    await auditLog.create({
      type: 'export_completed',
      account,
      exportId: data.exportId,
      messagesExported: data.messagesExported,
      messagesSkipped: data.messagesSkipped,
      bytesWritten: data.bytesWritten,
      timestamp: new Date(date)
    });

    // Check for high skip rate
    const total = data.messagesExported + data.messagesSkipped;
    const skipRate = total ? data.messagesSkipped / total : 0;
    if (skipRate > 0.1) {
      console.warn(`High skip rate (${(skipRate * 100).toFixed(1)}%) for export ${data.exportId}`);
    }

    // Trigger downstream processing
    await processExportFile(account, data.exportId);

  } catch (error) {
    console.error('Failed to process exportCompleted webhook:', error);
    throw error; // Respond with an error status so EmailEngine retries the delivery
  }
}
```

## Relationship to Other Events

The `exportCompleted` event is part of the export lifecycle:

1. **Create Export API call** - Export job is created and queued
2. **Export processing** - Worker indexes folders and exports messages
3. **exportCompleted** - Export finished successfully (this event)
4. **exportFailed** - Export encountered a fatal error (alternative outcome)

After receiving `exportCompleted`:

- The export file is available for download until `expiresAt`
- Download it, then delete the export to free disk space

## Best Practices

1. **Download promptly** - The file is deleted at `expiresAt`
2. **Verify message counts** - Compare `messagesExported` with the count you expected
3. **Monitor skip rates** - A high skip rate points at a mailbox that changed while the export ran
4. **Process asynchronously** - Respond to the webhook first, download afterwards
5. **Clean up after download** - Delete exports to free disk space

## Related Events

- [exportFailed](/docs/webhooks/exportfailed) - The export stopped on an error

## See Also

- [Webhooks Overview](/docs/webhooks/overview) - Configuring the webhook URL and the `webhookEvents` allowlist
- [Exporting Messages](/docs/receiving/exporting) - Creating exports, status fields, limits and retention
- [Download Export API](/docs/api/get-v-1-account-account-export-exportid-download) - Fetching the file this event announces
- [Delete Export API](/docs/api/delete-v-1-account-account-export-exportid) - Removing the export once it has been downloaded
