---
title: AI Agent Reference
sidebar_position: 6
description: Quick reference for AI coding assistants building EmailEngine integrations
---

# EmailEngine AI Agent Reference

This document is designed for AI coding assistants helping developers integrate with the EmailEngine API. It provides a consolidated, machine-parseable overview of all capabilities, endpoints, and patterns.

## What EmailEngine Does

EmailEngine is a **self-hosted email API gateway** that provides REST API access to email accounts via:
- IMAP/SMTP protocols
- Gmail API (native)
- Microsoft Graph API (native)
- OAuth2 authentication

**Key value proposition:** Instead of dealing with IMAP/SMTP protocols directly, developers interact with a single REST API. EmailEngine handles connection management, authentication, synchronization, and real-time notifications via webhooks.

## Quick Facts

| Aspect | Details |
|--------|---------|
| API Style | RESTful JSON |
| Authentication | Bearer token (`Authorization: Bearer TOKEN`) |
| Base URL | `http://localhost:3000/v1` (default) |
| Webhooks | HTTP POST to configured endpoint |
| Data Storage | Redis (credentials encrypted with `EENGINE_SECRET`) |
| Message Storage | None - fetched from mail server on demand |
| Admin Auth | Password + TOTP, passkeys (WebAuthn), SSO (OpenID Connect, Okta) |
| AI Agents | MCP server at `POST /mcp`, off by default (see [MCP](/docs/mcp)) |

## Core Capabilities Matrix

| Capability | Endpoint | Key Parameters |
|------------|----------|----------------|
| **Register account** | `POST /v1/account` | `account`, `imap`, `smtp`, or `oauth2` |
| **List accounts** | `GET /v1/accounts` | `page`, `pageSize`, `state` |
| **Get account** | `GET /v1/account/{account}` | - |
| **Update account** | `PUT /v1/account/{account}` | Partial updates supported |
| **Delete account** | `DELETE /v1/account/{account}` | optional `?revoke=true` to revoke the OAuth2 grant |
| **Reconnect account** | `PUT /v1/account/{account}/reconnect` | - |
| **Send email** | `POST /v1/account/{account}/submit` | `to`, `subject`, `text`/`html` |
| **Send stored draft** | `POST /v1/account/{account}/message/{message}/submit` | optional delivery options |
| **List messages** | `GET /v1/account/{account}/messages` | `path`, `page`, `pageSize` |
| **Get message** | `GET /v1/account/{account}/message/{message}` | `textType`, `embedAttachedImages`, `preProcessHtml`, `webSafeHtml` (shorthand for all three; an explicit `embedAttachedImages=false` still overrides it) |
| **Get message text** | `GET /v1/account/{account}/text/{text}` | `textType`, `maxBytes`, `webSafeHtml` (returns a single sanitized HTML rendering) |
| **Search messages** | `POST /v1/account/{account}/search` | `search` object |
| **Update message** | `PUT /v1/account/{account}/message/{message}` | `flags`, `labels`, `seen` |
| **Delete message** | `DELETE /v1/account/{account}/message/{message}` | - |
| **Move message** | `PUT /v1/account/{account}/message/{message}/move` | `path` (destination) |
| **Download attachment** | `GET /v1/account/{account}/attachment/{attachment}` | - |
| **List mailboxes** | `GET /v1/account/{account}/mailboxes` | - |
| **Create mailbox** | `POST /v1/account/{account}/mailbox` | `path` |
| **Delete mailbox** | `DELETE /v1/account/{account}/mailbox` | `path` |
| **Configure webhooks** | `POST /v1/settings` | `webhooks`, `webhookEvents` |
| **Manage templates** | `POST /v1/templates/template` | `name`, `content`, `format` |
| **View outbox** | `GET /v1/outbox` | - |
| **Cancel queued email** | `DELETE /v1/outbox/{queueId}` | - |
| **Generate auth form** | `POST /v1/authentication/form` | `account`, `redirectUrl` |

## Complete API Endpoints

