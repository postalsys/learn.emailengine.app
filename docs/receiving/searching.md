---
title: Searching Messages
sidebar_position: 6
description: "Complete guide to searching emails with EmailEngine - search queries, operators, filters, and per-backend limits"
keywords:
  - email search
  - search queries
  - IMAP search
  - message filtering
  - search operators
---

# Searching Messages

The [search endpoint](/docs/api/post-v-1-account-account-search) finds messages in a connected account. The search runs on the mail server, so only matching messages are transferred: for IMAP accounts the criteria are translated into an IMAP SEARCH command, for Gmail API accounts into a Gmail query string, and for Microsoft Graph accounts into an OData `$filter` (or `$search`, see below).

The request is a `POST` with the folder in the `path` query parameter and the criteria in a `search` object in the body. The response has the same shape as the [message listing](/docs/receiving/message-operations#listing-messages), newest first, with the same `cursor` paging.

## Basic Search

### Search by Subject

Find messages with specific subject text:

```bash
curl -X POST "https://emailengine.example.com/v1/account/example/search?path=INBOX" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "search": {
      "subject": "meeting"
    }
  }'
```

**JavaScript:**

```javascript
async function search(accountId, folderPath, searchCriteria, extraParams = {}) {
  const params = new URLSearchParams({ path: folderPath, ...extraParams });

  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/search?${params}`,
    {
      method: 'POST',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ search: searchCriteria })
    }
  );

  if (!response.ok) {
    throw new Error(`Search failed: ${response.status}`);
  }

  return await response.json();
}

const results = await search('example', 'INBOX', { subject: 'meeting' });
console.log(`Found ${results.messages.length} messages on this page`);
```

The examples below all go through this `search` helper.

### Search by Sender

Find messages from a specific sender:

```javascript
// Find all emails from john@example.com
const fromJohn = await search('example', 'INBOX', { from: 'john@example.com' });
```

### Search by Date

Find messages from a specific date range:

```javascript
// Find messages from the last week
const today = new Date();
const weekAgo = new Date(today);
weekAgo.setDate(weekAgo.getDate() - 7);

