---
title: Searching Messages
sidebar_position: 6
description: "Complete guide to searching emails with EmailEngine - search queries, operators, filters, and best practices"
keywords:
  - email search
  - search queries
  - IMAP search
  - message filtering
  - search operators
---

# Searching Messages

<!--
Source attribution:
- EmailEngine API documentation
- IMAP SEARCH command reference
- Common search patterns
-->

The [search endpoint](/docs/api/post-v-1-account-account-search) finds messages in a connected account. For IMAP accounts the terms are translated into IMAP SEARCH, which every standards-compliant server implements.

## Why Use Search?

**Performance**
- Server-side filtering reduces data transfer
- Faster than fetching all messages and filtering locally
- Scales to mailboxes with thousands of messages

**Flexibility**
- Combine multiple criteria
- Search by date ranges, flags, content, headers
- Use complex boolean logic

**Efficiency**
- Returns only matching messages
- Reduces API calls and processing time
- Lower memory usage

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
async function searchBySubject(accountId, folderPath, subjectText) {
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/search?path=${folderPath}`,
    {
      method: 'POST',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        search: {
          subject: subjectText
        }
      })
    }
  );

  return await response.json();
}

const results = await searchBySubject('example', 'INBOX', 'meeting');
console.log(`Found ${results.messages.length} messages`);
```

### Search by Sender

Find messages from a specific sender:

```javascript
async function searchBySender(accountId, folderPath, email) {
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/search?path=${folderPath}`,
    {
      method: 'POST',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        search: {
          from: email
        }
      })
    }
  );

  return await response.json();
}