### Account Management

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/v1/account` | Register new email account |
| `GET` | `/v1/accounts` | List all accounts (paginated) |
| `GET` | `/v1/account/{account}` | Get account details and status |
| `PUT` | `/v1/account/{account}` | Update account configuration |
| `DELETE` | `/v1/account/{account}` | Delete account |
| `PUT` | `/v1/account/{account}/reconnect` | Force reconnection |
| `PUT` | `/v1/account/{account}/flush` | Reset sync state, re-index |
| `PUT` | `/v1/account/{account}/sync` | Trigger immediate sync |
| `GET` | `/v1/account/{account}/oauth-token` | Get current OAuth2 access token |
| `POST` | `/v1/verifyAccount` | Test account credentials |
| `POST` | `/v1/authentication/form` | Generate hosted auth form URL |
| `GET` | `/v1/autoconfig` | Auto-detect IMAP/SMTP settings |
| `GET` | `/v1/account/{account}/server-signatures` | List server signatures for account |

### Message Operations

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/v1/account/{account}/messages` | List messages in mailbox |
| `GET` | `/v1/account/{account}/message/{message}` | Get message details |
| `GET` | `/v1/account/{account}/message/{message}/source` | Get raw RFC822 source |
| `PUT` | `/v1/account/{account}/message/{message}` | Update flags/labels |
| `DELETE` | `/v1/account/{account}/message/{message}` | Delete message |
| `PUT` | `/v1/account/{account}/message/{message}/move` | Move to another mailbox |
| `POST` | `/v1/account/{account}/message` | Upload message to mailbox |
| `POST` | `/v1/account/{account}/search` | Search messages |
| `GET` | `/v1/account/{account}/text/{text}` | Get message text part |
| `GET` | `/v1/account/{account}/attachment/{attachment}` | Download attachment |

### Bulk Message Operations

| Method | Endpoint | Description |
|--------|----------|-------------|
| `PUT` | `/v1/account/{account}/messages` | Update flags or labels on every message matching a search |
| `PUT` | `/v1/account/{account}/messages/move` | Move multiple messages |
| `PUT` | `/v1/account/{account}/messages/delete` | Delete multiple messages |

### Export Operations (Beta)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/v1/account/{account}/export` | Create new bulk message export |
| `GET` | `/v1/account/{account}/exports` | List account exports (paginated) |
| `GET` | `/v1/account/{account}/export/{exportId}` | Get export status and details |
| `GET` | `/v1/account/{account}/export/{exportId}/download` | Download completed export file |
| `DELETE` | `/v1/account/{account}/export/{exportId}` | Cancel or delete export |

### Mailbox Operations

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/v1/account/{account}/mailboxes` | List all mailboxes/folders |
| `POST` | `/v1/account/{account}/mailbox` | Create mailbox |
| `PUT` | `/v1/account/{account}/mailbox` | Rename mailbox |
| `DELETE` | `/v1/account/{account}/mailbox` | Delete mailbox |

### Sending Emails

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/v1/account/{account}/submit` | Send/queue email |
| `POST` | `/v1/account/{account}/message/{message}/submit` | Send a stored draft by message ID |
| `GET` | `/v1/outbox` | List queued emails |
| `GET` | `/v1/outbox/{queueId}` | Get queued email details |
| `DELETE` | `/v1/outbox/{queueId}` | Cancel queued email |

### Templates

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/v1/templates` | List all templates |
| `POST` | `/v1/templates/template` | Create template |
| `GET` | `/v1/templates/template/{template}` | Get template |
| `PUT` | `/v1/templates/template/{template}` | Update template |
| `DELETE` | `/v1/templates/template/{template}` | Delete template |
| `DELETE` | `/v1/templates/account/{account}` | Delete all templates for an account |

### Settings & Configuration

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/v1/settings` | Get all settings |
| `POST` | `/v1/settings` | Update settings |
| `GET` | `/v1/settings/queue/{queue}` | Get queue configuration |
| `PUT` | `/v1/settings/queue/{queue}` | Update queue configuration |

### Webhooks

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/v1/webhookRoutes` | List webhook routes |
| `GET` | `/v1/webhookRoutes/webhookRoute/{webhookRoute}` | Get a webhook route |

Webhook routes are read-only through the API - create and edit them in the EmailEngine dashboard. Webhook delivery itself is configured via `POST /v1/settings` (`webhooks`, `webhooksEnabled`, `webhookEvents`).

### OAuth2 Applications

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/v1/oauth2` | List OAuth2 apps |
| `POST` | `/v1/oauth2` | Register OAuth2 app |
| `GET` | `/v1/oauth2/{app}` | Get OAuth2 app |
| `PUT` | `/v1/oauth2/{app}` | Update OAuth2 app |
| `DELETE` | `/v1/oauth2/{app}` | Delete OAuth2 app |
| `POST` | `/v1/oauth2/{app}/verify` | Verify OAuth2 app setup (read-only diagnostic) |

