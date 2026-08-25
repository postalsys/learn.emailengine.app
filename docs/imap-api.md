---
title: IMAP API - REST API Wrapper for IMAP Protocol
sidebar_label: REST API Wrapper for IMAP Protocol
description: Access any IMAP mailbox through a simple REST API. EmailEngine converts complex IMAP protocol operations into easy HTTP requests with real-time webhooks.
sidebar_position: 99
slug: /imap-api
keywords:
  - IMAP API
  - IMAP REST API
  - IMAP wrapper
  - IMAP to REST
  - IMAP HTTP API
  - IMAP integration
---

# IMAP API - REST Interface for IMAP Mailboxes

EmailEngine provides a **REST API for IMAP mailboxes**, eliminating the need to implement complex IMAP protocol handling in your application.

## Why Use an IMAP API?

Working with IMAP directly requires:

- **Persistent TCP connections** - Managing long-lived socket connections
- **Protocol state machines** - Handling IMAP command sequences and states
- **IDLE implementation** - Maintaining connections for real-time updates
- **Provider quirks** - Dealing with non-standard implementations
- **Connection pooling** - Efficiently managing multiple mailbox connections
- **Error recovery** - Handling disconnections and reconnections

EmailEngine handles all of this complexity, exposing IMAP functionality through simple REST endpoints with real-time webhook notifications.

## IMAP API Operations

### List Mailboxes

Get all folders/mailboxes in an account:

```bash
curl "https://emailengine.example.com/v1/account/user123/mailboxes?counters=true" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

Response:
```json
{
  "mailboxes": [
    {"path": "INBOX", "specialUse": "\\Inbox", "status": {"messages": 1523, "unseen": 12}},
    {"path": "Sent", "specialUse": "\\Sent", "status": {"messages": 892, "unseen": 0}},
    {"path": "Drafts", "specialUse": "\\Drafts", "status": {"messages": 3, "unseen": 0}}
  ]
}
```

Message counts come from `status`, and only when `counters=true` is set. Without it the folder tree is returned without per-folder counts, which is much cheaper on an account with many folders.

### List Messages

Retrieve messages from a mailbox:

```bash
curl "https://emailengine.example.com/v1/account/user123/messages?path=INBOX&page=0&pageSize=20" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

Response:
```json
{
  "messages": [
    {
      "id": "AAAAAQAACnA",
      "uid": 1234,
      "date": "2025-01-15T10:30:00Z",
      "subject": "Meeting tomorrow",
      "from": {"address": "sender@example.com", "name": "John Doe"},
      "flags": ["\\Seen"]
    }
  ],
  "total": 1523,
  "page": 0,
  "pages": 77
}
```

### Get Message Content

Fetch a message with its body and attachment list. Body content is left out unless `textType` asks for it, because fetching it costs a round trip to the mail server:

```bash
curl "https://emailengine.example.com/v1/account/user123/message/AAAAAQAACnA?textType=*" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

Response:
```json
{
  "id": "AAAAAQAACnA",
  "subject": "Meeting tomorrow",
  "from": {"address": "sender@example.com", "name": "John Doe"},
  "to": [{"address": "you@example.com"}],
  "date": "2025-01-15T10:30:00Z",
  "text": {
    "plain": "Hi, let's meet tomorrow at 2pm...",
    "html": "<p>Hi, let's meet tomorrow at 2pm...</p>"
  },
  "attachments": [
    {
      "id": "AAAAAQAACnAy",
      "filename": "agenda.pdf",
      "contentType": "application/pdf",
      "encodedSize": 45231
    }
  ]
}
```

### Search Messages

Search using IMAP search criteria. The search terms go in the request body, so this is a POST:

```bash
curl -X POST "https://emailengine.example.com/v1/account/user123/search?path=INBOX" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "search": { "from": "john@example.com", "subject": "invoice" } }'
```

Or search the message body:

```bash
curl -X POST "https://emailengine.example.com/v1/account/user123/search?path=INBOX" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "search": { "body": "quarterly report" } }'
```

### Move/Copy Messages

Move a message to another folder:

```bash
curl -X PUT "https://emailengine.example.com/v1/account/user123/message/AAAAAQAACnA/move" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "path": "Archive" }'
```

### Update Flags

Mark messages as read, flagged, etc:

```bash
curl -X PUT "https://emailengine.example.com/v1/account/user123/message/AAAAAQAACnA" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "flags": { "add": ["\\Seen"], "delete": ["\\Flagged"] } }'
```

### Delete Messages

Move to trash or permanently delete:

```bash
curl -X DELETE "https://emailengine.example.com/v1/account/user123/message/AAAAAQAACnA" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

### Download Attachments

```bash
curl "https://emailengine.example.com/v1/account/user123/attachment/AAAAAQAACnAy" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -o agenda.pdf
```

## Real-Time Updates via Webhooks

EmailEngine maintains IMAP IDLE connections and sends webhooks when changes occur - no polling needed.

### New Message Webhook

```json
{
  "event": "messageNew",
  "account": "my-account",
  "data": {
    "id": "AAAAAQAACnB",
    "uid": 1235,
    "path": "INBOX",
    "subject": "New inquiry",
    "from": {"address": "prospect@example.com"}
  }
}
```

