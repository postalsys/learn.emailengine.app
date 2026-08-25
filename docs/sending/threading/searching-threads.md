---
title: Searching Thread Messages
sidebar_position: 3
description: How to find and retrieve all messages in an email thread
---

# Searching Thread Messages

Once you have a thread ID, you can search for all related messages across folders. The approach varies by email provider.

## Using the `\All` Folder

Gmail and Microsoft Graph API support a special `\All` folder that searches across all mailboxes simultaneously.

### Supported Providers

- Gmail (IMAP + OAuth2) - Supported
- Gmail (Gmail API) - Supported
- Microsoft Graph API - Supported
- Outlook/Microsoft 365 (IMAP + OAuth2) - Not supported
- Yahoo/AOL/Verizon - Not supported
- Generic IMAP - Not supported

### Search All Folders at Once

**Gmail Example** using the [Search Messages API endpoint](/docs/api/post-v-1-account-account-search):

```bash
curl -XPOST "https://emailengine.example.com/v1/account/gmail/search?path=%5CAll" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "search": {
      "threadId": "1759349012996310407"
    }
  }'
```

**Response**:

```json
{
  "total": 5,
  "page": 0,
  "pages": 1,
  "nextPageCursor": null,
  "prevPageCursor": null,
  "messages": [
    {
      "id": "AAAAKAAACKO",
      "uid": 2213,
      "threadId": "1759349012996310407",
      "subject": "Re: Project discussion",
      "from": { "name": "Colleague", "address": "colleague@example.com" },
      "date": "2025-10-10T16:20:00.000Z"
    },
    {
      "id": "AAAAKAAACKN",
      "uid": 445,
      "threadId": "1759349012996310407",
      "subject": "Re: Project discussion",
      "from": { "name": "Me", "address": "me@example.com" },
      "date": "2025-10-10T15:45:00.000Z"
    },
    {
      "id": "AAAAKAAACKM",
      "uid": 2211,
      "threadId": "1759349012996310407",
      "subject": "Project discussion",
      "from": { "name": "Colleague", "address": "colleague@example.com" },
      "date": "2025-10-10T14:30:00.000Z"
    }
  ]
}
```

Results are returned newest first.

**Microsoft Graph API Example**:

```bash
curl -XPOST "https://emailengine.example.com/v1/account/outlook-graph/search?path=%5CAll" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "search": {
      "threadId": "AAQkAGI2TH..."
    }
  }'
```

### Benefits of `\All` Folder

- **Single Request**: Get entire thread in one API call
- **Complete View**: Includes messages from Inbox, Sent, and all other folders
- **Efficient**: Faster than multiple folder queries
- **Sorted**: Messages are returned newest first - sort client-side if you need chronological order

## Folder-by-Folder Search

For providers without `\All` support, you must search each folder individually.

### Providers Requiring This Approach

- Yahoo/AOL/Verizon (OBJECTID support but no `\All`)
- Outlook/Microsoft 365 (IMAP backend)
- Generic IMAP providers

### Search Multiple Folders

**Step 1: Search Inbox**:

```bash
curl -XPOST "https://emailengine.example.com/v1/account/yahoo/search?path=INBOX" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "search": {
      "threadId": "501"
    }
  }'
```

**Step 2: Search Sent**:

```bash
curl -XPOST "https://emailengine.example.com/v1/account/yahoo/search?path=Sent" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "search": {
      "threadId": "501"
    }
  }'
```

**Step 3: Combine Results**:

```javascript
async function getCompleteThread(account, threadId) {
  const folders = ['INBOX', '\\Sent', '\\Archive', '\\Drafts'];
  const path = `https://emailengine.example.com/v1/account/${encodeURIComponent(account)}/search`;

  const perFolder = await Promise.all(
    folders.map(folder =>
      fetch(`${path}?path=${encodeURIComponent(folder)}`, {
        method: 'POST',
        headers: { Authorization: `Bearer ${token}`, 'Content-Type': 'application/json' },
        body: JSON.stringify({ search: { threadId } })
      })
        .then(res => res.json())
        .then(data => data.messages || [])
    )
  );

  return perFolder
    .flat()
    .filter((msg, i, all) => all.findIndex(m => m.id === msg.id) === i)
    .sort((a, b) => new Date(a.date) - new Date(b.date));
}
```

Searching the folders concurrently rather than one after another keeps this close to the latency of the slowest single folder. Referring to folders by their special-use flags (`\Sent`, `\Archive`) rather than by name avoids breaking on localized mailboxes.

### Limitations

- **Multiple Requests**: Slower than `\All` folder
- **Incomplete Results**: May miss messages moved to unexpected folders
- **Complexity**: Client-side merging and deduplication needed

## Generic IMAP Threading

For generic IMAP providers without native threading support, threads must be built manually from Message-ID headers. See the "Building Threads Manually" section below for implementation details.

## Building Threads Manually

Without native threading support, you can build threads from Message-ID headers.

### Extract Thread from Headers

A thread is the transitive closure over `Message-ID`, `In-Reply-To`, and `References`. Rather than walking parents and children separately, index every message by its `Message-ID` once, then expand outwards from the starting message:

```javascript
// Every Message-ID this message is connected to, in either direction
function linkedIds(msg) {
  const refs = (msg.headers?.references || []).join(' ');
  const inReplyTo = (msg.headers?.['in-reply-to'] || []).join(' ');

  return [...`${refs} ${inReplyTo}`.matchAll(/<([^>]+)>/g)].map(m => m[1]);
}