### SMTP Gateway

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/v1/gateways` | List SMTP gateways |
| `POST` | `/v1/gateway` | Register gateway |
| `GET` | `/v1/gateway/{gateway}` | Get gateway |
| `PUT` | `/v1/gateway/edit/{gateway}` | Update gateway |
| `DELETE` | `/v1/gateway/{gateway}` | Delete gateway |

### Access Tokens

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/v1/tokens` | List tokens, each with `id`, the SHA-256 hash identifying it. Add `?account={account}` to narrow it to one account |
| `POST` | `/v1/tokens` | Create token |
| `GET` | `/v1/tokens/{token}` | Get one token's metadata |
| `GET` | `/v1/tokens/{token}/log` | Read the token's audit log, when the audit log is enabled |
| `DELETE` | `/v1/tokens/{token}` | Delete token (accepts the token value or its `id` hash) |

### Blocklists

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/v1/blocklists` | List blocklists |
| `GET` | `/v1/blocklist/{listId}` | Get blocklist entries |
| `POST` | `/v1/blocklist/{listId}` | Add to blocklist |
| `DELETE` | `/v1/blocklist/{listId}` | Remove from blocklist |

### Monitoring & Stats

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/v1/stats` | Get usage statistics |
| `GET` | `/v1/logs/{account}` | Get account logs |
| `GET` | `/v1/changes` | Get recent changes |
| `GET` | `/v1/license` | Get license info |
| `POST` | `/v1/license` | Register a license key |
| `DELETE` | `/v1/license` | Remove the license key |
| `GET` | `/v1/pubsub/status` | List Pub/Sub subscription status |

### Deliverability Testing

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/v1/delivery-test/account/{account}` | Start delivery test |
| `GET` | `/v1/delivery-test/check/{deliveryTest}` | Check test results |

## MCP Endpoint (AI Agents)

EmailEngine also serves the Model Context Protocol, so an AI agent can call a curated tool set instead of the REST API. Off by default; an admin enables it under Configuration > MCP (`mcpEnabled` setting). Full docs: [MCP for AI Agents](/docs/mcp).

| Aspect | Details |
|--------|---------|
| Endpoint | `POST /mcp` |
| Transport | Streamable HTTP, JSON-RPC 2.0, stateless (no session id) |
| Protocol revisions | `2026-07-28` (modern, per-request `_meta` plus mirrored headers), `2025-11-25` and `2025-06-18` (legacy `initialize` handshake) |
| Authentication | `Authorization: Bearer <token>`. Prefer a token with the `mcp` scope, which opens this endpoint only |
| Methods | `initialize`, `server/discover`, `ping`, `tools/list`, `tools/call`, `resources/list`, `resources/read`, `resources/templates/list`, `subscriptions/listen` |
| Resources | `emailengine://account/{account}` per connected account |
| OAuth | `mcpOAuthEnabled` plus a Service URL adds dynamic client registration and an authorization code + PKCE flow for web connectors |

Every tool call is dispatched as the equivalent REST request with the caller's own credential, so scopes, permission narrowing, account binding, IP and referrer restrictions, rate limits and the audit log all apply unchanged. `tools/list` is filtered per credential, and a token bound to one account gets tools with no `account` argument (the binding is applied on dispatch).

Tool schemas are narrower than the endpoints they wrap: operator-level fields are hidden (`send_message` has no `gateway`, `envelope`, `headers`, `raw`, tracking or `mailMerge`), rendering options are pinned, and every paged listing caps `pageSize` at 100. `get_message` returns the body inline as sanitized web-safe HTML (32768-character budget, `text.hasMore` when longer, quoted history wrapped in `<details class="ee-collapsed-thread">`); `get_message_text` returns the same rendering with a 65536-character budget. A tool result is truncated above 128 KB.

### MCP Tools

