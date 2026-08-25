---
title: Accounts API
description: API endpoints for managing email accounts - register, update, delete, and retrieve account details
sidebar_position: 2
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Accounts API

The Accounts API allows you to programmatically manage email accounts in EmailEngine. You can register new accounts, update settings, monitor connection status, and handle OAuth2 authentication.

## Overview

Email accounts are the core resource in EmailEngine. Each account represents a connection to an email service (Gmail, Outlook, IMAP/SMTP server) and maintains:

- Connection credentials (OAuth2 tokens or passwords)
- Mailbox synchronization state
- Account-specific settings
- Connection status and health

### Account Object Structure

```json
{
  "account": "user@example.com",
  "name": "John Doe",
  "email": "user@example.com",
  "state": "connected",
  "syncTime": "2025-01-15T10:30:00.000Z",
  "notifyFrom": "2025-01-01T00:00:00.000Z",
  "lastError": null,
  "imap": {
    "host": "imap.gmail.com",
    "port": 993,
    "secure": true,
    "disabled": false
  },
  "smtp": {
    "host": "smtp.gmail.com",
    "port": 465,
    "secure": true,
    "disabled": false
  },
  "type": "gmail",
  "oauth2": {
    "provider": "AAABhaBPHscAAAAH",
    "auth": {
      "user": "user@example.com"
    }
  }
}
```

