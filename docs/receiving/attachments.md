---
title: Working with Attachments
sidebar_position: 7
description: "Complete guide to handling email attachments - downloading, uploading, inline images, and file management"
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

Attachments are listed with the message and fetched on demand. This page covers [downloading them](/docs/api/get-v-1-account-account-attachment-attachment), the inline images an HTML body references, and the metadata that comes with each one.

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

**id** - Unique attachment identifier (use for downloading)
**contentType** - MIME type of the file
**filename** - Original filename (may be null)
**encodedSize** - Size in email (base64 encoded, actual file size will be smaller)
**embedded** - True if the attachment is embedded in HTML content
**inline** - True if the attachment should be displayed inline rather than as a download
**encodedInMessage** - True if the attachment is part of an enclosed message/rfc822 attachment rather than the top-level message itself
**contentId** - Content-ID header value for embedding images in HTML
**method** - Calendar method (REQUEST, REPLY, CANCEL, etc.) for iCalendar attachments

## Listing Attachments

### Get Message with Attachments

Fetch a message to see its attachments:

```javascript
async function getMessageAttachments(accountId, messageId) {
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/message/${messageId}`,
    {
      headers: { 'Authorization': 'Bearer YOUR_ACCESS_TOKEN' }
    }
  );

  const message = await response.json();

  return {
    messageId: message.id,
    subject: message.subject,
    attachments: message.attachments || [],
    hasAttachments: message.attachments && message.attachments.length > 0
  };
}

const info = await getMessageAttachments('example', 'AAAAAQAAAeE');
console.log(`Message: ${info.subject}`);
console.log(`Attachments: ${info.attachments.length}`);

info.attachments.forEach(att => {
  console.log(`- ${att.filename} (${formatBytes(att.encodedSize)})`);
});
```

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

<Tabs groupId="programming-language">
<TabItem value="nodejs" label="Node.js">

```javascript
const fs = require('fs');

async function downloadAttachment(accountId, messageId, attachmentId, outputPath) {
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/attachment/${attachmentId}`,
    {
      headers: { 'Authorization': 'Bearer YOUR_ACCESS_TOKEN' }
    }
  );

  if (!response.ok) {
    throw new Error(`Download failed: ${response.statusText}`);
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
  'AAAAAQAAAeE',
  'AAAAAgAAAeEBAAAAAQAAAeE',
  './downloads/invoice.pdf'
);

console.log(`Downloaded to ${result.path} (${result.size} bytes)`);
```

</TabItem>
<TabItem value="python" label="Python">

```python
import requests

def download_attachment(account_id, message_id, attachment_id, output_path):
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
    'AAAAAQAAAeE',
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
      const result = await downloadAttachment(
        accountId,
        messageId,
        attachment.id,
        outputPath
      );

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
async function downloadToMemory(accountId, messageId, attachmentId) {
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/attachment/${attachmentId}`,
    {
      headers: { 'Authorization': 'Bearer YOUR_ACCESS_TOKEN' }
    }
  );

  if (!response.ok) {
    throw new Error(`Download failed: ${response.statusText}`);
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
const data = await downloadToMemory('example', 'AAAAAQAAAeE', 'AAAAAgAAAeEBAAAAAQAAAeE');

// Parse PDF, analyze image, etc.
console.log(`Loaded ${data.size} bytes of ${data.contentType}`);
```

## Working with Inline Images

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
      await downloadAttachment(accountId, messageId, image.id, outputPath);

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

Convert HTML with inline images to use local files:

```javascript
async function convertHtmlWithInlineImages(message, imageDir) {
  // message.text.html is a string containing the HTML content
  // (only present when the message was fetched with ?textType=* or ?textType=html)
  let html = message.text && message.text.html ? message.text.html : '';

  if (!html) return html;

  const inlineImages = getInlineImages(message);

  for (const image of inlineImages) {
    if (image.contentId) {
      // Find cid: references
      const cidPattern = new RegExp(`cid:${image.contentId}`, 'gi');

      // Generate filename
      const ext = image.contentType.split('/')[1] || 'bin';
      const filename = `inline-${image.contentId}.${ext}`;

      // Replace with local path
      html = html.replace(cidPattern, `${imageDir}/${filename}`);
    }
  }

  return html;
}

// Usage
// Fetch the message with ?textType=* so text.html is included in the response
const message = await fetch(
  `https://emailengine.example.com/v1/account/example/message/AAAAAQAAAeE?textType=*`,
  { headers: { 'Authorization': 'Bearer YOUR_ACCESS_TOKEN' } }
).then(r => r.json());

await downloadInlineImages('example', message.id, './images');
const convertedHtml = await convertHtmlWithInlineImages(message, './images');

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

    await downloadAttachment(accountId, messageId, attachment.id, outputPath);
    console.log(`Saved to ${outputPath}`);
  }
}