| Tool | Behavior | Wraps |
|------|----------|-------|
| `list_accounts` | read-only | `GET /v1/accounts` |
| `get_account` | read-only | `GET /v1/account/{account}` |
| `list_mailboxes` | read-only | `GET /v1/account/{account}/mailboxes` |
| `list_messages` | read-only | `GET /v1/account/{account}/messages` |
| `search_messages` | read-only | `POST /v1/account/{account}/search` |
| `get_message` | read-only | `GET /v1/account/{account}/message/{message}` (body inline) |
| `get_message_text` | read-only | `GET /v1/account/{account}/text/{text}` |
| `get_attachment` | read-only | `GET /v1/account/{account}/attachment/{attachment}` |
| `update_message` | write | `PUT /v1/account/{account}/message/{message}` |
| `move_message` | write | `PUT /v1/account/{account}/message/{message}/move` |
| `delete_message` | destructive | `DELETE /v1/account/{account}/message/{message}` |
| `create_draft` | write | `POST /v1/account/{account}/message` |
| `send_message` | sends email | `POST /v1/account/{account}/submit` |
| `get_outbox` | read-only | `GET /v1/outbox` |
| `list_templates` | read-only | `GET /v1/templates` |

### MCP Access Levels

| Level | Permissions record |
|-------|--------------------|
| Read-only (default) | `{"actions":["read"],"groups":["account","mailbox","message","outbox","template"]}` |
| Mail agent | `{"actions":["read","write","send"],"groups":["account","mailbox","message","submit","outbox","template"]}` |
| Full access | no `permissions` record; the `mcp` scope is the bound |

Bind an agent token to one account whenever possible - a bound credential loses the instance-wide tools (`list_accounts`, `get_outbox`), reaches nothing else, and its remaining tools drop the `account` argument.

Replies and forwards go through the `reference` block on `send_message` and `create_draft`: `{message, action: reply|reply-all|forward, inline, forwardAttachments}`. EmailEngine derives the subject, the recipients and the threading headers from the referenced message.

## Webhook Events (24 Total)

### Message Events

| Event | Description | Key Payload Fields |
|-------|-------------|-------------------|
| `messageNew` | New email received | `data.id`, `data.from`, `data.to`, `data.subject`, `data.text` |
| `messageDeleted` | Email deleted | `data.id` |
| `messageUpdated` | Flags/labels changed | `data.id`, `data.changes` |
| `messageMissing` | Message not found | `data.id` |

### Delivery Events

| Event | Description | Key Payload Fields |
|-------|-------------|-------------------|
| `messageSent` | Email sent successfully | `data.messageId`, `data.response` |
| `messageDeliveryError` | Delivery attempt failed | `data.error`, `data.job.attemptsMade` |
| `messageFailed` | Delivery permanently failed | `data.error`, `data.messageId` |
| `messageBounce` | Bounce notification received | `data.recipient`, `data.bounceMessage` |
| `messageComplaint` | Spam complaint (ARF) | `data.recipient` |

### Account Events

| Event | Description | Key Payload Fields |
|-------|-------------|-------------------|
| `accountAdded` | Account registered | `account` |
| `accountDeleted` | Account removed | `account` |
| `accountInitialized` | Account ready | `account`, `state` |
| `authenticationError` | Auth failed | `account`, `data.response` |
| `authenticationSuccess` | Auth succeeded | `account` |
| `connectError` | Connection failed | `account`, `data.response` |

### Mailbox Events

| Event | Description | Key Payload Fields |
|-------|-------------|-------------------|
| `mailboxNew` | Folder created | `data.path` |
| `mailboxDeleted` | Folder deleted | `data.path` |
| `mailboxReset` | Folder UIDVALIDITY changed | `data.path` |

### Tracking Events

| Event | Description | Key Payload Fields |
|-------|-------------|-------------------|
| `trackOpen` | Email opened | `data.messageId`, `data.recipient` |
| `trackClick` | Link clicked | `data.messageId`, `data.url` |
| `listUnsubscribe` | User unsubscribed | `data.recipient` |
| `listSubscribe` | User re-subscribed | `data.recipient` |

### Export Events

| Event | Description | Key Payload Fields |
|-------|-------------|-------------------|
| `exportCompleted` | Export finished successfully | `data.exportId`, `data.messagesExported`, `data.bytesWritten` |
| `exportFailed` | Export failed | `data.exportId`, `data.error` |

## Common Patterns

### Pattern 1: Register an IMAP/SMTP Account