The `state` field reports where the connection stands. See [Account States](#account-states) for the full list and what each one means for your application.

## Common Operations

### 1. Register Account

Register a new email account with EmailEngine.

**Endpoint:** `POST /v1/account`

**Request Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `account` | string | Yes | Unique account identifier (usually email address) |
| `name` | string | Yes | Display name for the account |
| `email` | string | No | Email address (defaults to `account`) |
| `imap` | object | No | IMAP connection settings |
| `smtp` | object | No | SMTP connection settings |
| `oauth2` | object | No | OAuth2 settings |
| `notifyFrom` | string | No | ISO date to send webhooks from (default: account creation time) |

Only `account` and `name` are required by the schema. In practice, provide either `imap` (usually together with `smtp`) or `oauth2` so the account can actually connect to a mail server.

**IMAP Configuration:**

```json
{
  "host": "imap.example.com",
  "port": 993,
  "secure": true,
  "auth": {
    "user": "username",
    "pass": "password"
  }
}
```

**SMTP Configuration:**

```json
{
  "host": "smtp.example.com",
  "port": 465,
  "secure": true,
  "auth": {
    "user": "username",
    "pass": "password"
  }
}
```

**Examples:**

<Tabs groupId="programming-language">
<TabItem value="curl" label="cURL">

```bash
curl -X POST http://localhost:3000/v1/account \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
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
  }'
```

</TabItem>
<TabItem value="python" label="Python">

```python
import requests

response = requests.post(
    'http://localhost:3000/v1/account',
    headers={
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
    },
    json={
        'account': 'user@example.com',
        'name': 'John Doe',
        'imap': {
            'host': 'imap.example.com',
            'port': 993,
            'secure': True,
            'auth': {
                'user': 'user@example.com',
                'pass': 'password'
            }
        }
    }
)

result = response.json()
print(f"Account registered: {result['account']}")
```

</TabItem>
</Tabs>

**Response:**
```json
{
  "account": "user@example.com",
  "state": "new"
}
```

The `state` field in this response indicates whether the account was created (`new`) or an existing account with the same ID was updated (`existing`).

**Use Cases:**
- Onboarding new users to your application
- Allowing users to connect multiple email accounts
- Automated account provisioning in bulk

[Detailed API reference →](/docs/api/post-v-1-account)

---

### 2. List Accounts

Retrieve all registered accounts.

**Endpoint:** `GET /v1/accounts`

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `page` | number | Page number (0-indexed) |
| `pageSize` | number | Items per page (default 20) |
| `state` | string | Filter by account state |
| `query` | string | Filter accounts by string match |

**Examples:**

```bash
curl "http://localhost:3000/v1/accounts?pageSize=50" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

**Response:**
```json
{
  "total": 2,
  "page": 0,
  "pages": 1,
  "accounts": [
    {
      "account": "user1@example.com",
      "name": "John Doe",
      "email": "user1@example.com",
      "state": "connected"
    },
    {
      "account": "user2@example.com",
      "name": "Jane Smith",
      "email": "user2@example.com",
      "state": "authenticationError"
    }
  ]
}
```

**Use Cases:**
- Dashboard displaying all connected accounts
- Health monitoring across accounts
- Bulk operations on multiple accounts

[Detailed API reference →](/docs/api/get-v-1-accounts)

---

### 3. Get Account Details

Retrieve detailed information about a specific account.

**Endpoint:** `GET /v1/account/:account`

**Path Parameters:**

| Parameter | Description |
|-----------|-------------|
| `account` | Account identifier |

**Examples:**

```bash
curl "http://localhost:3000/v1/account/user@example.com" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

**Response:**
```json
{
  "account": "user@example.com",
  "name": "John Doe",
  "email": "user@example.com",
  "state": "connected",
  "syncTime": "2025-01-15T10:30:00.000Z",
  "lastError": null,
  "counters": {
    "events": {
      "messageNew": 30,
      "messageDeleted": 5
    }
  }
}
```

**Use Cases:**
- Displaying account status in user interface
- Checking connection health
- Retrieving account statistics

[Detailed API reference →](/docs/api/get-v-1-account-account)

---

### 4. Update Account

Update account settings or credentials.

**Endpoint:** `PUT /v1/account/:account`

**Request Body:**
```json
{
  "name": "New Display Name",
  "imap": {
    "partial": true,
    "port": 993
  },
  "smtp": {
    "partial": true,
    "port": 465
  }
}
```

:::tip Partial Updates
Use `"partial": true` inside `imap`, `smtp`, or `oauth2` objects to update only the specified fields instead of replacing the entire configuration. Without this flag, the entire object will be replaced, potentially losing existing settings.

**Note:** The `partial` flag only works for main-level objects (`imap`, `smtp`, `oauth2`), not for nested objects like `imap.auth`.
:::

**Examples:**

```bash
curl -X PUT "http://localhost:3000/v1/account/user@example.com" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Updated Name",
    "imap": {
      "partial": true,
      "port": 993
    }
  }'
```

**Use Cases:**
- Updating account credentials after password change
- Changing display names
- Modifying connection settings

[Detailed API reference →](/docs/api/put-v-1-account-account)

---

### 5. Delete Account

Remove an account and stop synchronization.

**Endpoint:** `DELETE /v1/account/:account`

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `revoke` | boolean | If `true`, EmailEngine attempts to revoke the upstream OAuth2 grant at the provider before deleting the account. Currently supported for individual Gmail OAuth grants; for Gmail service-account integrations, Outlook, and non-OAuth2 accounts the flag is a no-op. Revoke failures are logged and do not block deletion. Default: `false` |

**Examples:**

```bash
curl -X DELETE "http://localhost:3000/v1/account/user@example.com" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"

# Also revoke the OAuth2 grant at the provider (Gmail OAuth accounts)
curl -X DELETE "http://localhost:3000/v1/account/user@example.com?revoke=true" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

**Response:**
```json
{
  "account": "user@example.com",
  "deleted": true
}
```

**Use Cases:**
- User disconnecting their email account
- Removing inactive accounts
- Cleanup during offboarding

[Detailed API reference →](/docs/api/delete-v-1-account-account)

---

### 6. Reconnect Account

Force reconnection to mail server (useful after credential updates).

**Endpoint:** `PUT /v1/account/:account/reconnect`

**Examples:**

```bash
curl -X PUT "http://localhost:3000/v1/account/user@example.com/reconnect" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"reconnect": true}'
```

**Use Cases:**
- Testing connection after credential update
- Recovering from connection errors
- Manual reconnection trigger

[Detailed API reference →](/docs/api/put-v-1-account-account-reconnect)

---

### 7. Account Operations: Reconnect vs Sync vs Flush

EmailEngine provides three distinct operations for managing account connections, each suited for different scenarios.

#### Reconnect

**Endpoint:** `PUT /v1/account/:account/reconnect`

Closes the existing IMAP connection entirely and opens a new one. This is a full disconnect/reconnect cycle.

```bash
curl -X PUT "http://localhost:3000/v1/account/user@example.com/reconnect" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"reconnect": true}'
```

**When to use:**
- After updating account credentials (password change, OAuth2 token update)
- To recover from persistent connection errors
- When you need a fresh IMAP session (e.g., after server-side configuration changes)

[Detailed API reference -->](/docs/api/put-v-1-account-account-reconnect)

---

#### Sync

**Endpoint:** `PUT /v1/account/:account/sync`

Triggers an immediate mailbox synchronization without disconnecting. Refreshes the folder list and syncs all monitored mailboxes.

```bash
curl -X PUT "http://localhost:3000/v1/account/user@example.com/sync" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"sync": true}'
```

**When to use:**
- When you need the latest messages immediately without waiting for the next poll cycle
- After bulk operations on the mail server (e.g., importing messages via another client)
- To ensure webhooks are triggered for recently arrived messages

[Detailed API reference -->](/docs/api/put-v-1-account-account-sync)

---

#### Flush

**Endpoint:** `PUT /v1/account/:account/flush`

Deletes all cached email data (message indexes, folder lists, bounce data) from Redis and Elasticsearch, then triggers a full re-sync from scratch. The account is paused during the operation and automatically resumed after completion.

```bash
curl -X PUT "http://localhost:3000/v1/account/user@example.com/flush" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"flush": true}'
```

:::warning Destructive Operation
Flush deletes all cached data for the account. This triggers a complete re-indexing of all messages, which may take significant time for large mailboxes and will re-trigger `messageNew` webhooks unless `notifyFrom` is set appropriately. Only one flush operation can run at a time across all accounts.
:::

**When to use:**
- To fix data corruption or synchronization issues
- After major mailbox reorganization on the server
- When message listings show stale or incorrect data
- As a last resort when reconnect and sync don't resolve issues

[Detailed API reference -->](/docs/api/put-v-1-account-account-flush)

---

#### Comparison

| Operation | Connection | Data | Duration | Impact |
|-----------|-----------|------|----------|--------|
| **Reconnect** | Closes and reopens | Preserved | Seconds | Brief interruption |
| **Sync** | Stays connected | Preserved | Seconds to minutes | No interruption |
| **Flush** | Paused temporarily | Deleted and rebuilt | Minutes to hours | Re-indexes everything |

**Decision guide:**
1. Try **sync** first - it's the least disruptive way to get fresh data
2. Use **reconnect** if the connection itself seems broken or after credential changes
3. Use **flush** only when cached data is corrupted or fundamentally out of sync

---

### 8. Verify Account Credentials

Pre-validate IMAP and SMTP credentials before creating an account.

**Endpoint:** `POST /v1/verifyAccount`

Tests IMAP and/or SMTP connections without creating or modifying any account. Both protocols are tested in parallel.

```bash
curl -X POST "http://localhost:3000/v1/verifyAccount" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
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
      "port": 587,
      "secure": false,
      "auth": {
        "user": "john@example.com",
        "pass": "password"
      }
    }
  }'