// Find all emails from john@example.com
const results = await searchBySender('example', 'INBOX', 'john@example.com');
```

### Search by Date

Find messages from a specific date range:

```javascript
async function searchByDateRange(accountId, folderPath, since, before) {
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/search?path=${folderPath}`,
    {
      method: 'POST',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        search: {
          since: since, // YYYY-MM-DD format
          before: before
        }
      })
    }
  );

  return await response.json();
}

// Find messages from last week
const today = new Date();
const weekAgo = new Date(today);
weekAgo.setDate(weekAgo.getDate() - 7);

const results = await searchByDateRange(
  'example',
  'INBOX',
  weekAgo.toISOString().split('T')[0], // 2025-10-06
  today.toISOString().split('T')[0]    // 2025-10-13
);
```

## Search Operators

### Text Search Operators

**subject** - Subject contains text
```javascript
{ search: { subject: 'invoice' } }
```

**body** - Message body contains text
```javascript
{ search: { body: 'payment' } }
```

**from** - Sender address contains text
```javascript
{ search: { from: 'john@example.com' } }
```

**to** - Recipient address contains text
```javascript
{ search: { to: 'jane@company.com' } }
```

**header** - Custom header search (object format)
```javascript
{ search: { header: { 'X-Custom-Header': 'value' } } }
```

### Date Operators

**since** - Messages on or after date (YYYY-MM-DD)
```javascript
{ search: { since: '2025-01-01' } }
```

**before** - Messages before date (YYYY-MM-DD)
```javascript
{ search: { before: '2025-12-31' } }
```

**sentSince** - Messages sent on or after date (YYYY-MM-DD)
```javascript
{ search: { sentSince: '2025-01-01' } }
```

**sentBefore** - Messages sent before date (YYYY-MM-DD)
```javascript
{ search: { sentBefore: '2025-12-31' } }
```

### Size Operators

**larger** - Messages larger than size (bytes)
```javascript
{ search: { larger: 1000000 } } // > 1MB
```

**smaller** - Messages smaller than size (bytes)
```javascript
{ search: { smaller: 10000 } } // < 10KB
```

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

**answered** - Messages that have been replied to
```javascript
{ search: { answered: true } }
```

**draft** - Draft messages
```javascript
{ search: { draft: true } }
```

### UID and ID Operators

**uid** - Specific UID or range (IMAP only)
```javascript
{ search: { uid: '12345' } }        // Single UID
{ search: { uid: '12345:12400' } }  // UID range
{ search: { uid: '12345:*' } }      // From UID to latest
```

**seq** - Sequence number range (IMAP only)
```javascript
{ search: { seq: '1:100' } }  // First 100 messages
```

**emailId** - Gmail Email ID (Gmail/modern IMAP only)
```javascript
{ search: { emailId: 'abc123def456' } }
```

**threadId** - Thread/conversation ID (Gmail/modern IMAP only)
```javascript
{ search: { threadId: 'thread_xyz789' } }
```

**emailIds** - Multiple specific Email IDs (Gmail/modern IMAP only)
```javascript
{ search: { emailIds: ['id1', 'id2', 'id3'] } }
```

### Gmail-Specific Operators

**gmailRaw** - Native Gmail search syntax (Gmail API only)
```javascript
{ search: { gmailRaw: 'is:unread category:primary' } }
{ search: { gmailRaw: 'from:boss@company.com has:attachment' } }
```

This passes Gmail's own search operators through untouched. See [Gmail search operators](https://support.google.com/mail/answer/7190).

### Label and Category Filtering

**labels** - Filter by Gmail labels or Outlook categories (Gmail and MS Graph accounts)

```javascript
{ search: { labels: { has: ['Important'] } } }            // Has the "Important" label
{ search: { labels: { has: ['Work', 'Urgent'] } } }       // Has ALL of these labels
{ search: { labels: { not: ['Processed'] } } }            // Has NONE of these labels
{ search: { labels: { has: ['Work'], not: ['Done'] } } }  // Combine both
```

- `has` matches messages that carry **all** of the listed labels/categories
- `not` excludes messages that carry **any** of the listed labels/categories
- Gmail API accounts: compiles to native `label:`/`-label:` query terms
- MS Graph (Outlook) accounts: filters by message categories
- Not supported for generic IMAP accounts or in `useOutlookSearch` mode - searching an account that cannot satisfy the filter returns HTTP 422

### Advanced IMAP Operators

**modseq** - Modification sequence (IMAP with CONDSTORE extension)
```javascript
{ search: { modseq: 12345 } }  // Messages modified after sequence 12345
```

**deleted** - Messages marked for deletion (IMAP only, not Gmail)
```javascript
{ search: { deleted: true } }
```

:::warning Provider Limitations
- `gmailRaw`: Gmail API only
- `labels`: Gmail and MS Graph (Outlook) accounts only - not supported for generic IMAP or in `useOutlookSearch` mode (returns HTTP 422 if the account cannot satisfy the filter)
- `emailId`, `threadId`, `emailIds`: Gmail and modern IMAP servers with RFC 8474 support
- `seq`, `modseq`, `deleted`, `answered`: Traditional IMAP only
- **Microsoft Graph API limitations:**
  - `to`, `cc`, `bcc`, `larger`, `smaller` fields are **not supported** for search by default
  - Use `from` and `subject` for filtering, or search `body` for recipient names
  - Consider using IMAP/SMTP mode for Outlook accounts if these search fields are required
  - Alternatively, use the `useOutlookSearch` query parameter to enable MS Graph `$search` mode, which supports `to`, `cc`, `bcc`, `larger`, `smaller`, `body`, `before`, `sentBefore`, `since`, and `sentSince` fields. Note: `$search` returns up to 1,000 results sorted by relevance (not date) and does not indicate total matches or pages.
:::

## Combined Searches

### Multiple Criteria (AND logic)

Combine multiple search criteria - all must match:

```javascript
async function searchUnreadFromSender(accountId, folderPath, sender) {
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/search?path=${folderPath}`,
    {
      method: 'POST',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        search: {
          from: sender,
          unseen: true
        }
      })
    }
  );

  return await response.json();
}