### Message Updated Webhook

```json
{
  "event": "messageUpdated",
  "account": "my-account",
  "data": {
    "id": "AAAAAQAACnA",
    "changes": {
      "flags": {
        "added": ["\\Seen"],
        "value": ["\\Seen"]
      }
    }
  }
}
```

### Message Deleted Webhook

```json
{
  "event": "messageDeleted",
  "account": "my-account",
  "path": "INBOX",
  "data": {
    "id": "AAAAAQAACnA",
    "uid": 1234
  }
}
```

[Full webhooks documentation →](/docs/webhooks/overview)

## Supported IMAP Providers

EmailEngine works with any IMAP server:

| Provider | Authentication | Notes |
|----------|---------------|-------|
| **Gmail** | OAuth2 or App Password | Full support including labels |
| **Google Workspace** | OAuth2 | Domain-wide delegation available |
| **Microsoft 365** | OAuth2 | Or use Microsoft Graph API |
| **Outlook.com** | OAuth2 | Consumer accounts |
| **Yahoo Mail** | OAuth2 | Including AOL, Verizon |
| **FastMail** | App Password | Full IMAP support |
| **ProtonMail** | Via Bridge | Requires ProtonMail Bridge |
| **Zoho Mail** | App Password | IMAP enabled accounts |
| **Custom IMAP** | User/Password or OAuth2 | Any standard IMAP server |

## IMAP API vs Direct IMAP

| Aspect | Direct IMAP | EmailEngine IMAP API |
|--------|-------------|---------------------|
| **Connection Management** | You handle | Automatic |
| **Real-time Updates** | Implement IDLE | Webhooks |
| **Authentication** | Handle OAuth2 flows | Built-in |
| **Error Recovery** | Build yourself | Automatic reconnection |
| **Protocol Complexity** | Full IMAP knowledge | Simple REST calls |
| **Scaling** | Connection pooling | Managed per-account |

## Performance Considerations

### Data Fetching

EmailEngine fetches message content on-demand from the IMAP server:

- **Metadata** (subject, from, date, flags) - Cached in Redis
- **Body content** - Fetched from IMAP when requested
- **Attachments** - Streamed from IMAP server

This means:
- First fetch may be slower than cached solutions
- No email content stored on your servers
- Always up-to-date with mailbox state

### Connection Handling

- One IMAP connection per registered account, plus one for each configured [sub-connection](/docs/accounts/managing-accounts#enable-sub-connections)
- Automatic IDLE for real-time updates
- Reconnection on network issues
- Connection pooling for SMTP

## Get Started with IMAP API

The examples above address a deployed instance at `emailengine.example.com`. The walkthrough below runs one locally, so it calls `http://localhost:3000`.

### 1. Install EmailEngine

```bash
# Using Docker
docker run -p 3000:3000 \
  --env EENGINE_REDIS="redis://host.docker.internal:6379/8" \
  postalsys/emailengine:v2
```

[Full installation guide →](/docs/installation)

### 2. Register an IMAP Account

```bash
curl -X POST http://localhost:3000/v1/account \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "my-mailbox",
    "email": "user@example.com",
    "imap": {
      "host": "imap.example.com",
      "port": 993,
      "secure": true,
      "auth": {
        "user": "user@example.com",
        "pass": "your-password"
      }
    }
  }'
```

### 3. List Messages

```bash
curl "http://localhost:3000/v1/account/my-mailbox/messages?path=INBOX" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

### 4. Configure Webhooks

```bash
curl -X POST http://localhost:3000/v1/settings \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "webhooks": "https://yourapp.com/webhooks",
    "webhooksEnabled": true,
    "webhookEvents": ["messageNew", "messageUpdated", "messageDeleted"]
  }'
```

## IMAP API Documentation

- [Account Management](/docs/accounts/managing-accounts) - Register and configure IMAP accounts
- [Messages API](/docs/api-reference/messages-api) - List, read, search, and manage messages
- [Webhooks](/docs/webhooks/overview) - Real-time event notifications
- [Gmail Setup](/docs/accounts/gmail/gmail-imap) - Configure Gmail IMAP access
- [Performance Tuning](/docs/advanced/performance-tuning) - Optimize for high volume

## Alternative: Native Provider APIs

For Gmail and Microsoft 365, EmailEngine also supports native APIs:

- **Gmail API** - Direct Google API integration with Pub/Sub
- **Microsoft Graph API** - Native Microsoft 365 integration

These provide additional features like native threading and can be faster for some operations.

[Gmail API setup →](/docs/accounts/gmail/gmail-api) | [Microsoft Graph setup →](/docs/accounts/microsoft-365/outlook-365)

## See Also

- [IMAP and SMTP accounts](/docs/accounts/imap-smtp) - Every connection setting, including TLS and autodiscovery
- [Messages API](/docs/api-reference/messages-api) - The full read and modify surface
- [Searching messages](/docs/receiving/searching) - What the search terms mean and which ones the server runs
- [Email API](/docs/email-api) - The same API described from the application side
- [Licensing](/docs/licensing) - Trial terms and what a production license covers