```

**Success response:**
```json
{
  "imap": { "success": true },
  "smtp": { "success": true }
}
```

**Failure response (bad IMAP credentials):**
```json
{
  "imap": {
    "success": false,
    "error": "Authentication failed",
    "code": "AUTHENTICATIONFAILED",
    "responseText": "NO [AUTHENTICATIONFAILED] Invalid credentials"
  },
  "smtp": { "success": true }
}
```

**Use Cases:**
- Validate credentials in your onboarding flow before creating the account
- Build a "test connection" button in your UI
- Verify server settings returned by the autoconfig endpoint

[Detailed API reference -->](/docs/api/post-v-1-verifyaccount)

---

## Account Object Reference

### Complete Field Reference

| Field | Type | Description |
|-------|------|-------------|
| `account` | string | Unique account identifier |
| `name` | string | Display name |
| `email` | string | Email address |
| `state` | string | Connection state (see Account States) |
| `syncTime` | string | ISO 8601 date-time of the last sync |
| `notifyFrom` | string | ISO date to send webhooks from |
| `lastError` | object | Last error details (if any) |
| `imap` | object | IMAP connection settings |
| `smtp` | object | SMTP connection settings |
| `oauth2` | object | OAuth2 configuration |
| `counters` | object | Cumulative event counters (`counters.events`) for the account lifetime |

### Account States

| State | Meaning |
|-------|---------|
| `init` | The account was just registered and has not connected yet |
| `unset` | No IMAP or OAuth2 configuration is set, so the account is not synced |
| `connecting` | Connecting to the mail server or authorizing with the provider |
| `syncing` | Connected and performing the initial or a periodic mailbox sync |
| `connected` | Connected and watching for changes. This is the healthy steady state |
| `disconnected` | The connection dropped and EmailEngine is retrying with backoff |
| `connectError` | The server could not be reached or the TLS handshake failed. Retried with backoff |
| `authenticationError` | The credentials were rejected. Requires re-authentication before syncing resumes |
| `paused` | Syncing was paused through the API. No connection is maintained |

`connected` and `syncing` are both healthy. Treat `connecting`, `syncing`, and `disconnected` as transient and let EmailEngine recover on its own. Only `authenticationError` always needs a human or an OAuth2 re-authorization; a `connectError` that persists usually points at the network or the server rather than at the account.

Watch these transitions live instead of polling with the account state stream below.

### Streaming Account State Changes

`GET /v1/changes` is a [Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events) stream that pushes a message every time an account changes state. It is the same feed the admin dashboard uses to repaint its status badges, so a monitoring view can follow every account without polling `/v1/accounts` on a timer.

```javascript
const stream = new EventSource(
  'http://localhost:3000/v1/changes?access_token=YOUR_ACCESS_TOKEN'
);

