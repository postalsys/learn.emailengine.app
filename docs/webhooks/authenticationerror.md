---
title: "authenticationError"
sidebar_position: 19
description: "Webhook event triggered when EmailEngine fails to authenticate an email account"
---

# authenticationError

The `authenticationError` webhook event is triggered when EmailEngine fails to authenticate an email account. It is the signal that credentials have stopped working and that someone has to act, because EmailEngine cannot repair a rejected password or a revoked OAuth2 grant on its own.

## When This Event is Triggered

The `authenticationError` event fires when:

- An IMAP server rejects the login
- An OAuth2 access token cannot be renewed from the stored refresh token (IMAP accounts that authenticate with OAuth2)
- The Gmail API or Microsoft Graph rejects the access token, or refuses to issue one
- An [external authentication server](/docs/accounts/authentication-server) fails to return credentials for the account

EmailEngine reports the **first occurrence** of a failure and suppresses repeats. A run of identical failures produces one webhook, followed by a second one only if the account is still failing when it is [switched off](#parked-after-repeated-failures). See [Webhook Deduplication](#webhook-deduplication) for the exact rule.

## Common Use Cases

- **Account monitoring** - Alert administrators when user accounts fail authentication
- **Credential rotation** - Trigger workflows to refresh or request new credentials
- **User notification** - Inform users that their email connection needs attention
- **Dashboard updates** - Update account status in your application's UI
- **Compliance logging** - Track authentication failures for security audits

## Payload Schema

### Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `serviceUrl` | string or null | Yes | The configured EmailEngine service URL. `null` when the `serviceUrl` setting is empty |
| `account` | string | Yes | Account ID that failed authentication |
| `date` | string | Yes | ISO 8601 timestamp when the webhook was generated |
| `event` | string | Yes | Always `authenticationError` |
| `data` | object | Yes | Error details (see below) |

### Error Data Fields (`data` object)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `response` | string | No | The error text: the IMAP server's response line, the provider's error message, or for a token renewal a summary in the form `Token request failed for gmail (refresh_token, HTTP 400): invalid_grant - Token has been expired or revoked.`. Missing only when an IMAP server rejected the login without any text |
| `serverResponseCode` | string | No | Code identifying the failure. Either a response code from the IMAP server or one of EmailEngine's own codes listed below. Missing when the IMAP server sent no response code |
| `tokenRequest` | object | No | Details of the failed token request. Only present with `serverResponseCode: "OauthRenewError"` |

There is no event ID in the body. EmailEngine sends it in the `X-EE-Wh-Event-Id` request header. See [Delivery and Retries](/docs/webhooks/overview#delivery-and-retries).

### Token Request Object (`tokenRequest`)

When an OAuth2 token renewal for an IMAP account fails, the request that failed is described here:

| Field | Type | Description |
|-------|------|-------------|
| `url` | string | Token endpoint the request went to |
| `method` | string | HTTP method, `post` |
| `grant` | string | OAuth2 grant type, `refresh_token` for a renewal |
| `provider` | string | OAuth2 provider of the application, for example `gmail`, `outlook` or `mailRu` |
| `status` | number | HTTP status code from the token endpoint |
| `clientId` | string | OAuth2 client ID of the application, unmasked |
| `scopes` | array | OAuth2 scopes the application is configured with |
| `response` | object | The token endpoint's JSON response body, typically with `error` and `error_description` |
| `authority` | string | Microsoft only: the tenant or authority the request was made against |
| `clientSecret` | string | Only when the provider's error mentions the client secret: the first eight characters of the configured secret followed by its length, so a rotated secret can be recognized without exposing it |
| `flag` | object | Only when the response matched a condition EmailEngine recognizes, such as an API that is not enabled or insufficient scopes: `message`, `code` and, when the provider offers one, a `url` to fix it. The same flag is shown on the OAuth2 application page |

## Server Response Codes

`serverResponseCode` values set by EmailEngine itself:

| Code | Account type | Description |
|------|--------------|-------------|
| `OauthRenewError` | IMAP with OAuth2 | The provider refused to renew the access token from the stored refresh token. `tokenRequest` carries the details |
| `ApiRequestError` | Gmail API, Microsoft Graph | The provider rejected the access token when EmailEngine fetched the account profile |
| `TokenGenerationError` | Gmail API, Microsoft Graph | The provider refused to issue an access token |
| `HTTPRequestError` | Any, with an authentication server | The [authentication server](/docs/accounts/authentication-server) did not return credentials for the account |

For an IMAP login failure the code is whatever the server sent in its response, if it sent one. The values are defined by RFC 5530 and the server, not by EmailEngine. Ones you will see:

| Code | Description |
|------|-------------|
| `AUTHENTICATIONFAILED` | The server rejected the credentials |
| `AUTHORIZATIONFAILED` | The credentials were accepted but the user is not allowed to use them for this mailbox |
| `WEBALERT` | Gmail: a web login is required before IMAP access is allowed |

A server that sends a bare `NO` without a response code produces a payload with `response` but no `serverResponseCode`.

## Example Payload (IMAP Authentication)

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T14:30:00.000Z",
  "event": "authenticationError",
  "data": {
    "response": "Invalid credentials (Failure)",
    "serverResponseCode": "AUTHENTICATIONFAILED"
  }
}
```

## Example Payload (OAuth2 Token Renewal)

An IMAP account whose stored Google refresh token has been revoked:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "gmail-user456",
  "date": "2025-10-17T15:45:00.000Z",
  "event": "authenticationError",
  "data": {
    "response": "Token request failed for gmail (refresh_token, HTTP 400): invalid_grant - Token has been expired or revoked.",
    "serverResponseCode": "OauthRenewError",
    "tokenRequest": {
      "url": "https://oauth2.googleapis.com/token",
      "method": "post",
      "grant": "refresh_token",
      "provider": "gmail",
      "status": 400,
      "clientId": "123456789012-abcdefghijklmnopqrstuvwxyz012345.apps.googleusercontent.com",
      "scopes": [
        "https://mail.google.com/"
      ],
      "response": {
        "error": "invalid_grant",
        "error_description": "Token has been expired or revoked."
      }
    }
  }
}
```

## Example Payload (API Request Error)

For Gmail API or Microsoft Graph API accounts:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "outlook-user789",
  "date": "2025-10-17T16:20:00.000Z",
  "event": "authenticationError",
  "data": {
    "response": "The access token has expired or is not yet valid.",
    "serverResponseCode": "ApiRequestError"
  }
}
```

## Handling the Event

### Basic Handler

```javascript
async function handleAuthenticationError(event) {
  const { account, data } = event;

  console.error(`Authentication failed for account ${account}:`);
  console.error(`  Error: ${data.response || 'no response text'}`);
  console.error(`  Code: ${data.serverResponseCode || 'none'}`);

  switch (data.serverResponseCode) {
    case 'OauthRenewError':
    case 'TokenGenerationError':
    case 'ApiRequestError':
      // The OAuth2 grant is gone. Only the user can grant a new one
      await sendUserNotification(account, 'Please reconnect your email account');
      break;
    case 'AUTHENTICATIONFAILED':
      // Password accounts: the password changed or was revoked
      await sendUserNotification(account, 'Please update your email password');
      break;
    default:
      await notifyAdmin(account, data);
  }
}
```

### Alerting Administrators

```javascript
async function handleAuthenticationError(event) {
  const { account, data, date } = event;

  // Send alert to monitoring system
  await sendAlert({
    severity: 'warning',
    title: 'Email Authentication Failed',
    message: `Account ${account} failed to authenticate`,
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
async function handleAuthenticationError(event) {
  const { account, data, date } = event;

  // Update account status in your database
  await db.accounts.update({
    where: { emailEngineId: account },
    data: {
      status: 'authentication_error',
      lastError: data.response,
      lastErrorCode: data.serverResponseCode,
      lastErrorAt: new Date(date)
    }
  });

  // Trigger UI notification if user is online
  await notifyConnectedUser(account, {
    type: 'account_error',
    message: 'Email connection lost, please reconnect'
  });
}
```

## Error Recovery

### Parked After Repeated Failures

An account that keeps failing authentication is eventually stopped rather than retried forever. Once the first error in the current run is older than [`EENGINE_MAX_IMAP_AUTH_FAILURE_TIME`](/docs/configuration/environment-variables#max-imap-auth-failure-time) (3 days by default), EmailEngine sets `imap.disabled` on the account, records the reason as its last error, and closes the connection. That protects the mail server from a pointless login loop, and the provider's token endpoint from a refresh loop for a grant that has been revoked.

This covers every account type, including Gmail API and Microsoft Graph accounts, which carry no other IMAP settings. Before EmailEngine 2.79.3 it only reached accounts with stored IMAP settings, so a revoked OAuth2 grant was retried indefinitely. Upgrading an instance that has collected such accounts switches every one already past the threshold off on its next failed refresh, so expect a burst of `authenticationError` events shortly after the restart onto 2.79.3 or later, one per account, and a matching drop in token-endpoint traffic.

EmailEngine sends a webhook for the switch-off even though the error itself has not changed, so a second `authenticationError` arriving for an account that has been failing for days is the account going offline. The payload is the same error as before, with nothing in it to mark the difference. The account is what changed. Since 2.79.4, [Get Account](/docs/api/get-v-1-account-account) reports it:

- `authFailureDisabledAt` is the time syncing was switched off, and `null` for every other account. It is read-only. `imap.disabled` is also the operator's own [send-only switch](/docs/accounts/managing-accounts#disabling-and-enabling-accounts), so this field is what tells an automatic disable from a deliberate one
- `state` is `unset`, the same state as an account with no IMAP or OAuth2 configuration at all
- `lastError.description` reads "IMAP was disabled for the account due to exceeding the authentication error threshold"

```javascript
async function handleAuthenticationError(event) {
  const { account, data } = event;

  const accountData = await ee(`/v1/account/${account}`);
  if (accountData.authFailureDisabledAt) {
    // Switched off. Nothing reconnects it until it is re-authorized or resumed
    await escalate(account, data.response, accountData.authFailureDisabledAt);
    return;
  }

  await notifyUser(account, data);
}
```

Accounts that 2.79.3 switched off before `authFailureDisabledAt` existed carry no marker, so 2.79.4 cannot tell them from a deliberate disable and re-authorizing them lifts nothing. 2.79.5 run a one-time backfill at startup that marks the OAuth2 accounts in that situation, after which they recover through the same paths as any other parked account. The recorded `authFailureDisabledAt` is then the time of that startup, not the time the account stopped syncing. Password accounts are not backfilled: their IMAP settings card still has the "Disable IMAP" checkbox, which clears the flag directly.

Delegated accounts, the shared mailboxes that borrow another account's OAuth2 grant, are switched off together with that account in 2.79.3 and 2.79.4, and re-authorizing the owner lifts only the owner. 2.79.5 no longer park a delegated account at all: the failures are the owner's, so the owner is what gets switched off, and re-authorizing it brings the shared mailboxes back with it.

### Re-authenticating an Account

Supplying working credentials lifts the switch-off and reconnects the account. Since 2.79.4 this happens on every path that carries new credentials, without a separate step to clear `imap.disabled`:

- **Re-authorizing an OAuth2 account** through the [hosted authentication form](/docs/accounts/hosted-authentication), or through a [`POST /v1/account`](/docs/api/post-v-1-account) with a fresh `oauth2` block for the existing account ID
- **Saving new IMAP settings** for a password account with [`PUT /v1/account/{account}`](/docs/api/put-v-1-account-account). The `imap` object you send replaces the stored one, so it carries no `disabled` flag unless you add one. In 2.79.5 a partial update that changes `imap.auth` lifts the switch-off as well; in 2.79.4 itself a partial update needs `"disabled": false` next to the new password:

```bash
curl -X PUT "https://emailengine.example.com/v1/account/user123" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "imap": {
      "host": "imap.example.com",
      "port": 993,
      "secure": true,
      "auth": {
        "user": "user@example.com",
        "pass": "new-password"
      }
    }
  }'
```

- **Resume syncing** in the admin interface. The account page of a switched-off account shows a "Syncing was switched off" alert with the time, and a "Resume syncing" button that lifts the flag and reconnects with the stored credentials. Use it when the credentials were fixed on the provider's side, for example a password reset back to the stored value. [List Accounts](/docs/api/get-v-1-accounts) carries `authFailureDisabledAt` on every switched-off account, so they can be found without opening each one.

A plain [reconnect](/docs/accounts/managing-accounts#reconnecting-accounts) is not enough. `PUT /v1/account/{account}/reconnect` answers `{"reconnect": false}` for a switched-off account instead of scheduling anything, and the "Reconnect" button on the account page is disabled, because a reconnect would only repeat the failing login.

An explicit `{"imap": {"partial": true, "disabled": false}}` still works and is the right call only when the credentials are already correct; sent without working credentials it starts a new run of failures that parks the account again once the window passes.

## Webhook Deduplication

EmailEngine stores the last error for each account and compares each new failure against it:

1. **First occurrence** - Reported at once, and the stored error is set
2. **Same code** - A failure with the same `serverResponseCode` as the stored error is treated as a repeat and not reported, even if `response` differs. A server that varies its message from attempt to attempt therefore produces one webhook
3. **Different code** - A different `serverResponseCode` is a new failure and is reported
4. **No code** - When neither error carries a code, the whole `data` object is compared, and any difference is reported
5. **Recovery** - A successful login sends [`authenticationSuccess`](/docs/webhooks/authenticationsuccess) and clears the stored error, so the next failure is a first occurrence again
6. **Switch-off** - The failure that [switches the account off](#parked-after-repeated-failures) is reported even though the error is unchanged

So a run of failures produces one webhook, and a second one only if the account is still failing three days later. That makes it safe to alert on without rate limiting on your end.

## Related Events

- [authenticationSuccess](/docs/webhooks/authenticationsuccess) - Triggered when authentication succeeds
- [connectError](/docs/webhooks/connecterror) - Triggered when the connection fails before authentication
- [accountAdded](/docs/webhooks/accountadded) - Triggered when a new account is registered
- [accountDeleted](/docs/webhooks/accountdeleted) - Triggered when an account is removed

## See Also

- [Webhooks Overview](/docs/webhooks/overview) - Configuring the webhook URL and the `webhookEvents` allowlist
- [Account Management](/docs/accounts/managing-accounts) - Account states, reconnecting, and disabling and enabling accounts
- [Max IMAP Auth Failure Time](/docs/configuration/environment-variables#max-imap-auth-failure-time) - The window after which a failing account is switched off
- [OAuth2 Token Management](/docs/accounts/oauth2-token-management) - Why refresh tokens stop working and how to get new ones
- [Troubleshooting](/docs/troubleshooting) - Diagnosing accounts that fail to authenticate
