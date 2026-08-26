---
title: Using an Authentication Server
sidebar_position: 10
description: Delegate OAuth2 flows to an external authentication server for centralized token management
---

# Using an Authentication Server

The authentication server feature lets you keep mailbox credentials outside EmailEngine. Instead of storing a password or an OAuth2 refresh token with the account, EmailEngine asks an HTTP endpoint of yours for a credential every time it needs one. This is useful when you already have an OAuth2 integration in your application and do not want to ask users for permission twice, or when passwords live in a vault and must not be copied into Redis.

## Overview

### Standard OAuth2 Flow

Normally with EmailEngine:

```mermaid
flowchart TD
    A[User clicks 'Connect Email'] --> B[Redirected to EmailEngine's hosted auth form]
    B --> C[EmailEngine redirects to Google/Microsoft OAuth]
    C --> D[User grants permissions]
    D --> E[EmailEngine stores tokens]
    E --> F[EmailEngine uses tokens for email access]
```

EmailEngine manages the entire OAuth2 lifecycle.

### Authentication Server Flow

With an authentication server:

```mermaid
flowchart TD
    A[User authenticates via YOUR OAuth2 flow] --> B[Your app stores tokens]
    B --> C[You register account in EmailEngine with useAuthServer: true]
    C --> D[When EmailEngine needs a credential]
    D --> E[EmailEngine calls YOUR authentication server]
    E --> F[Your server returns the current access token or password]
    F --> G[EmailEngine uses it for IMAP/SMTP/API]
```

Your application manages the credentials; EmailEngine only uses them.

## When to Use an Authentication Server

### Good Use Cases

**Existing OAuth2 Integration:**

- You already authenticate users with Google/Microsoft
- Users grant permissions for multiple services at once
- You want to manage tokens centrally
- Avoid asking users for permission multiple times

**Centralized Credential Management:**

- Single source of truth for OAuth2 tokens or passwords
- Consistent token refresh logic across services
- Easier to audit and monitor token usage
- Credentials that rotate, or that must not be stored in EmailEngine

**Custom Authentication Flows:**

- Non-standard OAuth2 providers
- Custom token acquisition logic
- Integration with existing identity systems

### When NOT to Use an Authentication Server

- EmailEngine is your only OAuth2 integration
- The hosted authentication form is sufficient
- You do not want to maintain a separate service that every connection depends on

:::tip Alternative Approach
If you do not need external credential management, use [Hosted Authentication](/docs/accounts/hosted-authentication) instead. EmailEngine then stores and refreshes the tokens itself.
:::

## How It Works

### Authentication Server Protocol

Your authentication server is a single HTTP endpoint that EmailEngine calls with a GET request.

**Request from EmailEngine:**

```http
GET /authenticate?account=user123&proto=imap HTTP/1.1
Host: auth.example.com
User-Agent: emailengine-app/2.79.4 (+https://emailengine.app/)
```

EmailEngine appends two query parameters to the configured URL, keeping any query string the URL already has:

| Parameter | Description |
|-----------|-------------|
| `account` | Account ID in EmailEngine |
| `proto` | What the credential is for: `imap` (mailbox sync), `smtp` (sending), or `api` (Gmail API and MS Graph accounts) |

If the configured URL carries a username and password (`https://ee:s3cret@auth.example.com/authenticate`), EmailEngine strips them from the URL and sends them as an HTTP Basic `Authorization` header instead, so the caller is authenticated without the secret landing in your access logs.

The request times out after 90 seconds (`EENGINE_FETCH_TIMEOUT`, in milliseconds). A `429` response or a connection-level failure (DNS, connection refused, reset, timeout) is retried up to 5 times; any other non-2xx status is a failure and is not retried.

**Response from your server (OAuth2):**

```json
{
  "user": "user@example.com",
  "accessToken": "ya29.a0AWY7CklEXAMPLEACCESSTOKEN"
}
```

**Response from your server (Password):**

```json
{
  "user": "user@example.com",
  "pass": "secretpassword"
}
```

**Response Fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `user` | Yes | Email address or username for authentication. At most 256 characters |
| `accessToken` | Conditional | OAuth2 access token, at most 16384 characters. Required if `pass` is not provided |
| `pass` | Conditional | Password, at most 256 characters (an empty string is accepted). Required if `accessToken` is not provided; when `accessToken` is present, `pass` must be omitted or `null` |

Any other field is discarded, so returning an expiry or a refresh token alongside them is harmless but has no effect: EmailEngine asks again when it next needs a credential rather than tracking the lifetime itself. A response that is not valid JSON, or that fails this validation, is a failure.

