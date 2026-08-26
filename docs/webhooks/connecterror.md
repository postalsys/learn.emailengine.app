---
title: "connectError"
sidebar_position: 20
description: "Webhook event triggered when EmailEngine fails to establish a connection to an email server"
---

# connectError

The `connectError` webhook event is triggered when EmailEngine fails to establish an IMAP connection to a mail server. It covers network-level and server-level failures, as distinct from a login that the server rejected, which is [`authenticationError`](/docs/webhooks/authenticationerror).

## When This Event is Triggered

The `connectError` event fires when an IMAP connection attempt fails for a reason other than rejected credentials:

- The server is unreachable (network timeout, DNS failure)
- The server refuses the connection (port closed, firewall blocking)
- The TLS handshake fails
- The server returns an error before authentication, or drops the connection during it

EmailEngine reports the **first occurrence** of a failure and suppresses repeats until the error changes or the account connects. See [Webhook Deduplication](#webhook-deduplication).

A connection attempt that EmailEngine itself interrupted, because the account was paused, deleted or reconfigured while it was connecting, does not fire the event.

:::note IMAP accounts only
Only the IMAP client emits `connectError`. Gmail API and Microsoft Graph accounts reach their provider over HTTPS, and a transport failure there is retried; if the provider rejects the credentials instead, [`authenticationError`](/docs/webhooks/authenticationerror) is sent. Do not rely on `connectError` to detect provider outages for API-based accounts.
:::

## Common Use Cases

- **Infrastructure monitoring** - Detect when email servers become unavailable
- **Network diagnostics** - Identify connectivity issues between EmailEngine and mail servers
- **Account health dashboards** - Display connection status in your application
- **Automated alerting** - Notify administrators of server outages
- **SLA tracking** - Monitor uptime and connection reliability

## Payload Schema

### Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `serviceUrl` | string or null | Yes | The configured EmailEngine service URL. `null` when the `serviceUrl` setting is empty |
| `account` | string | Yes | Account ID that experienced the connection failure |
| `date` | string | Yes | ISO 8601 timestamp when the webhook was generated |
| `event` | string | Yes | Always `connectError` |
| `data` | object | Yes | Error details (see below) |

### Error Data Fields (`data` object)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `response` | string | Yes | The server's response text if it sent one, otherwise the error message from the connection attempt |
| `serverResponseCode` | string | No | The server's response code if it sent one, otherwise the error code from the connection attempt, such as Node.js's `ECONNREFUSED`. Missing when the failure carried neither |

There is no event ID in the body. EmailEngine sends it in the `X-EE-Wh-Event-Id` request header. See [Delivery and Retries](/docs/webhooks/overview#delivery-and-retries).

## Server Response Codes

Most connection failures carry a Node.js system error code as `serverResponseCode`:

| Code | Description |
|------|-------------|
| `ECONNREFUSED` | Connection refused: nothing is accepting connections on the host and port |
| `ECONNRESET` | Connection reset: the server closed the connection unexpectedly |
| `ETIMEDOUT` | Connection timed out: no response from the server |
| `ENOTFOUND` | DNS lookup failed: the hostname could not be resolved |
| `EHOSTUNREACH` | Host unreachable: no route to the server |
| `ECONNABORTED` | Connection aborted |
| `CERT_HAS_EXPIRED` | The server's TLS certificate has expired |
| `UNABLE_TO_VERIFY_LEAF_SIGNATURE` | The server's TLS certificate could not be verified |
| `SELF_SIGNED_CERT_IN_CHAIN` | A self-signed certificate is in the chain |
| `DEPTH_ZERO_SELF_SIGNED_CERT` | The server presented a self-signed certificate |

When the server answered but refused service before the login, the code is the IMAP response code from that answer instead.

## Example Payload (Connection Refused)

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T14:30:00.000Z",
  "event": "connectError",
  "data": {
    "response": "connect ECONNREFUSED 192.168.1.100:993",
    "serverResponseCode": "ECONNREFUSED"
  }
}
```

## Example Payload (Connection Timeout)

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "office-account",
  "date": "2025-10-17T15:45:00.000Z",
  "event": "connectError",
  "data": {
    "response": "connect ETIMEDOUT 10.0.0.50:993",
    "serverResponseCode": "ETIMEDOUT"
  }
}
```

## Example Payload (DNS Failure)

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "remote-user",
  "date": "2025-10-17T16:20:00.000Z",
  "event": "connectError",
  "data": {
    "response": "getaddrinfo ENOTFOUND mail.invalid-domain.example",
    "serverResponseCode": "ENOTFOUND"
  }
}
```

## Example Payload (TLS Certificate Error)

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "secure-account",
  "date": "2025-10-17T17:00:00.000Z",
  "event": "connectError",
  "data": {
    "response": "certificate has expired",
    "serverResponseCode": "CERT_HAS_EXPIRED"
  }
}
```

## Handling the Event

### Basic Handler