```bash
curl -X POST "https://emailengine.example.com/v1/account" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user123",
    "name": "John Doe",
    "email": "john@example.com",
    "imap": {
      "host": "imap.example.com",
      "port": 993,
      "secure": true,
      "auth": {
        "user": "john@example.com",
        "pass": "password"
      }
    },
    "smtp": {
      "host": "smtp.example.com",
      "port": 465,
      "secure": true,
      "auth": {
        "user": "john@example.com",
        "pass": "password"
      }
    }
  }'
```

### Pattern 2: Register OAuth2 Account (Gmail/Outlook/Mail.ru)

```bash
curl -X POST "https://emailengine.example.com/v1/account" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user123",
    "email": "john@gmail.com",
    "oauth2": {
      "provider": "OAUTH_APP_ID",
      "refreshToken": "REFRESH_TOKEN",
      "auth": {
        "user": "john@gmail.com"
      }
    }
  }'
```

### Pattern 3: Send a Simple Email

```bash
curl -X POST "https://emailengine.example.com/v1/account/user123/submit" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "to": [{"address": "recipient@example.com", "name": "Recipient"}],
    "subject": "Hello",
    "text": "Plain text body",
    "html": "<p>HTML body</p>"
  }'
```

### Pattern 4: Send Email with Attachments

```bash
curl -X POST "https://emailengine.example.com/v1/account/user123/submit" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "to": [{"address": "recipient@example.com"}],
    "subject": "Document attached",
    "text": "Please find the document attached.",
    "attachments": [
      {
        "filename": "document.pdf",
        "content": "BASE64_ENCODED_CONTENT",
        "contentType": "application/pdf"
      }
    ]
  }'
```

### Pattern 5: Reply to an Email

```bash
curl -X POST "https://emailengine.example.com/v1/account/user123/submit" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "to": [{"address": "original-sender@example.com"}],
    "subject": "Re: Original Subject",
    "text": "My reply",
    "reference": {
      "message": "ORIGINAL_MESSAGE_ID",
      "action": "reply"
    }
  }'
```

### Pattern 6: Forward an Email

```bash
curl -X POST "https://emailengine.example.com/v1/account/user123/submit" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "to": [{"address": "forward-to@example.com"}],
    "subject": "Fwd: Original Subject",
    "text": "Forwarding this email",
    "reference": {
      "message": "ORIGINAL_MESSAGE_ID",
      "action": "forward"
    }
  }'
```

### Pattern 7: Search Messages

```bash
curl -X POST "https://emailengine.example.com/v1/account/user123/search" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "search": {
      "from": "sender@example.com",
      "subject": "invoice",
      "unseen": true,
      "since": "2024-01-01"
    }
  }'
```

### Pattern 8: Configure Webhooks

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "webhooks": "https://your-app.com/webhooks",
    "webhooksEnabled": true,
    "webhookEvents": ["messageNew", "messageSent", "messageFailed"]
  }'
```

### Pattern 9: Handle Webhook (Node.js)

```javascript
app.post('/webhooks', express.json(), (req, res) => {
  const { event, account, data } = req.body;

  // Acknowledge immediately
  res.status(200).json({ success: true });

  // Process asynchronously
  switch (event) {
    case 'messageNew':
      // New email: data.id, data.from, data.to, data.subject
      break;
    case 'messageSent':
      // Email sent: data.messageId
      break;
    case 'messageFailed':
      // Delivery failed: data.error
      break;
  }
});
```

### Pattern 10: List and Paginate Messages

```bash
# First page (20 messages)
curl "https://emailengine.example.com/v1/account/user123/messages?path=INBOX&page=0&pageSize=20" \
  -H "Authorization: Bearer YOUR_TOKEN"

# Next page
curl "https://emailengine.example.com/v1/account/user123/messages?path=INBOX&page=1&pageSize=20" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

### Pattern 11: Mail Merge (Bulk Personalized Emails)

```bash
curl -X POST "https://emailengine.example.com/v1/account/user123/submit" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": "Hello {{name}}",
    "html": "<p>Dear {{name}}, your order #{{orderId}} is ready.</p>",
    "mailMerge": [
      {
        "to": [{"address": "alice@example.com"}],
        "params": {"name": "Alice", "orderId": "1001"}
      },
      {
        "to": [{"address": "bob@example.com"}],
        "params": {"name": "Bob", "orderId": "1002"}
      }
    ]
  }'
```

### Pattern 12: Generate Hosted Authentication Form