stream.onmessage = event => {
  const { account, type, key, payload } = JSON.parse(event.data);
  if (type === 'state') {
    console.log(`${account} is now ${key}`);
  }
};
```

Each message carries:

| Field | Meaning |
|-------|---------|
| `account` | The account that changed, or `null` for instance-wide events |
| `type` | `state` for an account transition. `smtpServerState` and `imapProxyServerState` report the built-in servers starting or failing |
| `key` | The new value, so for `type: "state"` one of the account states above |
| `payload` | Extra context, such as `error` on a failure. `null` when there is none |
| `stateLabel` | A pre-rendered, translated badge label, resolved server-side so the admin UI and any client agree |

The connection stays open until the client closes it, so treat a disconnect as normal and reconnect. `EventSource` does this on its own.

:::note This stream is a signal, not a record
Events are only delivered to clients connected at the time. Nothing is buffered for a client that is not listening, so a state change during a reconnect is missed. Use it to drive a live view, and read `/v1/account/{account}` for the authoritative current state. For durable delivery, use [webhooks](/docs/webhooks/overview).
:::

Because `EventSource` cannot set request headers, this endpoint accepts the token as the `access_token` query parameter. Keep in mind that query strings are more likely to end up in proxy and server logs than an `Authorization` header.

## Common Patterns

### Bulk Account Registration

Register accounts with bounded concurrency rather than firing every request at once. Each registration opens a connection to a mail server, so an unbounded loop over thousands of accounts will hit provider connection limits before it hits EmailEngine's:

```javascript
async function registerAccounts(accounts, concurrency = 5) {
  const queue = [...accounts];
  const results = [];

  const worker = async () => {
    while (queue.length) {
      const body = queue.shift();
      const res = await fetch('http://localhost:3000/v1/account', {
        method: 'POST',
        headers: {
          Authorization: 'Bearer YOUR_ACCESS_TOKEN',
          'Content-Type': 'application/json'
        },
        body: JSON.stringify(body)
      });

      results.push(
        res.ok
          ? { account: body.account, ...(await res.json()) }
          : { account: body.account, error: (await res.json()).message }
      );
    }
  };

  await Promise.all(Array.from({ length: concurrency }, worker));
  return results;
}
```

Registration returns as soon as the account is stored, so a successful response does not mean the mailbox is reachable yet. Watch for [`authenticationError`](/docs/webhooks/authenticationerror) and [`accountInitialized`](/docs/webhooks/accountinitialized) webhooks to learn the real outcome.

See [Performance Tuning](/docs/advanced/performance-tuning) before onboarding accounts in the thousands.

### Health Monitoring

Ask EmailEngine for the accounts in a given state rather than listing everything and filtering client-side:

```bash
curl "http://localhost:3000/v1/accounts?state=authenticationError&pageSize=1000" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

