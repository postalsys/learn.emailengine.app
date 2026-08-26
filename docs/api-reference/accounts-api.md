---
title: Accounts API
description: API endpoints for managing email accounts - register, update, delete, and retrieve account details
sidebar_position: 3
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
  "authFailureDisabledAt": null,
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

The `state` field reports where the connection stands. See [Account States](#account-states) for the full list and what each one means for your application. `authFailureDisabledAt` is `null` unless EmailEngine itself switched syncing off after repeated authentication failures, see [Accounts switched off automatically](#accounts-switched-off-automatically).

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
curl -X POST https://emailengine.example.com/v1/account \
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
    'https://emailengine.example.com/v1/account',
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

The `state` field in this response indicates whether the account was created (`new`) or an existing account with the same ID was updated (`existing`). Credentials are not verified while handling this request: the account connects afterwards, and a bad password surfaces as the `authenticationError` state rather than as an error here.

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
| `page` | number | Page number (0-indexed, default 0) |
| `pageSize` | number | Items per page (default 20) |
| `state` | string | Filter by account state |
| `query` | string | Filter accounts by string match |

Each entry carries a subset of the account object: `account`, `name`, `email`, `type`, `app`, `state`, `webhooks`, `proxy`, `smtpEhloName`, `counters`, `syncTime`, `authFailureDisabledAt`, `lastError` and, for delegated accounts, `delegationError`.

**Examples:**

```bash
curl "https://emailengine.example.com/v1/accounts?pageSize=50" \
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
curl "https://emailengine.example.com/v1/account/user@example.com" \
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
curl -X PUT "https://emailengine.example.com/v1/account/user@example.com" \
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
curl -X DELETE "https://emailengine.example.com/v1/account/user@example.com" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"

# Also revoke the OAuth2 grant at the provider (Gmail OAuth accounts)
curl -X DELETE "https://emailengine.example.com/v1/account/user@example.com?revoke=true" \
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
curl -X PUT "https://emailengine.example.com/v1/account/user@example.com/reconnect" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"reconnect": true}'
```

**Response:**
```json
{
  "reconnect": true
}
```

The request only schedules the reconnect and returns before it finishes. Watch the account state, or the `accountInitialized` and `authenticationError` webhooks, for the result. `reconnect` is `false` when the body did not ask for one, and also when the account has been [switched off after authentication failures](#accounts-switched-off-automatically): nothing is scheduled then, because the same credentials would fail again. Since EmailEngine 2.79.4; earlier releases answered `true` and did nothing.

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
curl -X PUT "https://emailengine.example.com/v1/account/user@example.com/reconnect" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"reconnect": true}'
```

**When to use:**
- After updating account credentials (password change, OAuth2 token update)
- To recover from persistent connection errors
- When you need a fresh IMAP session (e.g., after server-side configuration changes)

[Detailed API reference →](/docs/api/put-v-1-account-account-reconnect)

---

#### Sync

**Endpoint:** `PUT /v1/account/:account/sync`

Triggers an immediate mailbox synchronization without disconnecting. Refreshes the folder list and syncs all monitored mailboxes.

```bash
curl -X PUT "https://emailengine.example.com/v1/account/user@example.com/sync" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"sync": true}'
```

The response is `{"sync": true}`. Like reconnect, the request schedules the work and returns before it finishes.

**When to use:**
- When you need the latest messages immediately without waiting for the next poll cycle
- After bulk operations on the mail server (e.g., importing messages via another client)
- To ensure webhooks are triggered for recently arrived messages

[Detailed API reference →](/docs/api/put-v-1-account-account-sync)

---

#### Flush

**Endpoint:** `PUT /v1/account/:account/flush`

Deletes all cached email data (message indexes, folder lists, bounce data) from Redis, and from the index of the deprecated Document Store when that is enabled, then triggers a full re-sync from scratch. The account is paused during the operation and automatically resumed after completion.

```bash
curl -X PUT "https://emailengine.example.com/v1/account/user@example.com/flush" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"flush": true}'
```

The response is `{"flush": true}`. The body also accepts `notifyFrom`, to set the webhook cutoff for the rebuilt index, and `imapIndexer`, to switch the indexing strategy at the same time.

:::warning Destructive Operation
Flush deletes all cached data for the account. This triggers a complete re-indexing of all messages, which may take significant time for large mailboxes and will re-trigger `messageNew` webhooks unless `notifyFrom` is set appropriately. Only one flush operation can run at a time across all accounts; a second request while one is running answers `429` with `One flush operation at a time allowed, try again later`.
:::

**When to use:**
- To fix data corruption or synchronization issues
- After major mailbox reorganization on the server
- When message listings show stale or incorrect data
- As a last resort when reconnect and sync don't resolve issues

[Detailed API reference →](/docs/api/put-v-1-account-account-flush)

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
curl -X POST "https://emailengine.example.com/v1/verifyAccount" \
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

[Detailed API reference →](/docs/api/post-v-1-verifyaccount)

---

## Account Object Reference

### Complete Field Reference

| Field | Type | Description |
|-------|------|-------------|
| `account` | string | Unique account identifier |
| `name` | string | Display name |
| `email` | string | Email address |
| `type` | string | How the account connects: `imap`, `gmail`, `gmailService`, `outlook`, `outlookService` or `mailRu`. `oauth2` is an OAuth2 account whose application is no longer configured in EmailEngine, `delegated` a shared mailbox reached through another account's credentials, `sending` a send-only account with no IMAP access, and `invalid` a delegated account whose delegation could not be resolved |
| `app` | string | OAuth2 application ID, for OAuth2 accounts |
| `state` | string | Connection state (see Account States) |
| `syncTime` | string | ISO 8601 date-time of the last sync (IMAP accounts only) |
| `notifyFrom` | string | ISO date to send webhooks from |
| `lastError` | object | Last error details, or `null` |
| `authFailureDisabledAt` | string | When EmailEngine switched syncing off after repeated authentication failures, or `null`. Read-only |
| `syncError` | object | The last mailbox sync error (IMAP accounts only) |
| `smtpStatus` | object | The last SMTP connection attempt, or `null` if none was made |
| `connections` | number | Open IMAP connections for this account |
| `sendOnly` | boolean | `true` for an account that does not sync messages |
| `imap` | object | IMAP connection settings. `imap.disabled` is the operator's own switch for turning syncing off |
| `smtp` | object | SMTP connection settings |
| `oauth2` | object | OAuth2 configuration |
| `path` | array | Mailbox folders to monitor (IMAP only), `"*"` for all |
| `subconnections` | array | Folders monitored with a dedicated IMAP connection each |
| `webhooks` | string | Account-specific webhook URL |
| `webhooksCustomHeaders` | array | Extra headers sent with every webhook for this account |
| `proxy` | string | Proxy URL for outbound connections |
| `smtpEhloName` | string | Hostname used in SMTP EHLO |
| `locale`, `tz` | string | Default locale and timezone for content rendered for this account |
| `counters` | object | Cumulative event counters (`counters.events`) for the account lifetime |
| `quota` | object | Mailbox quota, only with `?quota=true`; `false` if the server reports none |
| `outlookSubscription` | object | Microsoft Graph change subscription details (Outlook accounts only) |

The [endpoint reference](/docs/api/get-v-1-account-account) documents every nested field. Stored passwords and OAuth2 tokens are masked in the response.

### Account States

| State | Meaning |
|-------|---------|
| `init` | The account was just registered and has not connected yet |
| `unset` | The account is not syncing: either no IMAP or OAuth2 configuration is set, or syncing was switched off, by the operator or automatically after repeated authentication failures |
| `connecting` | Connecting to the mail server or authorizing with the provider |
| `syncing` | Connected and performing the initial or a periodic mailbox sync |
| `connected` | Connected and watching for changes. This is the healthy steady state |
| `disconnected` | The connection dropped and EmailEngine is retrying with backoff |
| `connectError` | The server could not be reached or the TLS handshake failed. Retried with backoff |
| `authenticationError` | The credentials were rejected. Requires re-authentication before syncing resumes |
| `paused` | Syncing was paused through the API. No connection is maintained |

`connected` and `syncing` are both healthy. Treat `connecting`, `syncing`, and `disconnected` as transient and let EmailEngine recover on its own. Only `authenticationError` always needs a human or an OAuth2 re-authorization; a `connectError` that persists usually points at the network or the server rather than at the account.

Watch these transitions live instead of polling with the account state stream below.

### Accounts switched off automatically

An account that keeps failing to authenticate for longer than `EENGINE_MAX_IMAP_AUTH_FAILURE_TIME` (3 days by default) is switched off: EmailEngine stops connecting, the account reports the `unset` state, and `authFailureDisabledAt` records when that happened. Since EmailEngine 2.79.3 this applies to OAuth2 accounts as well, so a revoked refresh token is no longer retried indefinitely.

The operator's own switch, `imap.disabled`, puts an account in the same `unset` state, so `authFailureDisabledAt` is what tells an automatic disable from a deliberate one: an account with `authFailureDisabledAt` set was parked by EmailEngine, one with `imap.disabled` and no timestamp was disabled on purpose.

Supplying working credentials lifts the switch and reconnects the account: `PUT /v1/account/{account}` with new `imap` settings, or a new OAuth2 authorization through the [hosted authentication form](/docs/accounts/hosted-authentication). In 2.79.4 a partial IMAP update that only changes the password keeps the flag unless it also sets `disabled` to `false`; 2.79.5 lifts it on the changed `imap.auth` alone. The admin account page shows the reason and the time, with a **Resume syncing** button. Calling reconnect on a parked account does nothing and answers `{"reconnect": false}`.

The field, both recovery paths and the reconnect response date from EmailEngine 2.79.4. 2.79.3 recorded the park only as `imap.disabled`, so re-authorizing an OAuth2 account it had parked reported success without lifting anything. 2.79.5 marks those OAuth2 accounts on its first start so that they become recoverable; their `authFailureDisabledAt` then holds the time of that start rather than the original park. 2.79.5 also stops parking delegated accounts, since the owner of the borrowed credential is what gets switched off, and stop reporting a switched-off account as send-only in the accounts listing and the admin interface, though this endpoint still answers `sending` with `sendOnly: true` for one.

See [Accounts switched off after authentication failures](/docs/accounts/managing-accounts#accounts-switched-off-after-authentication-failures) for the operator's view.

### Streaming Account State Changes

`GET /v1/changes` is a [Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events) stream that pushes a message every time an account changes state. It is the same feed the admin dashboard uses to repaint its status badges, so a monitoring view can follow every account without polling `/v1/accounts` on a timer.

```javascript
const stream = new EventSource(
  'https://emailengine.example.com/v1/changes?access_token=YOUR_ACCESS_TOKEN'
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
      const res = await fetch('https://emailengine.example.com/v1/account', {
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
curl "https://emailengine.example.com/v1/accounts?state=authenticationError&pageSize=1000" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

`authenticationError` is the state worth alerting on: it does not clear on its own and needs a credential update or a new OAuth2 authorization. So is `unset` with a non-null `authFailureDisabledAt`, which is where an account lands once those failures have gone on for days. `connectError` and `disconnected` are retried automatically, so alert on those only if an account stays there across several checks, and treat `connecting`, `syncing`, and `connected` as healthy.

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

Updating credentials through this endpoint triggers a reconnect, so there is no need to call reconnect separately. It also lifts the automatic switch-off described under [Account States](#accounts-switched-off-automatically).

### Account Synchronization Status

`GET /v1/account/{account}` reports where an account stands:

```bash
curl "https://emailengine.example.com/v1/account/user%40example.com" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

| Field | What it tells you |
|-------|-------------------|
| `state` | The current connection state, from the table above |
| `syncTime` | When the last sync completed. A timestamp that stops advancing points at a stalled account |
| `lastError` | Details of the most recent failure, or `null`. Present even after EmailEngine has recovered |
| `authFailureDisabledAt` | `null`, or the time EmailEngine gave up on the account's credentials and switched syncing off |

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

## See Also

- [Managing accounts](/docs/accounts/managing-accounts) - The same operations as a guide
- [Account types](/docs/accounts) - Choosing a backend before registering
- [Hosted authentication](/docs/accounts/hosted-authentication) - Letting the user supply the credentials
- [Account troubleshooting](/docs/accounts/troubleshooting) - When an account will not connect