```javascript
async function handleConnectError(event) {
  const { account, data } = event;

  console.error(`Connection failed for account ${account}:`);
  console.error(`  Error: ${data.response}`);
  console.error(`  Code: ${data.serverResponseCode || 'none'}`);

  // Take appropriate action based on error type
  switch (data.serverResponseCode) {
    case 'ECONNREFUSED':
    case 'ETIMEDOUT':
    case 'EHOSTUNREACH':
      await handleNetworkError(account, data);
      break;
    case 'ENOTFOUND':
      await handleDnsError(account, data);
      break;
    case 'CERT_HAS_EXPIRED':
    case 'UNABLE_TO_VERIFY_LEAF_SIGNATURE':
      await handleTlsError(account, data);
      break;
    default:
      await notifyAdmin(account, data);
  }
}

async function handleNetworkError(account, data) {
  // Server may be down or unreachable from EmailEngine's network
  console.log(`Network error for ${account}, server may be unreachable`);

  // Check whether other accounts on the same server are affected
  await checkServerStatus(account);

  await sendInfrastructureAlert({
    type: 'server_unreachable',
    account,
    error: data.response
  });
}

async function handleDnsError(account, data) {
  // The configured hostname does not resolve
  console.log(`DNS error for ${account}, check the hostname configuration`);
  await notifyAdmin(account, {
    message: 'DNS lookup failed, verify the mail server hostname',
    error: data.response
  });
}

async function handleTlsError(account, data) {
  // The server certificate was rejected
  console.log(`TLS error for ${account}, certificate problem`);
  await notifyAdmin(account, {
    message: 'TLS certificate error, the server certificate may need renewal',
    error: data.response
  });
}
```

### Alerting Administrators

```javascript
async function handleConnectError(event) {
  const { account, data, date } = event;

  // Determine severity based on error type
  let severity = 'warning';
  if (['ECONNREFUSED', 'ETIMEDOUT'].includes(data.serverResponseCode)) {
    severity = 'critical';
  }

  // Send alert to monitoring system
  await sendAlert({
    severity,
    title: 'Email Server Connection Failed',
    message: `Account ${account} cannot connect to email server`,
    details: {
      account,
      error: data.response,
      code: data.serverResponseCode,
      timestamp: date
    }
  });
}
```

### Updating Account Status in Database

```javascript
async function handleConnectError(event) {
  const { account, data, date } = event;

  // Update account status in your database
  await db.accounts.update({
    where: { emailEngineId: account },
    data: {
      status: 'connection_error',
      lastError: data.response,
      lastErrorCode: data.serverResponseCode,
      lastErrorAt: new Date(date)
    }
  });

  // Trigger UI notification if user is online
  await notifyConnectedUser(account, {
    type: 'connection_error',
    message: 'Unable to connect to email server'
  });
}
```

## Distinguishing connectError from authenticationError

| Aspect | connectError | authenticationError |
|--------|--------------|---------------------|
| **When triggered** | The connection cannot be established | The connection is established but the login is rejected |
| **Typical causes** | Network issues, server down, firewall, TLS problems | Invalid credentials, expired or revoked OAuth2 grants |
| **Account state** | `connectError` | `authenticationError`, then `unset` once the account is switched off |
| **User action** | Usually none, wait for the server to recover | Update credentials or re-authorize |
| **Resolution** | Automatic once the server is reachable | Requires new credentials |

## Webhook Deduplication

EmailEngine stores the last error for each account and compares each new failure against it:

1. **First occurrence** - Reported at once, and the stored error is set
2. **Same code** - A failure with the same `serverResponseCode` as the stored error is treated as a repeat and not reported, even if `response` differs
3. **Different code** - A different `serverResponseCode` is a new failure and is reported. A server that goes from `ETIMEDOUT` to `ECONNREFUSED` while it is being restarted therefore produces two webhooks
4. **No code** - When neither error carries a code, the whole `data` object is compared, and any difference is reported
5. **Recovery** - A successful login sends [`authenticationSuccess`](/docs/webhooks/authenticationsuccess) and clears the stored error, so the next failure is a first occurrence again

The stored error is shared between `connectError` and `authenticationError`, so a connection failure that follows an unresolved authentication failure is reported as a change, and the reverse likewise.

## Automatic Retry Behavior

After a connection failure the account stays in the `connectError` state and EmailEngine keeps retrying on its own, with exponential backoff that starts at 2 seconds and is capped at 10 minutes between attempts. The retries continue until:

- The connection succeeds
- The account is deleted, paused, or disabled
- The account configuration is updated, which starts a fresh connection

You do not need to request a [reconnect](/docs/accounts/managing-accounts#reconnecting-accounts) from your webhook handler; one only shortens the wait until the next attempt.

## Related Events

- [authenticationError](/docs/webhooks/authenticationerror) - Triggered when the login is rejected after the connection succeeds
- [authenticationSuccess](/docs/webhooks/authenticationsuccess) - Triggered when the account recovers and logs in
- [accountAdded](/docs/webhooks/accountadded) - Triggered when a new account is registered
- [accountDeleted](/docs/webhooks/accountdeleted) - Triggered when an account is removed

## See Also

- [Webhooks Overview](/docs/webhooks/overview) - Configuring the webhook URL and the `webhookEvents` allowlist
- [Account Management](/docs/accounts/managing-accounts) - Account states and reconnecting accounts
- [IMAP Configuration](/docs/accounts/imap-smtp) - Host, port and TLS settings that a connection failure points at
- [Troubleshooting](/docs/troubleshooting) - Diagnosing connectivity between EmailEngine and a mail server