const lastWeek = await search('example', 'INBOX', {
  since: weekAgo.toISOString().split('T')[0],
  before: today.toISOString().split('T')[0]
});
```

Dates are given as `YYYY-MM-DD` or as a full ISO 8601 timestamp.

## Search Operators

All criteria in one `search` object must match (AND). There is no OR; run several searches and merge the results when you need one.

### Text Search Operators

**subject** - Subject contains text
```javascript
{ search: { subject: 'invoice' } }
```

**body** - Message body contains text
```javascript
{ search: { body: 'payment' } }
```

**from** - Sender address or name contains text
```javascript
{ search: { from: 'john@example.com' } }
```

**to**, **cc**, **bcc** - Recipient address contains text (not available on Microsoft Graph accounts in the default mode)
```javascript
{ search: { to: 'jane@company.com' } }
```

**header** - Match a header value
```javascript
{ search: { header: { 'X-Custom-Header': 'value' } } }
```

IMAP accounts accept any header name. Gmail API and Microsoft Graph accounts accept only `Message-ID`; any other header name is rejected with a 400 and the code `UnsupportedSearchTerm`.

### Date Operators

**since** - Messages received on or after date
```javascript
{ search: { since: '2025-01-01' } }
```

**before** - Messages received before date
```javascript
{ search: { before: '2025-12-31' } }
```

**sentSince** - Messages sent on or after date (the Date header)
```javascript
{ search: { sentSince: '2025-01-01' } }
```

**sentBefore** - Messages sent before date (the Date header)
```javascript
{ search: { sentBefore: '2025-12-31' } }
```

Only IMAP servers distinguish received and sent dates. Gmail API and Microsoft Graph accounts treat `sentSince` like `since` and `sentBefore` like `before`.

### Size Operators

**larger** - Messages larger than size (bytes)
```javascript
{ search: { larger: 1000000 } }
```

**smaller** - Messages smaller than size (bytes)
```javascript
{ search: { smaller: 10000 } }
```

Size criteria are applied by IMAP accounts, Gmail IMAP included. Microsoft Graph accounts reject them in the default mode with a 400. Gmail API accounts do not translate them, so they have no effect there; use `gmailRaw` with Gmail's own `larger:` and `smaller:` operators instead.

### Flag Operators

**seen** - Messages that are read
```javascript
{ search: { seen: true } }
```

**unseen** - Messages that are unread
```javascript
{ search: { unseen: true } }
```

**flagged** - Messages that are flagged/starred
```javascript
{ search: { flagged: true } }
```

**answered** - Messages that have been replied to (IMAP only)
```javascript
{ search: { answered: true } }
```

**draft** - Draft messages (IMAP and Microsoft Graph; rejected by Gmail API accounts)
```javascript
{ search: { draft: true } }
```

**deleted** - Messages carrying the `\Deleted` flag but not yet expunged (IMAP only)
```javascript
{ search: { deleted: true } }
```

### UID and ID Operators

**uid** - Specific UID or range (IMAP only)
```javascript
{ search: { uid: '12345' } }
{ search: { uid: '12345:12400' } }
{ search: { uid: '12345:*' } }
```

**seq** - Sequence number range (IMAP only)
```javascript
{ search: { seq: '1:100' } }
```

**modseq** - Messages modified after a modification sequence (IMAP with the CONDSTORE extension only)
```javascript
{ search: { modseq: 12345 } }
```

**emailId** - A single message by its globally unique ID
```javascript
{ search: { emailId: '1743d29c-b67d-4747-9016-b8850a5a39bd' } }
```

**threadId** - Every message in a thread
```javascript
{ search: { threadId: '1743d29c-b67d-4747-9016-b8850a5a39bd' } }
```

**emailIds** - Several messages by their `emailId` values. Only the [bulk endpoints](/docs/receiving/message-operations#bulk-actions) read it, and only on Gmail API and Microsoft Graph accounts, where it replaces every other criterion. The search endpoint and IMAP accounts ignore it
```javascript
{ search: { emailIds: ['id1', 'id2', 'id3'] } }
```

`emailId` and `threadId` need a server that assigns such IDs: Gmail API, Microsoft Graph, Gmail over IMAP, and IMAP servers with the OBJECTID extension (RFC 8474); other IMAP servers fail the search with a 422 and the code `MissingServerExtension`. The values are the `emailId` and `threadId` fields of listed messages; see [Message IDs](/docs/advanced/ids-explained).

### Gmail-Specific Operators

**gmailRaw** - Native Gmail search syntax (Gmail API and Gmail IMAP accounts)
```javascript
{ search: { gmailRaw: 'is:unread category:primary' } }
{ search: { gmailRaw: 'from:boss@company.com has:attachment' } }
```

The string is appended to the query EmailEngine builds from the other criteria and passed to Gmail untouched, as a Gmail API query or an IMAP `X-GM-RAW` search. See [Gmail search operators](https://support.google.com/mail/answer/7190). A generic IMAP server has no such extension, so the search fails there with a 422 and the code `MissingServerExtension`.

### Label and Category Filtering

**labels** - Filter by Gmail labels or Outlook categories (added in v2.69.0)

```javascript
{ search: { labels: { has: ['Important'] } } }
{ search: { labels: { has: ['Work', 'Urgent'] } } }
{ search: { labels: { not: ['Processed'] } } }
{ search: { labels: { has: ['Work'], not: ['Done'] } } }
```

- `has` matches messages that carry **all** of the listed labels or categories
- `not` excludes messages that carry **any** of them
- Gmail API accounts compile the filter to `label:` and `-label:` query terms; Gmail over IMAP uses the server's label search
- Microsoft Graph accounts filter on the message's categories
- A generic IMAP server has no labels, so the search fails with a 422 and the code `MissingServerExtension`. In `useOutlookSearch` mode the filter is rejected with a 400 and the code `UnsupportedSearchTerm`

## Per-Backend Limits

Not every criterion reaches every server. A criterion the backend cannot honor is rejected with a 400 and the code `UnsupportedSearchTerm`; one that needs an IMAP extension the server lacks is rejected with a 422 and the code `MissingServerExtension`. Exceptions are noted.

| Criterion                             | IMAP                     | Gmail API                             | Microsoft Graph (default `$filter`)               |
| ------------------------------------- | ------------------------ | ------------------------------------- | ------------------------------------------------- |
| `subject`, `from`, `body`             | Yes                      | Yes                                   | Yes                                               |
| `to`, `cc`, `bcc`                     | Yes                      | Yes                                   | Rejected; use `useOutlookSearch`                  |
| `since`, `before`                     | Yes                      | Yes                                   | Yes                                               |
| `sentSince`, `sentBefore`             | Yes                      | Treated as `since` / `before`         | Treated as `since` / `before`                     |
| `larger`, `smaller`                   | Yes                      | Ignored                               | Rejected; use `useOutlookSearch`                  |
| `seen`, `unseen`, `flagged`           | Yes                      | Yes                                   | Yes                                               |
| `draft`                               | Yes                      | Rejected                              | Yes                                               |
| `answered`, `deleted`                 | Yes                      | Rejected                              | Rejected                                          |
| `uid`, `seq`, `modseq`                | Yes                      | Rejected                              | Rejected                                          |
| `header`                              | Any header               | `Message-ID` only                     | `Message-ID` only                                 |
| `emailId`, `threadId`                 | With OBJECTID or Gmail (422 elsewhere) | Yes                     | Yes                                               |
| `emailIds`                            | Ignored                  | Bulk endpoints only                   | Bulk endpoints only                               |
| `gmailRaw`                            | Gmail IMAP only (422 elsewhere) | Yes                            | No effect                                         |
| `labels`                              | Gmail IMAP only (422 elsewhere) | Yes                            | Yes                                               |

### Microsoft Graph `$search` mode

Add `useOutlookSearch=true` to the query string (available since v2.48.3) to run the criteria through Graph's `$search` instead of `$filter`. This mode accepts only `to`, `cc`, `bcc`, `larger`, `smaller`, `body`, `before`, `sentBefore`, `since` and `sentSince`. Any other field in the `search` object, `from`, `subject`, the flag criteria and `labels` included, is rejected with a 400 and the code `UnsupportedSearchTerm`, so it is a different search rather than a wider one. The cost:

- At most 1,000 results, ordered by relevance rather than date
- No `total`, no `pages` and no `prevPageCursor` in the response; only `nextPageCursor` pages through the results

Leave it off unless you need one of the fields `$filter` cannot reach.

## Combined Searches

### Multiple Criteria (AND logic)

Combine multiple search criteria - all must match:

```javascript
// Find unread messages from john@example.com
const unreadFromJohn = await search('example', 'INBOX', {
  from: 'john@example.com',
  unseen: true
});
```

### Complex Search Example

Find recent large unread invoices on an IMAP account:

```javascript
const sevenDaysAgo = new Date();
sevenDaysAgo.setDate(sevenDaysAgo.getDate() - 7);

