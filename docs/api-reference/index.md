---
title: API Reference Overview
description: Complete API reference for EmailEngine with authentication, conventions, and error handling
sidebar_position: 1
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# EmailEngine API Reference

The EmailEngine API is a REST interface for managing email accounts, sending and receiving messages, and configuring webhooks, so an application never has to speak IMAP or SMTP itself.

## Overview

### What is the EmailEngine API

The EmailEngine API is a RESTful HTTP API that:

- Manages email accounts across multiple providers (Gmail, Outlook, IMAP/SMTP)
- Sends and receives emails programmatically
- Provides real-time notifications via webhooks
- Handles OAuth2 authentication automatically
- Maintains mailbox synchronization in the background

### Architecture

- **RESTful design**: Uses standard HTTP methods (GET, POST, PUT, DELETE)
- **JSON format**: All requests and responses use JSON
- **Stateless**: Each request contains all necessary authentication
- **Event-driven**: Webhooks notify your application of changes in real-time

## Base URL

The default base URL for all API endpoints is:

```
http://localhost:3000/v1
```

For production deployments, replace with your EmailEngine instance URL:

```
https://emailengine.example.com/v1
```

### Versioning

The API version is included in the URL path (`/v1`). This ensures backward compatibility when new versions are released.

## Authentication

All API requests require authentication using Bearer tokens.

### API Token Authentication

Include your access token in the `Authorization` header:

```http
Authorization: Bearer YOUR_ACCESS_TOKEN
```

### Creating Access Tokens

**Via Settings Page (System-Wide Tokens):**

1. Log in to the EmailEngine web interface
2. Navigate to Integrations > Access Tokens
3. Click "Generate new token"
4. Assign a description and optional scope
5. Copy the generated token

**Via API (Account-Specific Tokens Only):**

```bash
curl -X POST http://localhost:3000/v1/tokens \
  -H "Authorization: Bearer EXISTING_SYSTEM_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user123",
    "description": "User API Token",
    "scopes": ["api"]
  }'
```

**Important:** Tokens created via API are ALWAYS account-specific. The `account` field is required.

**Via CLI (System-Wide or Account-Specific):**

```bash
# System-wide token
emailengine tokens issue -d "Admin token" -s "*"

# Account-specific token
emailengine tokens issue -d "User token" -s "api" -a "user123"
```

See [Access Tokens](/docs/api-reference/access-tokens) for complete documentation.

### Token Types

**System-Wide Tokens:**
- Created via web interface or CLI
- Access all accounts and endpoints
- Scopes: `"*"` (full), `"api"`, `"metrics"`, `"smtp"`, `"imap-proxy"`

**Account-Specific Tokens:**
- Created via API (requires `account` field)
- Restricted to single account only
- Cannot create other tokens
- Recommended for multi-tenant applications

### Security Best Practices

- Never expose tokens in client-side code
- Use environment variables for token storage
- Rotate tokens periodically
- Use account-specific tokens when possible
- Revoke unused tokens immediately

## Making Requests

### HTTP Methods

| Method | Purpose            | Example              |
| ------ | ------------------ | -------------------- |
| GET    | Retrieve resources | Get account details  |
| POST   | Create resources   | Register new account |
| PUT    | Update resources   | Update message flags |
| DELETE | Remove resources   | Delete account       |

### Request Headers

Required headers for most requests:

```http
Authorization: Bearer YOUR_ACCESS_TOKEN
Content-Type: application/json
```

### Request Body Format

Use JSON for request bodies:

```json
{
  "account": "user@example.com",
  "name": "John Doe",
  "imap": {
    "host": "imap.example.com",
    "port": 993,
    "secure": true,
    "auth": {
      "user": "user@example.com",
      "pass": "password"
    }
  }
}
```

### Example Requests

<Tabs groupId="language">
<TabItem value="curl" label="cURL" default>

```bash
curl http://localhost:3000/v1/accounts \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

</TabItem>
<TabItem value="nodejs" label="Node.js">

```javascript
const res = await fetch('http://localhost:3000/v1/accounts', {
  headers: { Authorization: 'Bearer YOUR_ACCESS_TOKEN' }
});

