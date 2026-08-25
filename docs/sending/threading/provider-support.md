---
title: Provider-Specific Threading
sidebar_position: 2
description: Detailed threading support for different email providers
---

# Provider-Specific Threading Support

Different email providers implement threading in different ways. Understanding these differences helps you choose the right approach for your application.

## Gmail / Google Workspace

Gmail has the best native threading support, working with both backend types.

### Native Thread Support

**Backends**:

- IMAP with OAuth2: Full threading support
- Gmail API: Full threading support

**Thread ID Property**: `X-GM-THRID` (from Gmail-specific `X-GM-EXT-1` IMAP extension)

**Characteristics**:

- Native threading support
- Thread ID format: Long numeric string (e.g., `"1759349012996310407"`)
- Availability: Webhooks, message listings, search results
- Supports `\All` folder for cross-folder thread search

### Example Response

```bash
curl "https://emailengine.example.com/v1/account/gmail/messages?path=INBOX" \
  -H "Authorization: Bearer <token>"
```

```json
{
  "messages": [
    {
      "id": "AAABkPHBeR0",
      "threadId": "1759349012996310407",
      "subject": "Project discussion",
      "from": {
        "address": "colleague@example.com"
      }
    }
  ]
}
```

### Gmail-Specific Features

**Cross-Folder Search with `\All`**:

Gmail supports searching across all folders using the special `\All` path:

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

This returns all messages in the thread from Inbox, Sent, and any other folder in a single response.

## Microsoft Graph API

Microsoft Graph API provides native threading support for Outlook/Microsoft 365 accounts.

### Native Thread Support

**Backend**:

- Microsoft Graph API: Full threading support
- IMAP with OAuth2: No native threading

**Important Distinction**: Only accounts added with **Microsoft Graph API backend** have native threading. Outlook/Microsoft 365 accounts added via **IMAP with OAuth2** do not have native thread IDs and must build threads manually from Message-ID headers.

**Thread ID Property**: Native conversation ID from Graph API

**Characteristics**:

- Native threading support
- Thread ID format: Graph API conversation ID
- Availability: Webhooks, message listings, search results
- Supports `\All` folder for cross-folder thread search

### Example Response

```json
{
  "messages": [
    {
      "id": "AAMkAGI2T...",
      "threadId": "AAQkAGI2TH...",
      "subject": "Meeting tomorrow",
      "from": {
        "address": "manager@example.com"
      }
    }
  ]
}
```

### Graph API-Specific Features

**Cross-Folder Search with `\All`**:

Like Gmail, Graph API supports the `\All` folder:

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

### Why IMAP Backend Doesn't Have Threading

When you add an Outlook/Microsoft 365 account using **IMAP with OAuth2**, EmailEngine connects via IMAP protocol, not Graph API. IMAP doesn't provide access to Microsoft's conversation/thread IDs, so these accounts behave like generic IMAP accounts and threads must be built manually from Message-ID headers.

**Account Backend Comparison**:

| Backend             | Threading   | \All Support |
| ------------------- | ----------- | ------------ |
| Microsoft Graph API | Native      | Yes          |
| IMAP + OAuth2       | Manual only | No           |

## Yahoo / AOL / Verizon

Yahoo, AOL, and Verizon accounts support the OBJECTID extension (RFC 8474).

### Native Thread Support

**IMAP Extension**: `OBJECTID` (RFC 8474)

**Thread ID Property**: `THREADID`

**Characteristics**:

- Native threading support
- Thread ID format: Short numeric string (e.g., `"501"`)
- Availability: Webhooks, message listings, search results
- **No `\All` folder support** - must search folders individually

### Example Response

```json
{
  "messages": [
    {
      "id": "AAAAKAAACKM",
      "threadId": "501",
      "subject": "Question about your service",
      "from": {
        "address": "customer@example.com"
      }
    }
  ]
}
```

### Limitations

**Folder-by-Folder Search**:

Unlike Gmail and Graph API, Yahoo/AOL don't support `\All`. To get a complete thread, you must:

1. Search in Inbox
2. Search in Sent
3. Search in any other relevant folders
4. Combine results client-side

**Example**:

```bash
# Search Inbox
curl -XPOST "https://emailengine.example.com/v1/account/yahoo/search?path=INBOX" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "search": { "threadId": "501" }
  }'

# Search Sent
curl -XPOST "https://emailengine.example.com/v1/account/yahoo/search?path=Sent" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "search": { "threadId": "501" }
  }'

# Combine results in your application
```

