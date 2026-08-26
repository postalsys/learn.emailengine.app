---
title: Searching Thread Messages
sidebar_position: 3
description: How to find and retrieve all messages in an email thread
---

# Searching Thread Messages

Once you have a thread ID, you can search for every message that carries it. Whether that takes one request or one per folder depends on the backend.

## Using the `\All` Folder

Gmail and the Microsoft Graph API expose a virtual `\All` folder that covers every message in the account, so one search against it returns the whole thread.

### Supported Providers

- Gmail over IMAP - Supported (the `[Gmail]/All Mail` folder carries the `\All` special-use flag)
- Gmail API - Supported
- Microsoft Graph API - Supported
- Microsoft 365 over IMAP - Not supported
- Yahoo, AOL, Verizon - Not supported
- Other IMAP servers - Not supported

### Search All Folders at Once

**Gmail example** using the [Search messages endpoint](/docs/api/post-v-1-account-account-search):

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
  "total": 3,
  "page": 0,
  "pages": 1,
  "nextPageCursor": null,
  "prevPageCursor": null,
  "messages": [
    {
      "id": "AAAAKAAACKO",
      "path": "INBOX",
      "threadId": "1759349012996310407",
      "subject": "Re: Project discussion",
      "from": { "name": "Colleague", "address": "colleague@example.com" },
      "date": "2025-10-10T16:20:00.000Z"
    },
    {
      "id": "AAAAKAAACKN",
      "path": "[Gmail]/Sent Mail",
      "threadId": "1759349012996310407",
      "subject": "Re: Project discussion",
      "from": { "name": "Me", "address": "me@example.com" },
      "date": "2025-10-10T15:45:00.000Z"
    },
    {
      "id": "AAAAKAAACKM",
      "path": "INBOX",
      "threadId": "1759349012996310407",
      "subject": "Project discussion",
      "from": { "name": "Colleague", "address": "colleague@example.com" },
      "date": "2025-10-10T14:30:00.000Z"
    }
  ]
}
```

Results are returned newest first. Sort client-side if you need chronological order. A `\All` listing carries `path` on every entry, which is how you tell which folder a message actually sits in.

**Microsoft Graph API example**:

```bash
curl -XPOST "https://emailengine.example.com/v1/account/outlook-graph/search?path=%5CAll" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "search": {
      "threadId": "AAQkAGI2THY2ZjRhLTVjNzgtNDMxYS05YTBmLTJiN2M4ZDkxZTQyMwAQAF3xTx0nRUxOhKcvLZQ9r1M="
    }
  }'
```

On the Gmail API the search resolves the thread through the API's own thread listing; on the Graph API it filters on `conversationId`. Either way one request returns messages from the Inbox, Sent Mail, and every other folder.

## Folder-by-Folder Search

Where there is no `\All` folder, search each folder in turn.

### Providers Requiring This Approach

- Yahoo, AOL, Verizon (OBJECTID support, no `\All` folder)
- Microsoft 365 over IMAP
- Other IMAP servers

### Search Multiple Folders

**Step 1: Search the Inbox**:

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

**Step 2: Search Sent Mail**:

```bash
curl -XPOST "https://emailengine.example.com/v1/account/yahoo/search?path=%5CSent" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "search": {
      "threadId": "501"
    }
  }'
```

**Step 3: Combine the results**:

```javascript
const token = process.env.EE_TOKEN;

async function getCompleteThread(account, threadId) {
  const folders = ['INBOX', '\\Sent', '\\Archive'];
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

Searching the folders concurrently keeps the total close to the latency of the slowest single folder. Referring to folders by their special-use flags (`\Sent`, `\Archive`) rather than by name avoids breaking on localized mailbox names.

### Limitations

- **Multiple requests**: slower than one `\All` search
- **Incomplete results**: a message moved to a folder you did not search is missed
- **Client-side merging**: results need deduplication, because the same message can appear in more than one folder

## Building Threads Manually

When the server assigns no `threadId`, a thread is the transitive closure over `Message-ID`, `In-Reply-To`, and `References`.

### What the API Gives You

- Listing and search entries carry `messageId` and `inReplyTo`
- The full `References` chain is only in the message detail response, under `headers.references`, so fetch each candidate with `GET /v1/account/{account}/message/{message}` when `inReplyTo` alone is not enough

Header values in the detail response are arrays, and both fields hold angle-bracketed IDs, so each one is joined before the IDs are extracted.

### Extract a Thread from Headers

Index every message by its `Message-ID` once, then expand outwards from the starting message in both directions:

```javascript
// Every Message-ID this message is connected to, in either direction
function linkedIds(msg) {
  const refs = (msg.headers?.references || []).join(' ');
  const inReplyTo = msg.inReplyTo || (msg.headers?.['in-reply-to'] || []).join(' ');

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

### Limitations

- Every candidate message has to be fetched from every folder first
- Expensive on large mailboxes
- A message with malformed or missing headers is left out
- The result is not a persistent ID; it has to be rebuilt each time

## Search Strategy by Provider

Which approach applies comes down to one question: does the account expose an `\All` folder?

| Provider | Strategy | Search by |
|----------|----------|-----------|
| Gmail (IMAP and API) | One search against `\All` | `threadId` |
| Microsoft 365 / Outlook (Graph) | One search against `\All` | `threadId` |
| Yahoo, AOL, Verizon | One search per folder | `threadId` |
| Other IMAP servers | One search per folder, then thread on headers | `subject`, then `References` |

One function covers all four cases, because the only variables are the folders to search and whether the results need merging:

```javascript
const BASE = 'https://emailengine.example.com';
const headers = {
  Authorization: `Bearer ${process.env.EE_TOKEN}`,
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
await searchThread('user@gmail.com', { threadId: '1759349012996310407' });

// Yahoo and other providers without \All
await searchThread('user@yahoo.com', { threadId: '501' }, ['INBOX', '\\Sent']);
```

Deduplicating on `id` matters for the per-folder path: a message can legitimately appear in more than one folder, and Gmail's labels make that the norm rather than the exception.

For an IMAP server without thread IDs, search by `subject` first and then group on `inReplyTo` and the `References` header, as described in [Building threads manually](#building-threads-manually) above.

## Pagination

Thread searches page like any other search:

```bash
curl -XPOST "https://emailengine.example.com/v1/account/gmail/search?path=%5CAll&pageSize=50" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "search": {
      "threadId": "1759349012996310407"
    }
  }'
```

**The response carries the paging metadata**:

```json
{
  "total": 127,
  "page": 0,
  "pages": 3,
  "nextPageCursor": "gmail_kcQIji3UobDDTxc",
  "prevPageCursor": null,
  "messages": []
}
```

The cursor carries the backend it came from as a prefix: `imap_` for IMAP accounts, `gmail_` for the Gmail API, `ms_` for the Graph API. Pass it back as the `cursor` query parameter rather than incrementing `page`, which only IMAP accounts support.

## See Also

- [Provider support](/docs/sending/threading/provider-support) - Which providers support the `\All` folder
- [Searching messages](/docs/receiving/searching) - The full search term reference
- [Threading overview](/docs/sending/threading/overview) - Where a `threadId` comes from
- [Messages API](/docs/api-reference/messages-api) - Paging through a large result set