const { accounts } = await res.json();
```

</TabItem>
<TabItem value="python" label="Python">

```python
import requests

res = requests.get(
    'http://localhost:3000/v1/accounts',
    headers={'Authorization': 'Bearer YOUR_ACCESS_TOKEN'}
)

accounts = res.json()['accounts']
```

</TabItem>
<TabItem value="php" label="PHP">

```php
<?php
$ch = curl_init('http://localhost:3000/v1/accounts');
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_HTTPHEADER, ['Authorization: Bearer YOUR_ACCESS_TOKEN']);

$accounts = json_decode(curl_exec($ch), true)['accounts'];
curl_close($ch);
```

</TabItem>
</Tabs>

## Response Format

### Success Responses

Every successful request answers **200 OK** with a JSON body. The API does not use 201 or 204, so a creation reports what it created in the body of a 200.

Example success response, from registering an account:

```json
{
  "account": "user123",
  "state": "new"
}
```

Here `state` reports whether the account was created or already existed (`new` or `existing`), not its connection state. Read `/v1/account/{account}` for that.

### Error Responses

Error requests return HTTP status codes in the 4xx or 5xx range:

- **400 Bad Request**: Invalid request parameters
- **401 Unauthorized**: Missing or invalid authentication
- **403 Forbidden**: Insufficient permissions
- **404 Not Found**: Resource not found
- **422 Unprocessable Entity**: The request was understood but the mail server or provider refused the operation
- **429 Too Many Requests**: Rate limit exceeded
- **500 Internal Server Error**: Server error
- **503 Service Unavailable**: Service temporarily unavailable

Example error response:

```json
{
  "statusCode": 404,
  "error": "Not Found",
  "message": "Account not found",
  "code": "AccountNotFound"
}
```

### Response Structure

Successful responses are not wrapped in an envelope. Each endpoint returns its own object with the fields it documents, so read the [full API reference](/docs/api/emailengine-api) for the exact shape of a given endpoint.

Paginated endpoints share one convention, wrapping their results alongside pagination counters, with the array named after what is being listed:

```json
{
  "total": 472,
  "page": 0,
  "pages": 24,
  "nextPageCursor": "eyJhIjoxfQ",
  "prevPageCursor": null,
  "messages": []
}
```

Not every list is paginated. `GET /v1/account/{account}/mailboxes` returns the folder tree in one response, as just `{ "mailboxes": [] }`.

## Error Handling

### HTTP Status Codes

| Code | Meaning             | Action                       |
| ---- | ------------------- | ---------------------------- |
| 400  | Bad Request         | Check request parameters     |
| 401  | Unauthorized        | Verify authentication token  |
| 403  | Forbidden           | Check token permissions      |
| 404  | Not Found           | Verify resource exists       |
| 422  | Unprocessable Entity | The provider refused the operation, not the request shape |
| 429  | Too Many Requests   | Implement retry with backoff |
| 500  | Server Error        | Retry after delay            |
| 503  | Service Unavailable | Service restarting, retry    |

### Error Response Format

```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": "Invalid input",
  "fields": [
    { "message": "\"pageSize\" must be less than or equal to 1000", "key": "pageSize" }
  ]
}
```

| Field | Contents |
|-------|----------|
| `statusCode` | The HTTP status code, repeated in the body |
| `error` | The HTTP status text, such as `Bad Request` or `Unauthorized` |
| `message` | The human-readable reason. **This is the field to show or log**, not `error` |
| `code` | A machine-readable code, present when the failure has one. See [Error Codes](/docs/reference/error-codes) |
| `fields` | Present on validation failures, listing each rejected input as `key` and `message` |

### Common Error Codes

| Code                     | Description                             | Solution                        |
| ------------------------ | --------------------------------------- | ------------------------------- |
| `InvalidToken`           | Missing or invalid API token            | Provide a valid token           |
| `AccountNotFound`        | Account doesn't exist                   | Verify account ID               |
| `MessageNotFound`        | Message doesn't exist                   | Check message ID                |
| `AuthenticationFails`    | The mail server rejected the credentials | Re-authorize the account       |
| `ConnectionError`        | Can't connect to mail server            | Check host, port, and network   |
| `RateLimitExceeded`      | Too many requests                       | Back off for `ttl` seconds      |

Validation failures do not carry a `code`. They return `400` with a `fields` array instead. See [Error Codes](/docs/reference/error-codes) for the full list.

### Retry Strategies

For transient errors (429, 500, 503):

```javascript
async function requestWithRetry(url, options, maxRetries = 3) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    const res = await fetch(url, options);
    if (res.ok) return res;

    if (res.status !== 429 && res.status < 500) {
      // The request itself is wrong, so retrying it changes nothing
      const { message } = await res.json();
      throw new Error(message);
    }

    // A 429 tells you exactly how long to wait, so prefer it over guessing
    const body = await res.json().catch(() => ({}));
    const delay = body.ttl ? body.ttl * 1000 : 2 ** attempt * 1000;
    await new Promise(r => setTimeout(r, delay));
  }

  throw new Error('Max retries exceeded');
}
```

Retry only `429` and `5xx`. Every other `4xx` reports something about the request that will not change on its own.

## Pagination

For endpoints that return lists (accounts, messages, etc.), use pagination parameters:

### Parameters

| Parameter  | Type   | Default | Description                                         |
| ---------- | ------ | ------- | --------------------------------------------------- |
| `page`     | number | 0       | Page number (0-indexed)                             |
| `pageSize` | number | 20      | Items per page                                      |
| `cursor`   | string | -       | Paging cursor from nextPageCursor or prevPageCursor |

### Example Request

```bash
curl "http://localhost:3000/v1/account/user@example.com/messages?path=INBOX&page=0&pageSize=50" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