// Find unread messages from john@example.com
const results = await searchUnreadFromSender('example', 'INBOX', 'john@example.com');
```

### Complex Search Example

Find recent large unread invoices:

```javascript
async function searchRecentLargeInvoices(accountId) {
  const sevenDaysAgo = new Date();
  sevenDaysAgo.setDate(sevenDaysAgo.getDate() - 7);

  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/search?path=INBOX`,
    {
      method: 'POST',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        search: {
          subject: 'invoice',
          unseen: true,
          larger: 500000, // > 500KB
          since: sevenDaysAgo.toISOString().split('T')[0]
        }
      })
    }
  );

  return await response.json();
}

const invoices = await searchRecentLargeInvoices('example');
console.log(`Found ${invoices.messages.length} large unread invoices from last 7 days`);
```

## Common Search Patterns

### Find Messages with Attachments

```javascript
async function searchWithAttachments(accountId, folderPath) {
  // Note: Not all IMAP servers support attachment search
  // Alternative: List messages and filter by the attachments array

  const params = new URLSearchParams({
    path: folderPath,
    pageSize: 100
  });

  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/messages?${params}`,
    {
      headers: { 'Authorization': 'Bearer YOUR_ACCESS_TOKEN' }
    }
  );

  const data = await response.json();

  // Filter messages with attachments
  return data.messages.filter(msg => msg.attachments && msg.attachments.length > 0);
}
```

### Search by Message ID

Find a specific message by its Message-ID header:

```javascript
async function findByMessageId(accountId, messageId) {
  // Search across all folders
  const folders = ['INBOX', 'Sent', 'Archive'];

  for (const folder of folders) {
    const response = await fetch(
      `https://emailengine.example.com/v1/account/${accountId}/search?path=${folder}`,
      {
        method: 'POST',
        headers: {
          'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          search: {
            header: { 'Message-ID': messageId }
          }
        })
      }
    );

    const data = await response.json();

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

### Search Today's Messages

```javascript
async function searchTodaysMessages(accountId, folderPath) {
  const today = new Date();
  const todayStr = today.toISOString().split('T')[0];
  const tomorrow = new Date(today);
  tomorrow.setDate(tomorrow.getDate() + 1);
  const tomorrowStr = tomorrow.toISOString().split('T')[0];

  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/search?path=${folderPath}`,
    {
      method: 'POST',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        search: {
          since: todayStr,
          before: tomorrowStr
        }
      })
    }
  );

  return await response.json();
}
```

### Search This Month

```javascript
async function searchThisMonth(accountId, folderPath) {
  const now = new Date();
  const firstDay = new Date(now.getFullYear(), now.getMonth(), 1);
  const lastDay = new Date(now.getFullYear(), now.getMonth() + 1, 0);

  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/search?path=${folderPath}`,
    {
      method: 'POST',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        search: {
          since: firstDay.toISOString().split('T')[0],
          before: lastDay.toISOString().split('T')[0]
        }
      })
    }
  );

  return await response.json();
}
```

### Search Unread Important Messages

```javascript
async function searchUnreadImportant(accountId) {
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/search?path=INBOX`,
    {
      method: 'POST',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        search: {
          unseen: true,
          flagged: true
        }
      })
    }
  );

  return await response.json();
}
```

### Search by Keywords

Search for messages containing specific keywords in the message body:

```javascript
async function searchByKeywords(accountId, folderPath, keywords) {
  // Search in message body for each keyword
  const results = [];

  for (const keyword of keywords) {
    const response = await fetch(
      `https://emailengine.example.com/v1/account/${accountId}/search?path=${folderPath}`,
      {
        method: 'POST',
        headers: {
          'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          search: {
            body: keyword
          }
        })
      }
    );

    const data = await response.json();
    results.push(...data.messages);
  }

  // Remove duplicates
  const uniqueMessages = Array.from(
    new Map(results.map(msg => [msg.id, msg])).values()
  );

  return uniqueMessages;
}

// Search for messages about invoices or payments
const results = await searchByKeywords('example', 'INBOX', ['invoice', 'payment', 'bill']);
```

## Advanced Search Techniques

### Search with Pagination

Handle large search results:

```javascript
async function searchWithPagination(accountId, folderPath, searchCriteria, pageSize = 20) {
  const allResults = [];
  let page = 0;
  let hasMore = true;

  while (hasMore) {
    const response = await fetch(
      `https://emailengine.example.com/v1/account/${accountId}/search?path=${folderPath}&page=${page}&pageSize=${pageSize}`,
      {
        method: 'POST',
        headers: {
          'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          search: searchCriteria
        })
      }
    );

    const data = await response.json();
    allResults.push(...data.messages);

    page++;
    hasMore = page < data.pages;
  }

  return allResults;
}

// Find all unread messages (might be hundreds)
const allUnread = await searchWithPagination('example', 'INBOX', {
  unseen: true
}, 100);
```

### Search Multiple Folders

Search across multiple folders:

```javascript
async function searchMultipleFolders(accountId, folders, searchCriteria) {
  const results = await Promise.all(
    folders.map(async (folder) => {
      const response = await fetch(
        `https://emailengine.example.com/v1/account/${accountId}/search?path=${folder}`,
        {
          method: 'POST',
          headers: {
            'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({
            search: searchCriteria
          })
        }
      );

      const data = await response.json();
      return data.messages.map(msg => ({ ...msg, folder }));
    })
  );

  return results.flat();
}