```bash
# Generate form URL
curl -X POST "https://emailengine.example.com/v1/authentication/form" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "new-user",
    "name": "New User",
    "redirectUrl": "https://your-app.com/settings"
  }'

# Response: {"url": "https://emailengine.example.com/accounts/new?data=..."}
# Redirect user to this URL to complete authentication

# Add "expectedEmail" to reject the setup unless the user authenticates as that
# address. Stored on the account, so it also covers later links that omit it.
# Comparison is exact apart from case, so Gmail dot/googlemail/+tag variants
# are rejected - pass the address exactly as the provider reports it.
# A rejected setup is NOT redirected to redirectUrl: EmailEngine shows the user
# both addresses and a button to retry. Only success redirects.
# Clear it with PUT /v1/account/{account} {"expectedEmail": null}
```

## Account Types

| Type | Value | Description | Requirements |
|------|-------|-------------|--------------|
| IMAP/SMTP | `imap` | Standard email protocol | Host, port, credentials |
| Gmail OAuth2 | `gmail` | Gmail via OAuth2 + IMAP | OAuth2 app, refresh token |
| Gmail API | `gmail` | Gmail native API | OAuth2 app, Cloud Pub/Sub |
| Gmail Service Account | `gmailService` | Google Workspace domain-wide | Service account key |
| Outlook OAuth2 | `outlook` | Microsoft via OAuth2 | Azure AD app, refresh token |
| MS Graph API | `outlook` | Microsoft native API | Azure AD app, graph subscription |
| Outlook Application Access | `outlookService` | Microsoft 365 via client credentials | Azure AD app, tenant ID |
| Mail.ru OAuth2 | `mailRu` | Mail.ru via OAuth2 + IMAP | OAuth2 app, refresh token |
| Generic OAuth2 | `oauth2` | Generic OAuth2 provider | OAuth2 app configuration |

## Account States

| State | Description | Next Steps |
|-------|-------------|------------|
| `init` | Being initialized | Wait |
| `connecting` | Establishing connection | Wait |
| `syncing` | Initial or periodic sync in progress | Wait |
| `connected` | Active and operational | Ready for API calls |
| `disconnected` | Connection lost | Will auto-reconnect |
| `authenticationError` | Credentials rejected | Update credentials or re-authorize |
| `connectError` | Network/server error | Check connectivity |
| `paused` | Syncing paused through the API | Resume syncing |
| `unset` | No usable IMAP or OAuth2 configuration, or `imap.disabled` is set | Finish setup, or clear `imap.disabled` |

## Error Handling

### HTTP Status Codes

| Code | Meaning | Action |
|------|---------|--------|
| `200` | Success | - |
| `400` | Bad Request | Check request parameters |
| `401` | Unauthorized | Verify API token |
| `403` | Forbidden | Check token permissions |
| `404` | Not Found | Verify account/message ID |
| `429` | Rate Limited | Retry with backoff |
| `500` | Server Error | Retry after delay |
| `503` | Unavailable | Service restarting, retry |

### Error Response Format

```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": "Human-readable message",
  "code": "ErrorCode"
}
```

`error` carries the HTTP status phrase, not the explanation. Read `message`.

### Common Error Codes

| Code | Description |
|------|-------------|
| `AccountNotFound` | Account doesn't exist |
| `MessageNotFound` | Message doesn't exist |
| `InvalidRequest` | Request validation failed |
| `InvalidToken` | Missing or invalid API access token |
| `AccountAlreadyExists` | An account with the same ID (or OAuth2 user) already exists |
| `RateLimitExceeded` | Too many requests |
| `ConnectionError` | Can't connect to mail server |

## Decision Trees

### Choosing Account Type

```mermaid
flowchart TD
    A[Email Provider?] --> B{Gmail personal?}
    B -->|Yes| C[Gmail OAuth2<br/>IMAP or API]
    B -->|No| D{Google Workspace<br/>domain-wide?}
    D -->|Yes| E[Service Account]
    D -->|No| F{Microsoft 365 /<br/>Outlook.com?}
    F -->|Yes| G[Outlook OAuth2<br/>IMAP or MS Graph]
    F -->|No| H{Yahoo / AOL /<br/>Verizon?}
    H -->|Yes| I[IMAP/SMTP<br/>with app password]
    H -->|No| J[IMAP/SMTP<br/>with credentials]
```

### Gmail API vs Gmail IMAP