`authenticationError` is the state worth alerting on: it does not clear on its own and needs a credential update or a new OAuth2 authorization. `connectError` and `disconnected` are retried automatically, so alert on those only if an account stays there across several checks, and treat `connecting`, `syncing`, and `connected` as healthy.

For a live view rather than a poll, subscribe to the [account state stream](#streaming-account-state-changes).

### Credential Rotation

Send only the fields that change. Setting `imap.partial` keeps the rest of the IMAP configuration intact, so you do not have to resend host, port, and TLS settings just to replace a password:

```json
{
  "imap": {
    "partial": true,
    "auth": { "user": "user@example.com", "pass": "new-password" }
  }
}
```

Updating credentials on an account in an error state triggers a reconnect, so there is no need to call reconnect separately.

### Account Synchronization Status

`GET /v1/account/{account}` reports where an account stands:

```bash
curl "http://localhost:3000/v1/account/user%40example.com" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

| Field | What it tells you |
|-------|-------------------|
| `state` | The current connection state, from the table above |
| `syncTime` | When the last sync completed. A timestamp that stops advancing points at a stalled account |
| `lastError` | Details of the most recent failure, or `null`. Present even after EmailEngine has recovered |

## Error Handling

### Common Errors

**Account Already Exists:**
```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": "Another account for the same OAuth2 user already exists",
  "code": "AccountAlreadyExists",
  "existingAccount": "user123"
}
```
**Solution:** Use PUT to update existing account or choose different account ID.

**Authentication Failed:**

`POST /v1/account` does not validate credentials at registration time - invalid credentials surface later when the account moves to the `authenticationError` state. To check credentials up front, use `POST /v1/verifyAccount`, which reports the server response:
```json
{
  "imap": {
    "success": false,
    "error": "Authentication failed",
    "code": "AUTHENTICATIONFAILED",
    "responseText": "NO [AUTHENTICATIONFAILED] Invalid credentials"
  },
  "smtp": { "success": true }
}
```
**Solution:** Verify credentials, check if 2FA/app passwords are required.

**Account Not Found:**
```json
{
  "statusCode": 404,
  "error": "Not Found",
  "message": "Account record was not found for requested ID"
}
```
**Solution:** Verify account ID is correct and account exists.

### Troubleshooting

For accounts stuck in error states:

1. Check `lastError` field for details
2. Verify credentials are current
3. Test connection with manual reconnect
4. Check mail server accessibility
5. Review OAuth2 token expiration