// Search for "invoice" in Inbox and Archive
const results = await searchMultipleFolders('example', ['INBOX', 'Archive'], {
  subject: 'invoice'
});
```

### Search and Process

Search and immediately process results:

```javascript
async function searchAndProcess(accountId, folderPath, searchCriteria, processor) {
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/search?path=${folderPath}`,
    {
      method: 'POST',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        search: searchCriteria
      })
    }
  );

  const data = await response.json();

  for (const message of data.messages) {
    try {
      await processor(accountId, message);
    } catch (err) {
      console.error(`Failed to process ${message.id}:`, err);
    }
  }

  return data.messages.length;
}

// Archive all read messages older than 30 days
const thirtyDaysAgo = new Date();
thirtyDaysAgo.setDate(thirtyDaysAgo.getDate() - 30);

const archived = await searchAndProcess(
  'example',
  'INBOX',
  {
    seen: true,
    before: thirtyDaysAgo.toISOString().split('T')[0]
  },
  async (accountId, message) => {
    await archiveMessage(accountId, message.id);
    console.log(`Archived: ${message.subject}`);
  }
);

console.log(`Archived ${archived} old messages`);
```

## Search Performance Tips

### 1. Use Specific Search Criteria

```javascript
async function search(accountId, folderPath, searchCriteria) {
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/search?path=${folderPath}`,
    {
      method: 'POST',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ search: searchCriteria })
    }
  );
  return await response.json();
}

// Slow - searches entire message body
const results = await search('example', 'INBOX', {
  body: 'meeting'
});

// Faster - limits search to subject only
const results = await search('example', 'INBOX', {
  subject: 'meeting'
});

// Even faster - adds date constraint
const results = await search('example', 'INBOX', {
  subject: 'meeting',
  since: '2025-10-01'
});
```

### 2. Search Smaller Folders First

```javascript
async function smartSearch(accountId, searchParams) {
  // Try INBOX first (usually smaller than Archive)
  let results = await search(accountId, 'INBOX', searchParams);

  if (results.messages.length === 0) {
    // Only search archive if nothing found
    results = await search(accountId, 'Archive', searchParams);
  }

  return results;
}
```

### 3. Use Date Ranges

Always limit searches with date ranges when possible:

```javascript
// Bad - searches all messages (could be years worth)
async function search(accountId, folderPath, searchCriteria) {
  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/search?path=${folderPath}`,
    {
      method: 'POST',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ search: searchCriteria })
    }
  );
  return await response.json();
}

const results = await search('example', 'INBOX', {
  from: 'john@example.com'
});

// Good - limits to last 90 days
const ninetyDaysAgo = new Date();
ninetyDaysAgo.setDate(ninetyDaysAgo.getDate() - 90);

const results = await search('example', 'INBOX', {
  from: 'john@example.com',
  since: ninetyDaysAgo.toISOString().split('T')[0]
});
```

### 4. Cache Search Results

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

  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/search?path=${folderPath}`,
    {
      method: 'POST',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ search: searchCriteria })
    }
  );

  const results = await response.json();
  searchCache.set(accountId, folderPath, searchCriteria, results);

  return results;
}
```

## Provider-Specific Considerations

### Gmail API

Gmail API search has some differences:
- Labels are used instead of folders
- All messages are in `[Gmail]/All Mail`
- Use the `labels` search filter to filter by label server-side

```javascript
async function searchGmailByLabel(accountId, label) {
  const params = new URLSearchParams({ path: '\\All' });

  const response = await fetch(
    `https://emailengine.example.com/v1/account/${accountId}/search?${params}`,
    {
      method: 'POST',
      headers: {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        search: {
          labels: { has: [label] }
        }
      })
    }
  );

  return await response.json();
}
```

You can also use `gmailRaw` with `label:` terms for more complex label queries.

### IMAP Limitations

Some IMAP servers have limitations:
- Body search might not be supported
- Date search formats may vary
- Some servers don't support all flags

Always test searches with your specific provider.