```mermaid
flowchart TD
    A[Gmail Account Setup] --> B{Need limited OAuth2 scopes?<br/>gmail.readonly, gmail.modify}
    B -->|Yes| C[Gmail API<br/>required for limited scopes]
    B -->|No| D{Need maximum<br/>listing performance?}
    D -->|Yes| E[Gmail IMAP<br/>faster listing]
    D -->|No| F{Need real-time<br/>push notifications?}
    F -->|Yes| G[Gmail API<br/>uses Cloud Pub/Sub]
    F -->|No| H[Either works<br/>Gmail API recommended]
```

### MS Graph vs Outlook IMAP

```mermaid
flowchart TD
    A[Microsoft Account Setup] --> AA{Need app-level access<br/>without user login?}
    AA -->|Yes| AB[outlookService<br/>Client Credentials]
    AA -->|No| B{Need shared<br/>mailbox access?}
    B -->|Yes| C[MS Graph API]
    B -->|No| D{Need advanced search?}
    D -->|Yes| E[Outlook IMAP<br/>Graph has limited search]
    D -->|No| F{Outlook.com /<br/>Hotmail?}
    F -->|Yes| G[Either works<br/>MS Graph recommended]
    F -->|No| H[MS Graph API<br/>recommended]
```

## Key Settings (via POST /v1/settings)

| Setting | Type | Description |
|---------|------|-------------|
| `webhooks` | string | Webhook URL |
| `webhooksEnabled` | boolean | Enable webhook delivery |
| `webhookEvents` | array | Event types to trigger |
| `inboxNewOnly` | boolean | Only trigger `messageNew` for Inbox folder |
| `serviceUrl` | string | Public URL of EmailEngine instance |
| `serviceSecret` | string | HMAC secret for webhook signature verification |
| `resolveGmailCategories` | boolean | Detect Gmail tabs (Primary, Social, etc.) for IMAP |
| `smtpEhloName` | string | Custom EHLO hostname for SMTP connections |
| `ignoreMailCertErrors` | boolean | Accept invalid TLS certificates |
| `trackOpens` | boolean | Enable email open tracking |
| `trackClicks` | boolean | Enable click tracking |
| `imapIndexer` | string | Indexing strategy: `full` or `fast` |
| `scriptEnv` | string | JSON environment variables for pre-processing scripts |
| `httpProxyEnabled` | boolean | Route outbound HTTP/HTTPS requests through proxy |
| `httpProxyUrl` | string | HTTP/SOCKS proxy URL for outbound requests |
| `pageBrandName` | string | Custom brand name displayed in page titles |
| `notifyText` | boolean | Include plain text content in webhook payloads |
| `notifyTextSize` | number | Max text size in webhook payloads (bytes) |
| `notifyAttachments` | boolean | Include attachments in webhook payloads |
| `notifyAttachmentSize` | number | Max attachment size in webhook payloads (bytes) |
| `notifyCalendarEvents` | boolean | Include calendar events in webhook payloads |
| `notifyWebSafeHtml` | boolean | Replace the HTML body in webhook payloads with a web-safe version (sanitized, inline images embedded, quoted thread history folded into `<details class="ee-collapsed-thread">`) |
| `locale` | string | UI language/locale |
| `timezone` | string | Default timezone (IANA identifier) |
| `templateHeader` | string | Custom HTML header for hosted pages |
| `templateHtmlHead` | string | Custom HTML for page head section |
| `imapClientName` | string | IMAP ID extension client name |
| `imapClientVersion` | string | IMAP ID extension version |
| `imapClientVendor` | string | IMAP ID extension vendor |
| `imapClientSupportUrl` | string | IMAP ID extension support URL |