## Other IMAP Providers

Generic IMAP providers without threading extensions do not have native threading support.

### Manual Threading Required

**Capability**: No native threading

**Thread Building**: Must analyze Message-ID, In-Reply-To, and References headers manually

**Characteristics**:

- **No automatic thread IDs** - EmailEngine doesn't provide `threadId` property
- **No `\All` folder support** - must search folders individually
- Must implement client-side threading logic

### How to Build Threads Manually

1. Retrieve messages from relevant folders (Inbox, Sent, etc.)
2. Extract Message-ID, In-Reply-To, and References headers
3. Build thread relationships by matching Message-IDs
4. Group related messages together
5. Sort chronologically

See [Searching Thread Messages](./searching-threads.md#building-threads-manually) examples.

### Folder-by-Folder Search Required

Generic IMAP doesn't support `\All` folder, so you must search each folder individually:

```bash
# Search Inbox
curl -XPOST "https://emailengine.example.com/v1/account/generic/search?path=INBOX" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "search": { "subject": "Project discussion" }
  }'

# Search Sent
curl -XPOST "https://emailengine.example.com/v1/account/generic/search?path=Sent" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "search": { "subject": "Project discussion" }
  }'
```

Then analyze the Message-ID headers to build thread relationships.

## Provider Comparison Table

| Provider          | Backend       | Native Threading | Thread ID Format      | \All Folder |
| ----------------- | ------------- | ---------------- | --------------------- | ----------- |
| Gmail             | IMAP + OAuth2 | Yes              | Long numeric          | Yes         |
| Gmail             | Gmail API     | Yes              | Long numeric          | Yes         |
| Microsoft 365     | Graph API     | Yes              | Graph conversation ID | Yes         |
| Microsoft 365     | IMAP + OAuth2 | No               | N/A (manual only)     | No          |
| Yahoo/AOL/Verizon | IMAP          | Yes (OBJECTID)   | Short numeric         | No          |
| Generic IMAP      | IMAP          | No               | N/A (manual only)     | No          |

## Choosing the Right Backend

### For Gmail Accounts

Always use either:

- IMAP with OAuth2 (has native threading)
- Gmail API (has native threading)

Both provide excellent threading support with `\All` folder capability.

### For Microsoft 365 / Outlook Accounts

**Recommended**: Use **Microsoft Graph API** backend for:

- Native threading support
- `\All` folder cross-search capability
- Better performance
- Modern API features

**Avoid**: IMAP with OAuth2 if you need threading, as it requires manual thread building and doesn't support `\All`.

### For Yahoo / AOL / Verizon

Use standard IMAP connection. Threading works natively but:

- No `\All` support
- Must search multiple folders separately

### For Other Providers

**Manual Threading Required**: Plan to implement client-side threading logic for:

- Building thread relationships from Message-ID headers
- Cross-folder thread queries
- Maintaining thread state in your application

## Migration Considerations

### Switching from IMAP to Graph API

If you have Microsoft 365 accounts on IMAP backend and want native threading:

1. Account re-registration required (different authentication flow)
2. Thread IDs will become available (manual threading → Graph conversation IDs)
3. Update your application to use native thread IDs from Graph API
4. Historical thread mappings may need migration

## Testing Thread Support

### Check Your Provider

```bash
# Get account information
curl "https://emailengine.example.com/v1/account/example" \
  -H "Authorization: Bearer <token>"
```

Look for:

- `type`: Account type (imap, gmail, gmailService, outlook, outlookService, mailRu, oauth2)

### Test Thread ID Availability

```bash
# List messages and check for threadId
curl "https://emailengine.example.com/v1/account/example/messages?path=INBOX" \
  -H "Authorization: Bearer <token>"
```

If `threadId` is missing:

- Provider doesn't support native threading
- Must build threads manually from Message-ID headers

## See Also

- [Threading overview](/docs/sending/threading/overview) - The headers and IDs this table is about
- [Searching threads](/docs/sending/threading/searching-threads) - The search strategy each provider needs
- [Account types](/docs/accounts) - Choosing the backend that gives you native threading
- [Searching messages](/docs/receiving/searching) - What else the search endpoint can filter on