const invoices = await search('example', 'INBOX', {
  subject: 'invoice',
  unseen: true,
  larger: 500000,
  since: sevenDaysAgo.toISOString().split('T')[0]
});

console.log(`Found ${invoices.messages.length} large unread invoices from the last 7 days`);
```

## Common Search Patterns

### Find Messages with Attachments

There is no attachment criterion. List or search the folder and filter on the `attachments` array of each result; on a Gmail API account, `gmailRaw: 'has:attachment'` does it server-side:

```javascript
async function searchWithAttachments(accountId, folderPath, searchCriteria = {}) {
  const data = await search(accountId, folderPath, searchCriteria, { pageSize: 100 });

  return data.messages.filter(msg => msg.attachments && msg.attachments.length > 0);
}
```

### Search by Message ID

Find a message by its Message-ID header. The `Message-ID` header search works on every backend:

```javascript
async function findByMessageId(accountId, messageId) {
  // INBOX and the sent folder cover most messages; \Sent resolves to the real folder
  const folders = ['INBOX', '\\Sent'];

  for (const folder of folders) {
    const data = await search(accountId, folder, {
      header: { 'Message-ID': messageId }
    });

    if (data.messages && data.messages.length > 0) {
      return {
        folder: folder,
        message: data.messages[0]
      };
    }
  }

  return null; // Not found
}