**Key Points:**

- Return either `accessToken` (OAuth2) or `pass` (password), not both
- For OAuth2, return an access token that is valid right now. EmailEngine uses it immediately; if it has expired, authentication fails. Refresh it on your side before responding
- When `accessToken` is returned for an IMAP or SMTP connection, EmailEngine logs in with `XOAUTH2`
- EmailEngine stores nothing it receives. Every connection asks again

### When EmailEngine Calls Your Server

- **IMAP** (`proto=imap`): every time a connection to the mailbox is set up, which is the initial connection and every reconnect
- **SMTP** (`proto=smtp`): every message submission
- **API** (`proto=api`): whenever a Gmail API or MS Graph account needs an access token for a request. Nothing the server returns is cached

### What Happens on Failure

A failed call (unreachable server, non-2xx status, invalid response) is handled per protocol:

- **IMAP**: the account reports an [`authenticationError`](/docs/webhooks/authenticationerror) webhook with `serverResponseCode: "HTTPRequestError"` and the error message in `response`, enters the `authenticationError` state, and keeps retrying on the normal reconnect schedule. For an account registered with an `oauth2` block, a connection-level failure to reach the server (DNS, refused, reset, timeout) is instead logged as a warning and treated as a connection problem, so a brief outage of your server does not report an authentication error for every account on the instance
- **SMTP**: the submission fails with the error and is logged
- **API**: the request that needed the token fails

For an account registered with an `oauth2` block, the SMTP and API paths report the failure the same way the IMAP path does: an `authenticationError` webhook with `serverResponseCode: "HTTPRequestError"`, and the account enters the `authenticationError` state.

