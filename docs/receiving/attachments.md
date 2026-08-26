---
title: Working with Attachments
sidebar_position: 7
description: "Listing and downloading email attachments, inline images and cid references, attachment metadata, and attachments in webhook payloads"
keywords:
  - email attachments
  - download attachments
  - inline images
  - file attachments
  - attachment handling
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Working with Attachments

Attachments are listed with the message and fetched on demand. This page covers [downloading them](/docs/api/get-v-1-account-account-attachment-attachment), the inline images an HTML body references, the metadata that comes with each one, and how attachments reach a webhook payload. Attaching files to an outgoing message is covered in [Basic sending](/docs/sending/basic-sending).

## Understanding Attachments

### Attachment Types

**Regular Attachments**
- Files explicitly attached to the email
- Displayed in attachment list
- `embedded: false`, `inline: false`
- Examples: PDFs, documents, spreadsheets

**Inline Attachments**
- Images embedded in HTML content
- Referenced with `cid:` URLs
- `embedded: true` or `inline: true`
- Examples: logos, signatures, embedded images

**Both Types**
- EmailEngine handles both transparently
- Each attachment has a unique ID
- Content-Type indicates file type

### Attachment Metadata

Each attachment includes:

```json
{
  "id": "AAAAAgAAAeEBAAAAAQAAAeE",
  "contentType": "application/pdf",
  "filename": "invoice.pdf",
  "encodedSize": 45000,
  "embedded": false,
  "inline": false,
  "encodedInMessage": false,
  "contentId": "<unique-image-id@localhost>"
}
```

| Field | Type | Meaning |
|-------|------|---------|
| `id` | string | Attachment identifier. This is the value the download endpoint takes. It encodes the message and the MIME part, so it changes when the message moves; see [EmailEngine IDs Explained](/docs/advanced/ids-explained) |
| `contentType` | string | MIME type of the part |
| `filename` | string | Original filename. Absent when the part carries none |
| `encodedSize` | integer | Size as stored in the message, that is base64-encoded. The decoded file is roughly 75% of this |
| `embedded` | boolean | Referenced from the HTML body by a `cid:` URL |
| `inline` | boolean | The part is marked `Content-Disposition: inline` rather than `attachment` |
| `encodedInMessage` | boolean | The part belongs to an enclosed `message/rfc822` attachment rather than the message itself |
| `contentId` | string | The `Content-ID` header, angle brackets included, for example `<unique-image-id@localhost>`. `cid:` references in the HTML use the value without the brackets |
| `method` | string | Calendar method (`REQUEST`, `REPLY`, `CANCEL` and so on) for iCalendar attachments |