## Per-Account IMAP Settings

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `disabled` | boolean | `false` | Disable IMAP (send-only mode) |
| `resyncDelay` | number | `900` | Seconds between full mailbox resyncs |
| `sentMailPath` | string | auto | Custom Sent folder path |
| `draftsMailPath` | string | auto | Custom Drafts folder path |
| `junkMailPath` | string | auto | Custom Junk folder path |
| `trashMailPath` | string | auto | Custom Trash folder path |
| `archiveMailPath` | string | auto | Custom Archive folder path |
| `useAuthServer` | boolean | `false` | Fetch credentials from external auth server |

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `EENGINE_REDIS` | Yes | Redis URL (e.g., `redis://localhost:6379/8`) |
| `EENGINE_SECRET` | Prod | Encryption key for credentials (32+ hex chars) |
| `EENGINE_PORT` | No | API port (default: 3000) |
| `EENGINE_HOST` | No | Bind address (default: 127.0.0.1) |
| `EENGINE_WORKERS` | No | Account worker count - IMAP, Gmail API, Outlook/Graph (default: 4) |
| `EENGINE_WORKERS_API` | No | API/HTTP worker count (default: 1); values >1 need SO_REUSEPORT (Linux, Node.js 23.1+) |
| `EENGINE_WORKERS_WEBHOOKS` | No | Webhook worker count (default: 1) |
| `EENGINE_WORKERS_SUBMIT` | No | Submit worker count (default: 1) |
| `EENGINE_WORKERS_EXPORT` | No | Export worker count (default: 1) |
| `EENGINE_LOG_LEVEL` | No | Log level (trace/debug/info/warn/error) |
| `EENGINE_CORS_MAX_AGE` | No | CORS preflight cache duration in seconds (default: 60) |
| `EENGINE_HTTP_PROXY_ENABLED` | No | Enable HTTP proxy for outbound requests |
| `EENGINE_HTTP_PROXY_URL` | No | HTTP/SOCKS proxy URL for outbound requests |

## Submit API Key Parameters

The `POST /v1/account/{account}/submit` endpoint accepts these key parameters:

| Parameter | Type | Description |
|-----------|------|-------------|
| `to` | array | Recipients `[{address, name}]` (required) |
| `cc` | array | CC recipients |
| `bcc` | array | BCC recipients |
| `subject` | string | Email subject |
| `text` | string | Plain text body |
| `html` | string | HTML body |
| `from` | object | Override sender `{address, name}` |
| `replyTo` | object | Reply-to address |
| `attachments` | array | Attachments `[{filename, content, contentType}]` |
| `headers` | object | Custom headers |
| `reference` | object | For replies/forwards `{message, action}` |
| `template` | string | Template ID to use |
| `mailMerge` | array | Bulk send with personalization |
| `sendAt` | string | Schedule sending (ISO 8601) |
| `trackOpens` | boolean | Enable open tracking |
| `trackClicks` | boolean | Enable click tracking |
| `copy` | boolean | Save to Sent folder (default: true) |
| `dryRun` | boolean | Preview without sending |
| `gateway` | string | Use specific SMTP gateway |
| `deliveryAttempts` | number | Max retry attempts |

To send an email that already exists as a draft, use `POST /v1/account/{account}/message/{message}/submit` with the draft's message ID instead of composing content. The optional body accepts the delivery options above (`envelope`, `copy`, `sentMailPath`, `sendAt`, `deliveryAttempts`, `gateway`, `dsn`, `proxy`, `localAddress`) but no content fields - the draft is sent as stored. Gmail and MS Graph accounts send it with the provider's native draft-send call; the draft is removed after sending on all account types.

## Search Parameters

The `POST /v1/account/{account}/search` endpoint accepts these search criteria:

| Parameter | Type | Description |
|-----------|------|-------------|
| `from` | string | Sender address/name |
| `to` | string | Recipient address/name |
| `subject` | string | Subject contains |
| `body` | string | Body contains |
| `unseen` | boolean | Unread only |
| `flagged` | boolean | Starred/flagged only |
| `since` | string | After date (YYYY-MM-DD) |
| `before` | string | Before date (YYYY-MM-DD) |
| `header` | object | Custom header match |
| `emailId` | string | Specific message ID |
| `threadId` | string | Specific thread ID |
| `labels` | object | `{ "has": [...], "not": [...] }` - filter by Gmail labels or Outlook categories. `has` matches messages with ALL listed labels, `not` excludes messages with ANY of them. Gmail and MS Graph accounts only; returns HTTP 422 if the account cannot satisfy the filter |

## See Also

- [Full API Reference](/docs/api/emailengine-api) - Auto-generated OpenAPI documentation
- [OpenAPI Specification](/docs/api-reference/openapi-spec) - Download the raw spec or generate a client
- [Webhook Events Reference](/docs/reference/webhook-events) - Complete webhook payload docs
- [Quick Reference](/docs/reference/quick-reference) - Tables for common lookups
- [MCP for AI Agents](/docs/mcp) - Tool set, access control and protocol details for agent access
- [Machine-Readable Capabilities](/capabilities.json) - JSON capabilities manifest