Because every account depends on the same endpoint, an outage of the authentication server affects all of them at once. An account that keeps failing authentication for longer than [`EENGINE_MAX_IMAP_AUTH_FAILURE_TIME`](/docs/configuration/environment-variables#max-imap-auth-failure-time) (three days by default) is switched off: it reports the `unset` state with a non-null `authFailureDisabledAt` (since v2.79.4). An authentication-server account has no credentials to re-supply, so once the server is fixed bring the account back with **Resume syncing** on its page in the admin interface, or by registering it again with `POST /v1/account` under the same account ID. See [Accounts switched off after authentication failures](/docs/accounts/managing-accounts#accounts-switched-off-after-authentication-failures).

## Setup Guide

### Step 1: Decide What Your Server Returns

The authentication server can return OAuth2 access tokens or passwords, per account.

#### For OAuth2 Accounts (Gmail, Microsoft 365)

Your own OAuth2 application with Google or Microsoft requests the scopes EmailEngine needs for the protocol it will use. These are the scopes EmailEngine's own applications request:

**Microsoft 365, IMAP and SMTP:**

```
IMAP.AccessAsUser.All
SMTP.Send
offline_access
```

**Microsoft 365, MS Graph API:**

```
Mail.ReadWrite
Mail.Send
offline_access
```

**Gmail, IMAP and SMTP:**

```
https://mail.google.com/
```

**Gmail, Gmail API:**

```
https://www.googleapis.com/auth/gmail.modify
```

EmailEngine's own applications also request the identity scopes (`openid`, `email` and `profile` for Google; `openid`, `profile` and `User.Read` for Microsoft) to learn the signed-in address. They are not needed for the mailbox connection itself. Include the scopes in the `scope` parameter when you send users to the provider's sign-in page. See the [Gmail](./gmail/gmail-imap) and [Microsoft 365](./microsoft-365/outlook-365) setup guides for the provider-side configuration, and the [Gmail API scopes reference](./gmail/gmail-api-scopes) for narrower Gmail scope sets.

#### For Password-Based Accounts

For IMAP/SMTP accounts that use password authentication, your authentication server returns the username and password. No OAuth2 setup is required.

This is useful for:
- Self-hosted email servers
- Email providers that do not support OAuth2
- Centralized password management across multiple EmailEngine instances
- Credential rotation without updating every account in EmailEngine

### Step 2: Build the Authentication Server

Create an HTTP endpoint that returns credentials for accounts.

#### Example: Password-Based Authentication Server

For IMAP/SMTP accounts using password authentication:

```javascript
const express = require("express");
const app = express();

// Your credential storage (e.g., database, vault, secrets manager)
const credentialStore = {
  user123: {
    email: "user@example.com",
    password: "secretpassword",
  },
  user456: {
    email: "another@company.com",
    password: "anotherpassword",
  },
};

app.get("/authenticate", async (req, res) => {
  const { account, proto } = req.query;

  if (!account) {
    return res.status(400).json({ error: "Missing account parameter" });
  }

  // Fetch credentials for this account
  const credentials = credentialStore[account];

  if (!credentials) {
    return res.status(404).json({ error: "Account not found" });
  }

  // Return username and password
  res.json({
    user: credentials.email,
    pass: credentials.password,
  });
});

app.listen(3001, () => {
  console.log("Authentication server running on port 3001");
});
```

#### Example: OAuth2 Authentication Server

For OAuth2 accounts where you manage tokens externally. The server refreshes an expired token before answering, because EmailEngine uses whatever it receives right away:

```javascript
const express = require("express");
const app = express();

// Your token storage (e.g., database, Redis)
const tokenStore = {
  user123: {
    email: "user@gmail.com",
    accessToken: "ya29.a0AWY7CklEXAMPLEACCESSTOKEN",
    refreshToken: "1//0gDj5EXAMPLEREFRESHTOKEN",
    expiresAt: "2024-01-15T10:30:00Z",
  },
};

app.get("/authenticate", async (req, res) => {
  const { account, proto } = req.query;

  if (!account) {
    return res.status(400).json({ error: "Missing account parameter" });
  }

  // Fetch tokens for this account
  const tokens = tokenStore[account];

  if (!tokens) {
    return res.status(404).json({ error: "Account not found" });
  }

  // Refresh the token if it has expired
  if (new Date(tokens.expiresAt) <= new Date()) {
    const newTokens = await refreshAccessToken(tokens.refreshToken);

    // Update storage
    tokenStore[account] = {
      ...tokens,
      ...newTokens,
    };

    return res.json({
      user: tokens.email,
      accessToken: newTokens.accessToken,
    });
  }

  // Return current token
  res.json({
    user: tokens.email,
    accessToken: tokens.accessToken,
  });
});

async function refreshAccessToken(refreshToken) {
  // Token refresh is provider-specific; this is Google's endpoint
  const response = await fetch("https://oauth2.googleapis.com/token", {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      client_id: process.env.GOOGLE_CLIENT_ID,
      client_secret: process.env.GOOGLE_CLIENT_SECRET,
      refresh_token: refreshToken,
      grant_type: "refresh_token",
    }),
  });

  const data = await response.json();

  return {
    accessToken: data.access_token,
    expiresAt: new Date(Date.now() + data.expires_in * 1000).toISOString(),
  };
}

app.listen(3001, () => {
  console.log("Authentication server running on port 3001");
});
```

A single server can serve both kinds of account: look the account up, and answer with `pass` or `accessToken` depending on what is stored for it. The `proto` parameter lets you hand out different credentials for IMAP and SMTP if the mailbox needs that.

:::tip Reference Implementation
The EmailEngine repository ships a complete example, [examples/auth-server.js](https://github.com/postalsys/emailengine/blob/master/examples/auth-server.js), that serves password accounts, OAuth2 accounts with cached token refresh, and per-protocol credentials.
:::

### Step 3: Configure EmailEngine

Set the authentication server URL with the [settings API](/docs/api/post-v-1-settings). The `authServer` setting has no field in the admin interface; it is set through the API or with `EENGINE_SETTINGS`:

```bash
curl -X POST https://emailengine.example.com/v1/settings \
  -H "Authorization: Bearer YOUR_EMAILENGINE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "authServer": "https://auth.example.com/authenticate"
  }'
```

The value must be an absolute `http://` or `https://` URL. Setting it to an empty string removes it. It applies to the whole instance; there is no per-account URL.

### Step 4: Register Accounts

An account uses the authentication server when its connection configuration carries `useAuthServer: true`. The `auth` block must then be left out; the API rejects an `auth` object next to the flag.

#### IMAP/SMTP Accounts (Password or OAuth2 Token)

Register the account with its host and port, and let the server supply the credential. This works for password accounts and for OAuth2 providers alike: when your server returns `accessToken`, EmailEngine logs in with `XOAUTH2`.

```bash
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_EMAILENGINE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user123",
    "name": "John Doe",
    "email": "john@example.com",
    "imap": {
      "useAuthServer": true,
      "host": "imap.example.com",
      "port": 993,
      "secure": true
    },
    "smtp": {
      "useAuthServer": true,
      "host": "smtp.example.com",
      "port": 587,
      "secure": false
    }
  }'
```

For Microsoft 365 over IMAP/SMTP the hosts are `outlook.office365.com:993` and `smtp-mail.outlook.com:587`; for Gmail, `imap.gmail.com:993` and `smtp.gmail.com:587`.

**Key Points:**

- Set `useAuthServer: true` in both `imap` and `smtp` sections
- Omit `auth` entirely (EmailEngine fetches the credential from your server)
- Specify host and port as for any other account
- EmailEngine calls the server with `proto=imap` and `proto=smtp`

#### OAuth2 Accounts Registered Against an OAuth2 Application

An account can also carry the flag in its `oauth2` block, together with the ID of an [OAuth2 application](/docs/accounts/oauth2-setup) in EmailEngine. EmailEngine takes the provider's host and port, and whether the account syncs over IMAP or an API, from the application:

```bash
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_EMAILENGINE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user123",
    "name": "John Doe",
    "email": "john@gmail.com",
    "oauth2": {
      "useAuthServer": true,
      "provider": "AAABhaBPHscAAAAH",
      "auth": {
        "user": "john@gmail.com"
      }
    }
  }'
```

**Key Points:**

- `provider` is the OAuth2 application ID in EmailEngine (shown under **Integrations** > **OAuth2 Apps**), not the provider's client ID
- `oauth2.auth` is required here: `auth.user` names the mailbox and must be the address the server returns as `user`. A Microsoft 365 shared mailbox is still addressed by `auth.delegatedUser`, which takes precedence over the username the server returns
- With an application that uses the **IMAP and SMTP** base scope, EmailEngine calls the server with `proto=imap` and `proto=smtp`; with a **Gmail API** or **MS Graph API** application, with `proto=api`
- Do not send `accessToken` or `refreshToken`; the server owns them

:::info OAuth2 Application Still Required
The application is still resolved on every connection, because the provider's host, port and base scope come from it. EmailEngine does not use the application's client ID and secret to refresh tokens for such an account; it fetches the token from your server instead.
:::

:::note Version notes
- Since v2.77.0, `oauth2.useAuthServer` is honored on the IMAP sync and SMTP send paths as well as for API accounts. Earlier releases read the flag only for Gmail API and MS Graph accounts, and an `oauth2` account with the flag and no stored tokens failed with a token renewal error.
- Since v2.78.0, an account that carries `useAuthServer` but also has stored credentials falls back to those credentials, with a warning in the log, when no `authServer` is configured. This keeps instances working where the flag was set long ago without ever taking effect. Once `authServer` is set, the flag is honored and the stored credentials are ignored.
:::

## Advanced Patterns

### Multiple EmailEngine Instances

If you run several EmailEngine instances, point each at the same authentication server:

```bash
curl -X POST https://ee1.example.com/v1/settings \
  -H "Authorization: Bearer TOKEN1" \
  -H "Content-Type: application/json" \
  -d '{ "authServer": "https://auth.example.com/authenticate" }'

curl -X POST https://ee2.example.com/v1/settings \
  -H "Authorization: Bearer TOKEN2" \
  -H "Content-Type: application/json" \
  -d '{ "authServer": "https://auth.example.com/authenticate" }'
```

All instances then share one credential store. Account IDs are what the server sees, so keep them unique across instances or include an instance marker in the URL's query string, which EmailEngine preserves.

### Routing by Account

The `authServer` setting is instance-wide, so a deployment with several identity providers routes inside the server:

```javascript
const authEndpoints = {
  user123: "https://auth.example.com/google",
  user456: "https://auth.example.com/microsoft",
};

app.get("/authenticate", (req, res) => {
  const { account, proto } = req.query;
  const endpoint = authEndpoints[account];

  if (!endpoint) {
    return res.status(404).json({ error: "Account not found" });
  }

  // Proxy to the appropriate endpoint
  fetch(`${endpoint}?account=${encodeURIComponent(account)}&proto=${encodeURIComponent(proto)}`)
    .then((r) => r.json())
    .then((data) => res.json(data))
    .catch((err) => res.status(500).json({ error: err.message }));
});
```

### Availability

Every connection, submission and API request depends on the server answering within the timeout, so monitor it like any other production dependency. EmailEngine retries only `429` responses and connection-level failures; a `500` is reported as a credential failure straight away.

## See Also

- [Hosted authentication](/docs/accounts/hosted-authentication) - Letting EmailEngine own the OAuth2 flow instead
- [OAuth2 setup](/docs/accounts/oauth2-setup) - Registering the provider application an `oauth2` account still needs
- [Managing accounts](/docs/accounts/managing-accounts) - Registering and updating the accounts that use the server
- [authenticationError webhook](/docs/webhooks/authenticationerror) - What EmailEngine reports when the server fails
- [Security](/docs/deployment/security) - Protecting the endpoint that hands out credentials