const result = await findByMessageId('example', '<abc123@example.com>');
if (result) {
  console.log(`Found in ${result.folder}: ${result.message.subject}`);
}
```

On Gmail IMAP, Gmail API and Microsoft Graph accounts, searching `path=\All` covers the whole account in one request.

### Search Today's Messages

```javascript
async function searchTodaysMessages(accountId, folderPath) {
  const today = new Date();
  const todayStr = today.toISOString().split('T')[0];
  const tomorrow = new Date(today);
  tomorrow.setDate(tomorrow.getDate() + 1);
  const tomorrowStr = tomorrow.toISOString().split('T')[0];

  return await search(accountId, folderPath, {
    since: todayStr,
    before: tomorrowStr
  });
}
```

### Search This Month

```javascript
async function searchThisMonth(accountId, folderPath) {
  const now = new Date();
  const firstDay = new Date(now.getFullYear(), now.getMonth(), 1);
  const nextMonth = new Date(now.getFullYear(), now.getMonth() + 1, 1);

  return await search(accountId, folderPath, {
    since: firstDay.toISOString().split('T')[0],
    before: nextMonth.toISOString().split('T')[0]
  });
}
```

### Search by Keywords

There is no OR operator, so search once per keyword and merge on the message `id`:

```javascript
async function searchByKeywords(accountId, folderPath, keywords) {
  const results = [];

  for (const keyword of keywords) {
    const data = await search(accountId, folderPath, { body: keyword });
    results.push(...data.messages);
  }

  // Remove duplicates
  return Array.from(new Map(results.map(msg => [msg.id, msg])).values());
}

// Search for messages about invoices or payments
const matches = await searchByKeywords('example', 'INBOX', ['invoice', 'payment', 'bill']);
```

## Advanced Search Techniques

### Search with Pagination

Search results page the same way as listings: follow `nextPageCursor` until it is `null`. IMAP and Microsoft Graph accounts also honor the `page` number; Gmail API accounts reject a page above 0, and the cursor works everywhere.

```javascript
async function searchAll(accountId, folderPath, searchCriteria, pageSize = 100) {
  const allResults = [];
  let cursor = null;

  do {
    const params = { pageSize };
    if (cursor) {
      params.cursor = cursor;
    }

    const data = await search(accountId, folderPath, searchCriteria, params);
    allResults.push(...data.messages);
    cursor = data.nextPageCursor;
  } while (cursor);

  return allResults;
}

// Find all unread messages (might be hundreds)
const allUnread = await searchAll('example', 'INBOX', { unseen: true });
```

`total` and `pages` are exact for IMAP accounts, approximate for Gmail API accounts, and can be missing for Microsoft Graph accounts, so do not drive the loop from them.

### Search Multiple Folders

```javascript
async function searchMultipleFolders(accountId, folders, searchCriteria) {
  const results = [];

  for (const folder of folders) {
    const data = await search(accountId, folder, searchCriteria);
    results.push(...data.messages.map(msg => ({ ...msg, folder })));
  }

  return results;
}

// Search for "invoice" in INBOX and the sent folder
const found = await searchMultipleFolders('example', ['INBOX', '\\Sent'], {
  subject: 'invoice'
});
```

Where `\All` is available it is one request instead of one per folder, and each result then carries its own `path`.

### Search and Process

Search and immediately process results:

```javascript
async function searchAndProcess(accountId, folderPath, searchCriteria, processor) {
  const data = await search(accountId, folderPath, searchCriteria);

  for (const message of data.messages) {
    try {
      await processor(accountId, message);
    } catch (err) {
      console.error(`Failed to process ${message.id}:`, err);
    }
  }

  return data.messages.length;
}