Attachment content is never part of a message response. The list carries metadata only; the bytes come from the download endpoint below, or, for webhooks, from the [attachment options](#attachments-in-webhook-payloads).

## Listing Attachments

### Get Message with Attachments

Fetch a message to see its attachments. The `getMessage` helper defined here is reused by the later examples on this page:

```javascript
const fs = require('fs');

const BASE_URL = 'https://emailengine.example.com';
const HEADERS = { Authorization: 'Bearer YOUR_ACCESS_TOKEN' };

async function getMessage(accountId, messageId) {
  const response = await fetch(
    `${BASE_URL}/v1/account/${accountId}/message/${messageId}`,
    { headers: HEADERS }
  );

  if (!response.ok) {
    throw new Error(`Fetching the message failed: ${response.status}`);
  }

  return await response.json();
}

const message = await getMessage('example', 'AAAAAQAAAeE');
console.log(`Message: ${message.subject}`);
console.log(`Attachments: ${(message.attachments || []).length}`);

for (const att of message.attachments || []) {
  console.log(`- ${att.filename || '(no filename)'} ${att.contentType}, ${att.encodedSize} bytes encoded`);
}
```

Listings from `GET /v1/account/{account}/messages` and search results carry the same `attachments` array, so a folder can be scanned for attachments without fetching each message.

### Filter by Attachment Type

```javascript
function filterAttachments(attachments, options = {}) {
  return attachments.filter(att => {
    // Filter by type
    if (options.type) {
      if (!att.contentType.includes(options.type)) {
        return false;
      }
    }

    // Filter out inline images
    if (options.excludeInline && att.embedded) {
      return false;
    }

    // Filter by size (using encodedSize - actual file will be smaller)
    if (options.minSize && att.encodedSize < options.minSize) {
      return false;
    }

    if (options.maxSize && att.encodedSize > options.maxSize) {
      return false;
    }

    return true;
  });
}

// Get only PDF attachments
const pdfs = filterAttachments(message.attachments, {
  type: 'pdf',
  excludeInline: true
});

// Get large attachments (>1MB)
const largeFiles = filterAttachments(message.attachments, {
  minSize: 1024 * 1024,
  excludeInline: true
});
```

## Downloading Attachments

### Download Single Attachment

Download an attachment by its ID:

```bash
curl "https://emailengine.example.com/v1/account/example/attachment/AAAAAgAAAeEBAAAAAQAAAeE" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  --output invoice.pdf
```

The response body is the decoded file, streamed as it is read from the mail server. `Content-Type` is the part's MIME type (`application/octet-stream` when the part declares none) and `Content-Disposition` is `attachment` with the original filename, RFC 2231 encoded when it is not plain ASCII. There is no size limit on the endpoint itself; the request timeout (`EENGINE_TIMEOUT`, or the `x-ee-timeout` header) and the mail server decide how long a large download may take. Errors are `404` when the attachment ID does not resolve and `503` when the account's connection is unavailable.

<Tabs groupId="programming-language">
<TabItem value="nodejs" label="Node.js">

```javascript
async function downloadAttachment(accountId, attachmentId, outputPath) {
  const response = await fetch(
    `${BASE_URL}/v1/account/${accountId}/attachment/${attachmentId}`,
    { headers: HEADERS }
  );

  if (!response.ok) {
    throw new Error(`Download failed: ${response.status}`);
  }

  const buffer = Buffer.from(await response.arrayBuffer());
  fs.writeFileSync(outputPath, buffer);

  return {
    path: outputPath,
    size: buffer.length
  };
}

// Download attachment
const result = await downloadAttachment(
  'example',
  'AAAAAgAAAeEBAAAAAQAAAeE',
  './downloads/invoice.pdf'
);

console.log(`Downloaded to ${result.path} (${result.size} bytes)`);
```

</TabItem>
<TabItem value="python" label="Python">

```python
import requests

def download_attachment(account_id, attachment_id, output_path):
    url = f"https://emailengine.example.com/v1/account/{account_id}/attachment/{attachment_id}"
    headers = {"Authorization": "Bearer YOUR_ACCESS_TOKEN"}

    response = requests.get(url, headers=headers)
    response.raise_for_status()

    with open(output_path, 'wb') as f:
        f.write(response.content)

    return {
        'path': output_path,
        'size': len(response.content)
    }

# Download attachment
result = download_attachment(
    'example',
    'AAAAAgAAAeEBAAAAAQAAAeE',
    './downloads/invoice.pdf'
)

print(f"Downloaded to {result['path']} ({result['size']} bytes)")
```

</TabItem>
</Tabs>

### Download All Attachments

Download all attachments from a message:

```javascript
async function downloadAllAttachments(accountId, messageId, outputDir) {
  // Get message to find attachments
  const message = await getMessage(accountId, messageId);

  if (!message.attachments || message.attachments.length === 0) {
    return [];
  }

  // Create output directory if needed
  if (!fs.existsSync(outputDir)) {
    fs.mkdirSync(outputDir, { recursive: true });
  }

  const downloads = [];

  for (const attachment of message.attachments) {
    // Skip inline images
    if (attachment.embedded) {
      continue;
    }

    // Generate safe filename
    const filename = attachment.filename || `attachment-${attachment.id}`;
    const safeName = filename.replace(/[^a-zA-Z0-9.-]/g, '_');
    const outputPath = `${outputDir}/${safeName}`;

    try {
      const result = await downloadAttachment(accountId, attachment.id, outputPath);

      downloads.push({
        ...result,
        originalName: attachment.filename,
        contentType: attachment.contentType
      });

      console.log(`Downloaded: ${filename}`);
    } catch (err) {
      console.error(`Failed to download ${filename}:`, err.message);
    }
  }

  return downloads;
}

// Download all attachments from a message
const downloads = await downloadAllAttachments(
  'example',
  'AAAAAQAAAeE',
  './downloads/message-AAAAAQAAAeE'
);

console.log(`Downloaded ${downloads.length} attachments`);
```

### Download to Memory

For processing without saving to disk:

```javascript
async function downloadToMemory(accountId, attachmentId) {
  const response = await fetch(
    `${BASE_URL}/v1/account/${accountId}/attachment/${attachmentId}`,
    { headers: HEADERS }
  );

  if (!response.ok) {
    throw new Error(`Download failed: ${response.status}`);
  }

  const buffer = Buffer.from(await response.arrayBuffer());
  const contentType = response.headers.get('content-type');

  return {
    buffer,
    contentType,
    size: buffer.length
  };
}

// Process attachment in memory
const data = await downloadToMemory('example', 'AAAAAgAAAeEBAAAAAQAAAeE');

// Parse PDF, analyze image, etc.
console.log(`Loaded ${data.size} bytes of ${data.contentType}`);
```

## Working with Inline Images

An HTML body refers to an embedded image as `<img src="cid:unique-image-id@localhost">`, and the matching attachment carries `contentId: "<unique-image-id@localhost>"` and `embedded: true`.

If you are going to display the HTML, let EmailEngine do the substitution: `GET /v1/account/{account}/message/{message}?webSafeHtml=true` returns sanitized HTML with every `cid:` reference replaced by a data URI, and `embedAttachedImages=true` does the substitution without the sanitizing. See [Web-safe HTML](/docs/receiving/web-safe-html). The rest of this section is for the case where you want the images as files.

### Identify Inline Images

```javascript
function getInlineImages(message) {
  if (!message.attachments) return [];

  return message.attachments.filter(att => att.embedded);
}

const inlineImages = getInlineImages(message);
console.log(`Message has ${inlineImages.length} inline images`);
```

### Download Inline Images

```javascript
async function downloadInlineImages(accountId, messageId, outputDir) {
  const message = await getMessage(accountId, messageId);
  const inlineImages = getInlineImages(message);

  if (inlineImages.length === 0) {
    return [];
  }

  if (!fs.existsSync(outputDir)) {
    fs.mkdirSync(outputDir, { recursive: true });
  }

  const downloads = [];

  for (const image of inlineImages) {
    // Use Content-ID as filename if no filename provided
    let filename = image.filename;

    if (!filename) {
      const ext = image.contentType.split('/')[1] || 'bin';
      filename = `inline-${image.contentId || image.id}.${ext}`;
    }

    const safeName = filename.replace(/[^a-zA-Z0-9.-]/g, '_');
    const outputPath = `${outputDir}/${safeName}`;

    try {
      await downloadAttachment(accountId, image.id, outputPath);

      downloads.push({
        path: outputPath,
        contentId: image.contentId,
        contentType: image.contentType
      });
    } catch (err) {
      console.error(`Failed to download inline image:`, err);
    }
  }

  return downloads;
}
```

### Replace CID References in HTML

Convert HTML with inline images to use local files. The `contentId` value includes the angle brackets and the `cid:` reference does not, so strip them before matching:

```javascript
function convertHtmlWithInlineImages(message, imageDir) {
  // message.text.html is only present when the message was fetched
  // with ?textType=* or ?textType=html
  let html = message.text && message.text.html ? message.text.html : '';

  if (!html) return html;

  for (const image of getInlineImages(message)) {
    if (!image.contentId) continue;

    const cid = image.contentId.replace(/^<|>$/g, '');
    const escaped = cid.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
    const cidPattern = new RegExp(`cid:${escaped}`, 'gi');

    const ext = image.contentType.split('/')[1] || 'bin';
    const filename = `inline-${cid}.${ext}`.replace(/[^a-zA-Z0-9.-]/g, '_');

    html = html.replace(cidPattern, `${imageDir}/${filename}`);
  }

  return html;
}

// Usage
// Fetch the message with ?textType=* so text.html is included in the response
const message = await fetch(
  `${BASE_URL}/v1/account/example/message/AAAAAQAAAeE?textType=*`,
  { headers: HEADERS }
).then(r => r.json());

await downloadInlineImages('example', message.id, './images');
const convertedHtml = convertHtmlWithInlineImages(message, './images');

// Save HTML file
fs.writeFileSync('./message.html', convertedHtml);
```

## Common Patterns

### Save Attachments by Type

```javascript
async function saveAttachmentsByType(accountId, messageId, baseDir) {
  const message = await getMessage(accountId, messageId);

  const typeMap = {
    'application/pdf': 'pdfs',
    'image/': 'images',
    'application/vnd.openxmlformats-officedocument': 'documents',
    'application/vnd.ms-': 'documents',
    'text/': 'text',
    'application/zip': 'archives'
  };

  for (const attachment of message.attachments || []) {
    if (attachment.embedded) continue;

    // Determine type directory
    let typeDir = 'other';
    for (const [pattern, dir] of Object.entries(typeMap)) {
      if (attachment.contentType.includes(pattern)) {
        typeDir = dir;
        break;
      }
    }

    const outputDir = `${baseDir}/${typeDir}`;
    if (!fs.existsSync(outputDir)) {
      fs.mkdirSync(outputDir, { recursive: true });
    }

    const filename = attachment.filename || `attachment-${attachment.id}`;
    const safeName = filename.replace(/[^a-zA-Z0-9.-]/g, '_');
    const outputPath = `${outputDir}/${safeName}`;

    await downloadAttachment(accountId, attachment.id, outputPath);
    console.log(`Saved to ${outputPath}`);
  }
}

// Organize attachments by type
await saveAttachmentsByType('example', 'AAAAAQAAAeE', './organized');
```

### Process Large Attachments

Handle large attachments with streaming instead of buffering the whole file. `fetch` in Node.js returns a web `ReadableStream`, so convert it with `Readable.fromWeb` before piping it into a file:

```javascript
const { Readable } = require('stream');
const { pipeline } = require('stream/promises');

async function downloadLargeAttachment(accountId, attachmentId, outputPath) {
  const response = await fetch(
    `${BASE_URL}/v1/account/${accountId}/attachment/${attachmentId}`,
    { headers: HEADERS }
  );

  if (!response.ok) {
    throw new Error(`Download failed: ${response.status}`);
  }

  await pipeline(Readable.fromWeb(response.body), fs.createWriteStream(outputPath));

  return {
    path: outputPath,
    size: fs.statSync(outputPath).size
  };
}
```

### Extract Attachment Metadata

The listing endpoint already includes the `attachments` array for every message, so this needs one request per page rather than one per message:

```javascript
async function extractAttachmentMetadata(accountId, folderPath) {
  const metadata = [];
  const pageSize = 100;

  for (let page = 0; ; page++) {
    const url = new URL(`${BASE_URL}/v1/account/${accountId}/messages`);
    url.searchParams.set('path', folderPath);
    url.searchParams.set('page', page);
    url.searchParams.set('pageSize', pageSize);

    const response = await fetch(url, { headers: HEADERS });
    if (!response.ok) {
      throw new Error(`Listing failed: ${response.status}`);
    }

    const listing = await response.json();

    for (const message of listing.messages) {
      for (const attachment of message.attachments || []) {
        if (attachment.embedded) continue;

        metadata.push({
          messageId: message.id,
          messageSubject: message.subject,
          messageDate: message.date,
          messageFrom: message.from && message.from.address,
          attachmentId: attachment.id,
          filename: attachment.filename,
          contentType: attachment.contentType,
          encodedSize: attachment.encodedSize
        });
      }
    }

    if (page + 1 >= listing.pages) break;
  }

  return metadata;
}

// Extract metadata for all attachments in INBOX
const metadata = await extractAttachmentMetadata('example', 'INBOX');

// Export to CSV
const csv = [
  ['Message Subject', 'From', 'Date', 'Filename', 'Type', 'Encoded Size'],
  ...metadata.map(m => [
    m.messageSubject,
    m.messageFrom,
    m.messageDate,
    m.filename,
    m.contentType,
    m.encodedSize
  ])
].map(row => row.join(',')).join('\n');

fs.writeFileSync('./attachments.csv', csv);
```

### Virus Scanning

Scan a downloaded file before keeping it. This example uses the `clamscan` package against a local ClamAV installation:

```javascript
const NodeClam = require('clamscan');

async function downloadWithVirusScan(accountId, attachmentId, outputPath) {
  // Download to temporary location
  const tempPath = `${outputPath}.tmp`;

  await downloadAttachment(accountId, attachmentId, tempPath);

  // Scan with ClamAV
  const clamscan = await new NodeClam().init();
  const { isInfected, viruses } = await clamscan.scanFile(tempPath);

  if (isInfected) {
    // Delete infected file
    fs.unlinkSync(tempPath);
    throw new Error(`Virus detected: ${viruses.join(', ')}`);
  }

  // Move to final location
  fs.renameSync(tempPath, outputPath);

  return { path: outputPath, safe: true };
}

try {
  await downloadWithVirusScan('example', 'AAAAAgAAAeEBAAAAAQAAAeE', './file.pdf');
  console.log('File is safe');
} catch (err) {
  console.error('Security issue:', err.message);
}
```

## Handling Large Attachments

EmailEngine does not cap attachment downloads. If your own pipeline has a limit, `encodedSize` is known before any download starts, so check it first.

### Check Size Before Downloading

```javascript
async function downloadIfSmallEnough(accountId, attachment, maxSize, outputPath) {
  // encodedSize is the base64-encoded size; the decoded file is roughly 75% of it
  if (attachment.encodedSize > maxSize) {
    console.log(`Skipping ${attachment.filename}: too large (${attachment.encodedSize} > ${maxSize})`);
    return null;
  }

  return await downloadAttachment(accountId, attachment.id, outputPath);
}

// Download only attachments under 10MB
const MAX_SIZE = 10 * 1024 * 1024;

const message = await getMessage('example', 'AAAAAQAAAeE');

for (const attachment of message.attachments || []) {
  await downloadIfSmallEnough(
    'example',
    attachment,
    MAX_SIZE,
    `./downloads/${attachment.filename || attachment.id}`
  );
}
```

### Download with Progress

Track download progress for large files. The response is streamed, so it carries no `Content-Length`; use the decoded size estimated from `encodedSize` as the total:

```javascript
async function downloadWithProgress(accountId, attachment, outputPath, onProgress) {
  const response = await fetch(
    `${BASE_URL}/v1/account/${accountId}/attachment/${attachment.id}`,
    { headers: HEADERS }
  );

  if (!response.ok) {
    throw new Error(`Download failed: ${response.status}`);
  }

  const estimatedTotal = Math.round(attachment.encodedSize * 0.75);
  let downloadedSize = 0;

  const fileStream = fs.createWriteStream(outputPath);

  for await (const chunk of Readable.fromWeb(response.body)) {
    downloadedSize += chunk.length;
    if (onProgress) {
      onProgress(downloadedSize, estimatedTotal);
    }
    if (!fileStream.write(chunk)) {
      await new Promise(resolve => fileStream.once('drain', resolve));
    }
  }

  await new Promise((resolve, reject) => fileStream.end(err => (err ? reject(err) : resolve())));

  return { path: outputPath, size: downloadedSize };
}

// Usage
const message = await getMessage('example', 'AAAAAQAAAeE');
const largest = message.attachments.reduce((a, b) => (b.encodedSize > a.encodedSize ? b : a));

await downloadWithProgress('example', largest, './large-file.zip', (downloaded, total) => {
  console.log(`Downloaded ${downloaded} of about ${total} bytes`);
});
```

## Attachments in Webhook Payloads

The `messageNew` webhook carries the same `attachments` array as the API, metadata only, unless you turn content inclusion on:

| Setting | Default | Effect |
|---------|---------|--------|
| `notifyAttachments` | `false` | Add a base64 `content` field to every attachment in the payload |
| `notifyAttachmentSize` | unset | With `notifyAttachments` on, skip the content of any attachment whose `encodedSize` is above this many bytes. Unset means no limit |

Inline images are a separate rule: when the payload includes the HTML body, every attachment the HTML references by `cid:` gets its `content` filled in as well, up to 2 MB encoded per attachment, whether or not `notifyAttachments` is set. That is what lets a webhook consumer render the body without a second request.

Anything not included stays reachable by `id` through the download endpoint. The [messageNew reference](/docs/webhooks/messagenew#attachment-options) documents the payload and the related text options.

## Utility Functions

### Format Bytes

```javascript
function formatBytes(bytes, decimals = 2) {
  if (bytes === 0) return '0 Bytes';

  const k = 1024;
  const sizes = ['Bytes', 'KB', 'MB', 'GB'];
  const i = Math.floor(Math.log(bytes) / Math.log(k));

  return parseFloat((bytes / Math.pow(k, i)).toFixed(decimals)) + ' ' + sizes[i];
}

console.log(formatBytes(1234567)); // "1.18 MB"
```

### Get File Extension

```javascript
function getFileExtension(contentType) {
  const mimeMap = {
    'application/pdf': 'pdf',
    'image/png': 'png',
    'image/jpeg': 'jpg',
    'image/gif': 'gif',
    'application/zip': 'zip',
    'text/plain': 'txt',
    'text/html': 'html',
    'application/vnd.openxmlformats-officedocument.wordprocessingml.document': 'docx',
    'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet': 'xlsx'
  };

  return mimeMap[contentType] || 'bin';
}
```

## See Also

- [Message operations](/docs/receiving/message-operations) - Where the attachment list comes from
- [Web-safe HTML](/docs/receiving/web-safe-html) - Inline images, and when to leave them out
- [messageNew webhook](/docs/webhooks/messagenew#attachment-options) - Attachment content in webhook payloads
- [Basic sending](/docs/sending/basic-sending) - Attaching files to an outgoing message
- [Attachment API](/docs/api/get-v-1-account-account-attachment-attachment) - The endpoint reference