function buildThread(startId, allMessages) {
  const byMessageId = new Map(allMessages.map(m => [m.messageId?.replace(/^<|>$/g, ''), m]));

  const thread = new Map();
  const pending = [byMessageId.get(startId)].filter(Boolean);

  while (pending.length) {
    const msg = pending.pop();
    if (thread.has(msg.id)) continue;
    thread.set(msg.id, msg);

    // Messages this one points at
    for (const id of linkedIds(msg)) {
      const parent = byMessageId.get(id);
      if (parent) pending.push(parent);
    }

    // Messages that point at this one
    const own = msg.messageId?.replace(/^<|>$/g, '');
    for (const other of allMessages) {
      if (own && linkedIds(other).includes(own)) pending.push(other);
    }
  }

  return [...thread.values()].sort((a, b) => new Date(a.date) - new Date(b.date));
}
```

Both header fields hold angle-bracketed IDs, and EmailEngine returns raw header values as arrays, so each one is joined before the IDs are extracted. Request the headers explicitly when searching, since they are not part of the default message listing.

### Limitations

- Must fetch all messages from all folders first
- Computationally expensive for large mailboxes
- May miss messages if headers are malformed
- Doesn't provide persistent thread IDs

## Search Strategy by Provider

Which approach applies comes down to one question: does the account expose an `\All` folder?

| Provider | Strategy | Search by |
|----------|----------|-----------|
| Gmail (IMAP and API) | One search against `\All` | `threadId` |
| Microsoft 365 / Outlook (Graph) | One search against `\All` | `threadId` |
| Yahoo, AOL, Verizon | One search per folder | `threadId` |
| Generic IMAP | One search per folder, then thread on headers | `subject`, then `References` |

A single function covers all four, because the only variables are the folders to search and whether the results need merging:

```javascript
const BASE = 'https://emailengine.example.com';
const headers = {
  Authorization: `Bearer ${token}`,
  'Content-Type': 'application/json'
};

async function searchThread(account, search, folders = ['\\All']) {
  const path = `${BASE}/v1/account/${encodeURIComponent(account)}/search`;

  const results = await Promise.all(
    folders.map(folder =>
      fetch(`${path}?path=${encodeURIComponent(folder)}`, {
        method: 'POST',
        headers,
        body: JSON.stringify({ search })
      })
        .then(res => res.json())
        .then(data => data.messages || [])
    )
  );

  // One \All search returns an already complete thread; per-folder searches need merging
  return results
    .flat()
    .filter((msg, i, all) => all.findIndex(m => m.id === msg.id) === i)
    .sort((a, b) => new Date(a.date) - new Date(b.date));
}

// Gmail or Microsoft Graph
await searchThread('user@gmail.com', { threadId: '501' });

// Yahoo and other providers without \All
await searchThread('user@yahoo.com', { threadId: '501' }, ['INBOX', '\\Sent']);
```

Deduplicating on `id` matters for the per-folder path: a message can legitimately appear in more than one folder, and Gmail's labels make that the norm rather than the exception.

For generic IMAP, search by `subject` first and then group on the `References` and `In-Reply-To` headers, as described in [Building Threads Manually](#building-threads-manually) above.

## Pagination

Thread searches support pagination for long threads:

```bash
curl -XPOST "https://emailengine.example.com/v1/account/gmail/search?path=%5CAll&page=0&pageSize=50" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "search": {
      "threadId": "1759349012996310407"
    }
  }'
```

**Response includes pagination metadata**:

```json
{
  "total": 127,
  "page": 0,
  "pages": 3,
  "messages": []
}
```

## See Also

- [Provider support](/docs/sending/threading/provider-support) - Which providers support the `\All` folder
- [Searching messages](/docs/receiving/searching) - The full search term reference
- [Threading overview](/docs/sending/threading/overview) - Where a `threadId` comes from
- [Messages API](/docs/api-reference/messages-api) - Paging through a large result set