// Log read messages older than 30 days
const thirtyDaysAgo = new Date();
thirtyDaysAgo.setDate(thirtyDaysAgo.getDate() - 30);

const processed = await searchAndProcess(
  'example',
  'INBOX',
  {
    seen: true,
    before: thirtyDaysAgo.toISOString().split('T')[0]
  },
  async (accountId, message) => {
    console.log(`Old message: ${message.subject}`);
  }
);

console.log(`Processed ${processed} old messages`);
```

When the action is a flag change, a move or a delete, send the criteria to the [bulk endpoints](/docs/receiving/message-operations#bulk-actions) instead and let the server apply it to every match in one request.

## Search Performance Tips

### Narrow the Criteria

An IMAP SEARCH runs over every message in the folder, and a `body` search reads every message body. Prefer header fields, and add a date range whenever the application knows one:

```javascript
// Reads every body in the folder
const byBody = await search('example', 'INBOX', { body: 'meeting' });

// Header only
const bySubject = await search('example', 'INBOX', { subject: 'meeting' });

// Header only, and bounded by date
const recentBySubject = await search('example', 'INBOX', {
  subject: 'meeting',
  since: '2025-10-01'
});
```

### Cache Search Results

Repeated identical searches within a short window can be answered from a local cache:

```javascript
class SearchCache {
  constructor(ttl = 60000) {
    this.cache = new Map();
    this.ttl = ttl;
  }

  key(accountId, folderPath, searchCriteria) {
    return JSON.stringify({ accountId, folderPath, searchCriteria });
  }

  get(accountId, folderPath, searchCriteria) {
    const k = this.key(accountId, folderPath, searchCriteria);
    const cached = this.cache.get(k);

    if (cached && Date.now() - cached.timestamp < this.ttl) {
      return cached.results;
    }

    return null;
  }

  set(accountId, folderPath, searchCriteria, results) {
    const k = this.key(accountId, folderPath, searchCriteria);
    this.cache.set(k, {
      results,
      timestamp: Date.now()
    });

    // Clean up old entries
    if (this.cache.size > 100) {
      const oldestKey = this.cache.keys().next().value;
      this.cache.delete(oldestKey);
    }
  }
}

const searchCache = new SearchCache(60000); // 1 min TTL

async function cachedSearch(accountId, folderPath, searchCriteria) {
  const cached = searchCache.get(accountId, folderPath, searchCriteria);
  if (cached) return cached;

  const results = await search(accountId, folderPath, searchCriteria);
  searchCache.set(accountId, folderPath, searchCriteria, results);

  return results;
}
```

## Provider-Specific Considerations

### Gmail API

- Folders are labels, and a message can be in several. `path=\All` searches every message in the account regardless of label
- `labels` filters by label server-side; `gmailRaw` gives access to every Gmail operator, including `label:`, `has:attachment`, `larger:` and `category:`

```javascript
async function searchGmailByLabel(accountId, label) {
  return await search(accountId, '\\All', {
    labels: { has: [label] }
  });
}
```

### IMAP

- Text matching (`subject`, `body`, `from`, and so on) is done by the server. [RFC 3501](https://www.rfc-editor.org/rfc/rfc3501#section-6.4.4) specifies a case-insensitive substring match, but how a server handles encodings and word boundaries in body text is its own
- A `body` search can be slow on a large folder, because most servers read every message to answer it

## See Also

- [Message operations](/docs/receiving/message-operations) - Acting on the messages a search returns, one at a time or in bulk
- [Searching threads](/docs/sending/threading/searching-threads) - Retrieving a whole conversation
- [Message IDs](/docs/advanced/ids-explained) - The identifiers a search result carries
- [Search API](/docs/api/post-v-1-account-account-search) - The endpoint reference