// Organize attachments by type
await saveAttachmentsByType('example', 'AAAAAQAAAeE', './organized');
```

### Process Large Attachments

Handle large attachments with streaming:

```javascript
async function downloadLargeAttachment(accountId, messageId, attachmentId, outputPath) {
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/attachment/${attachmentId}`,
    {
      headers: { 'Authorization': 'Bearer YOUR_ACCESS_TOKEN' }
    }
  );

  if (!response.ok) {
    throw new Error(`Download failed: ${response.statusText}`);
  }

  // Stream to file
  const fileStream = fs.createWriteStream(outputPath);

  return new Promise((resolve, reject) => {
    response.body.pipe(fileStream);

    response.body.on('error', reject);
    fileStream.on('finish', () => {
      resolve({
        path: outputPath,
        size: fs.statSync(outputPath).size
      });
    });
    fileStream.on('error', reject);
  });
}
```

### Extract Attachment Metadata

```javascript
async function extractAttachmentMetadata(accountId, folderPath) {
  const messages = await listAllMessages(accountId, folderPath);

  const metadata = [];

  for (const message of messages) {
    if (!message.attachments || !message.attachments.length) continue;

    const fullMessage = await getMessage(accountId, message.id);

    for (const attachment of fullMessage.attachments || []) {
      if (attachment.embedded) continue;

      metadata.push({
        messageId: message.id,
        messageSubject: message.subject,
        messageDate: message.date,
        messageFrom: message.from.address,
        attachmentId: attachment.id,
        filename: attachment.filename,
        contentType: attachment.contentType,
        encodedSize: attachment.encodedSize
      });
    }
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

Scan attachments before downloading:

```javascript
const ClamScan = require('clamscan');

async function downloadWithVirusScan(accountId, messageId, attachmentId, outputPath) {
  // Download to temporary location
  const tempPath = `${outputPath}.tmp`;

  await downloadAttachment(accountId, messageId, attachmentId, tempPath);

  // Scan with ClamAV
  const clamscan = await new ClamScan().init();
  const { isInfected, viruses } = await clamscan.isInfected(tempPath);

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
  await downloadWithVirusScan('example', 'AAAAAQAAAeE', 'AAAAAgAAAeEBAAAAAQAAAeE', './file.pdf');
  console.log('File is safe');
} catch (err) {
  console.error('Security issue:', err.message);
}
```

## Handling Attachment Size Limits

### Check Size Before Downloading

```javascript
async function downloadIfSmallEnough(accountId, messageId, attachment, maxSize, outputPath) {
  // Note: encodedSize is the base64 encoded size; actual file will be smaller
  if (attachment.encodedSize > maxSize) {
    console.log(`Skipping ${attachment.filename}: too large (${attachment.encodedSize} > ${maxSize})`);
    return null;
  }

  return await downloadAttachment(accountId, messageId, attachment.id, outputPath);
}

// Download only attachments under 10MB
const MAX_SIZE = 10 * 1024 * 1024;

const message = await getMessage('example', 'AAAAAQAAAeE');

for (const attachment of message.attachments || []) {
  await downloadIfSmallEnough(
    'example',
    message.id,
    attachment,
    MAX_SIZE,
    `./downloads/${attachment.filename}`
  );
}
```

### Download with Progress

Track download progress for large files:

```javascript
async function downloadWithProgress(accountId, messageId, attachmentId, outputPath, onProgress) {
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/attachment/${attachmentId}`,
    {
      headers: { 'Authorization': 'Bearer YOUR_ACCESS_TOKEN' }
    }
  );

  if (!response.ok) {
    throw new Error(`Download failed: ${response.statusText}`);
  }

  const totalSize = parseInt(response.headers.get('content-length'), 10);
  let downloadedSize = 0;

  const fileStream = fs.createWriteStream(outputPath);

  response.body.on('data', (chunk) => {
    downloadedSize += chunk.length;
    const progress = (downloadedSize / totalSize) * 100;

    if (onProgress) {
      onProgress(downloadedSize, totalSize, progress);
    }
  });

  return new Promise((resolve, reject) => {
    response.body.pipe(fileStream);
    response.body.on('error', reject);
    fileStream.on('finish', () => resolve({ path: outputPath, size: downloadedSize }));
    fileStream.on('error', reject);
  });
}

// Usage
await downloadWithProgress(
  'example',
  'AAAAAQAAAeE',
  'AAAAAgAAAeEBAAAAAQAAAeE',
  './large-file.zip',
  (downloaded, total, progress) => {
    console.log(`Progress: ${progress.toFixed(1)}% (${downloaded}/${total})`);
  }
);
```

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
- [Basic sending](/docs/sending/basic-sending) - Attaching files to an outgoing message
- [Attachment API](/docs/api/get-v-1-account-account-attachment-attachment) - The endpoint reference