### Response Metadata

Paginated responses include navigation metadata:

```json
{
  "total": 523,
  "page": 0,
  "pages": 11,
  "messages": []
}
```

### Navigation Example

```javascript
async function* eachMessage(account, path = 'INBOX') {
  const base = `http://localhost:3000/v1/account/${encodeURIComponent(account)}/messages`;

  for (let page = 0; ; page++) {
    const res = await fetch(`${base}?path=${encodeURIComponent(path)}&page=${page}&pageSize=100`, {
      headers: { Authorization: 'Bearer YOUR_ACCESS_TOKEN' }
    });

    const { messages, pages } = await res.json();
    yield* messages;

    if (page >= pages - 1) return;
  }
}

for await (const message of eachMessage('user@example.com')) {
  console.log(message.id, message.subject);
}
```

:::caution Paging a mailbox that is changing
Pages are computed per request, so a message arriving mid-walk shifts everything down by one and you can see the same message twice or skip one. Deduplicate on `id` while walking, and prefer [webhooks](/docs/webhooks/overview) over re-listing to keep up with new mail.
:::

## Filtering & Search

### Query Parameters

The messages list endpoint supports basic query parameters:

```bash
# List messages in a specific mailbox
curl "http://localhost:3000/v1/account/user@example.com/messages?path=INBOX" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

### Search Syntax

Use the search endpoint for advanced queries including flag filtering:

```bash
# Search for unread messages from a specific sender
curl -X POST http://localhost:3000/v1/account/user@example.com/search \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "search": {
      "from": "sender@example.com",
      "subject": "Important",
      "unseen": true
    }
  }'
```

## Webhooks

Instead of polling the API, use webhooks to receive real-time notifications.

### Event-Driven Architecture

Webhooks provide instant notifications when:

- New messages arrive
- Messages are deleted or updated
- Accounts connect or disconnect
- Messages are sent or fail

### Benefits

- **Real-time**: Immediate notification of events
- **Efficient**: No polling overhead
- **Scalable**: Handles high-volume accounts
- **Reliable**: Automatic retry logic

### Setup

Register a webhook endpoint:

```bash
curl -X POST http://localhost:3000/v1/settings \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "webhooks": "https://your-app.com/webhook",
    "webhookEvents": ["messageNew", "messageSent"]
  }'
```

Learn more in the [Webhooks API documentation](./webhooks-api.md).

## Quick Start Example

Register an account, send a message, and subscribe to new mail. Each step is one request:

```javascript
const BASE = 'http://localhost:3000/v1';
const TOKEN = 'YOUR_ACCESS_TOKEN';

const api = (path, body) =>
  fetch(BASE + path, {
    method: body ? 'POST' : 'GET',
    headers: { Authorization: `Bearer ${TOKEN}`, 'Content-Type': 'application/json' },
    body: body && JSON.stringify(body)
  }).then(res => res.json());

// 1. Register an account. EmailEngine verifies the credentials and starts syncing
const { account } = await api('/account', {
  account: 'user@example.com',
  name: 'John Doe',
  imap: {
    host: 'imap.example.com', port: 993, secure: true,
    auth: { user: 'user@example.com', pass: 'password' }
  },
  smtp: {
    host: 'smtp.example.com', port: 465, secure: true,
    auth: { user: 'user@example.com', pass: 'password' }
  }
});

// 2. Subscribe to new mail before it can arrive
await api('/settings', {
  webhooks: 'https://your-app.com/webhook',
  webhookEvents: ['messageNew']
});

// 3. Send a message
const { messageId } = await api(`/account/${encodeURIComponent(account)}/submit`, {
  to: [{ address: 'recipient@example.com' }],
  subject: 'Hello from EmailEngine',
  text: 'This is a test email sent via the API'
});

// 4. Read the mailbox
const { messages } = await api(`/account/${encodeURIComponent(account)}/messages?path=INBOX&pageSize=10`);
```

Registering an account returns as soon as the account is stored, so it is normal for the first `/messages` call to come back empty while the initial sync runs. Rather than sleeping, subscribe to the [`accountInitialized`](/docs/webhooks/accountinitialized) webhook, which fires once the first sync completes.

## API Categories

The EmailEngine API is organized into these main categories:

### Accounts API

Manage email accounts, credentials, and connections.

- Register and delete accounts
- Update account settings
- Monitor account status
- Handle OAuth2 authentication

[View Accounts API documentation](./accounts-api.md)

### Messages API

Read, search, and manage email messages.

- List and filter messages
- Get message details and source
- Update message flags
- Move and delete messages
- Search messages

[View Messages API documentation](./messages-api.md)

### Sending API

Send emails with attachments and templates.

- Send immediate emails (Submit API)
- Queue emails for later (Outbox API)
- Handle replies and forwards
- Track delivery status

[View Sending API documentation](./sending-api.md)

### Webhooks API

Configure webhooks and event notifications.

- Register webhook endpoints
- Filter events
- Secure webhooks
- Monitor webhook delivery

[View Webhooks API documentation](./webhooks-api.md)

### MCP Endpoint

Not part of the REST API, but the same functionality for a different caller: `POST /mcp` serves the Model Context Protocol, so an AI agent can call a curated tool set with an access token you narrow and revoke like any other.

- 15 tools over accounts, folders, messages, sending and templates
- Every tool call is dispatched as the equivalent REST request, with the same permission checks
- Off by default

[View MCP documentation](/docs/mcp)

### Full API Reference

Complete auto-generated API documentation with all endpoints, parameters, and examples.

[Browse full API reference](/docs/api/emailengine-api)

### OpenAPI Specification

The machine-readable document behind the reference above, served by every EmailEngine instance at `/swagger.json`.

- Import the API into Postman, Insomnia, or Bruno
- Generate a typed client in your language
- Feed the API surface to AI coding assistants

[View OpenAPI documentation](./openapi-spec.md)

## Support

- **Documentation**: Browse the complete [documentation](/docs)
- **GitHub Issues**: Report bugs or request features
- **Community**: Join discussions on GitHub
- **Professional Support**: Contact for enterprise support options

## See Also

- [Accounts API](/docs/api-reference/accounts-api) - Registering and managing accounts
- [Messages API](/docs/api-reference/messages-api) - Reading, searching, and modifying mail
- [Sending API](/docs/api-reference/sending-api) - The submit endpoint and its options
- [Access tokens](/docs/api-reference/access-tokens) - Scopes, restrictions, and the audit log
- [Full endpoint reference](/docs/api/emailengine-api) - Every endpoint with its schemas
