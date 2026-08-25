---
title: Using an Authentication Server
sidebar_position: 10
description: Delegate OAuth2 flows to an external authentication server for centralized token management
---

# Using an Authentication Server

The authentication server feature allows you to manage OAuth2 tokens externally while still using EmailEngine for email operations. This is useful when you already have an OAuth2 integration in your application and don't want to ask users for permission twice.

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
    C --> D[When EmailEngine needs authentication]
    D --> E[EmailEngine calls YOUR authentication server]
    E --> F[Your server returns current access token]
    F --> G[EmailEngine uses token for IMAP/SMTP/API]
```

Your application manages tokens, EmailEngine just uses them.

## When to Use Authentication Server

### Good Use Cases

**Existing OAuth2 Integration:**

- You already authenticate users with Google/Microsoft
- Users grant permissions for multiple services at once
- You want to manage tokens centrally
- Avoid asking users for permission multiple times

**Centralized Token Management:**

- Single source of truth for OAuth2 tokens
- Consistent token refresh logic across services
- Easier to audit and monitor token usage
- Simplified token revocation

**Custom Authentication Flows:**

- Non-standard OAuth2 providers
- Custom token acquisition logic
- Special security requirements
- Integration with existing identity systems

### When NOT to Use Authentication Server

**Simple Deployments:**

- EmailEngine is your only OAuth2 integration
- Hosted authentication form is sufficient
- Don't want to maintain a separate authentication service

**Quick Setup:**

- Want to get started fast
- Don't need custom OAuth2 flows
- EmailEngine's built-in OAuth2 is sufficient

:::tip Alternative Approach
If you don't need external OAuth2 management, consider using [Hosted Authentication](/docs/accounts/hosted-authentication) instead. It's simpler and handles everything automatically.
:::

## How It Works

### Authentication Server Protocol

Your authentication server is a simple HTTP endpoint:

**Request from EmailEngine:**

```http
GET /authenticate?account=user123&proto=imap
```

**Query Parameters:**

| Parameter | Description |
|-----------|-------------|
| `account` | Account ID in EmailEngine |
| `proto` | Protocol being authenticated: `imap`, `smtp`, or `api` |

**Response from your server (OAuth2):**

```json
{
  "user": "user@example.com",
  "accessToken": "ya29.a0AWY7Ckl..."
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
| `user` | Yes | Email address or username for authentication |
| `accessToken` | Conditional | OAuth2 access token (required if `pass` not provided) |
| `pass` | Conditional | Password (required if `accessToken` not provided) |

**Key Points:**

- EmailEngine calls your server when it needs to authenticate
- Return either `accessToken` (OAuth2) or `pass` (password) - not both
- For OAuth2: return a **currently valid** (not expired) access token
- For passwords: return the current password for the account
- Your server handles credential management (token refresh, password storage, etc.)
- EmailEngine doesn't store credentials when using auth server - it fetches them on-demand

## Setup Guide

### Step 1: Configure Credentials

The authentication server can return either OAuth2 tokens or regular passwords, depending on your account type.

#### For OAuth2 Accounts (Gmail, Outlook)

Set up your OAuth2 application with Google or Microsoft.

#### For Outlook/Microsoft 365

In your Azure AD application, include the required scopes:

**For IMAP/SMTP:**

```
IMAP.AccessAsUser.All
SMTP.Send
offline_access
```

**For MS Graph API:**

```
Mail.ReadWrite
Mail.Send
offline_access
```

Ensure when redirecting users to Microsoft's sign-in page, you include the appropriate scopes in the `scope` parameter.

#### For Gmail

In your Google Cloud Console application, configure the required scopes:

**For IMAP/SMTP:**

```
https://mail.google.com/
```

**For Gmail API:**

```
https://www.googleapis.com/auth/gmail.modify
```

[See Gmail OAuth2 setup guide for details →](./gmail/gmail-imap)
[See Outlook OAuth2 setup guide for details →](./microsoft-365/outlook-365)

#### For Password-Based Accounts

For regular IMAP/SMTP accounts that use password authentication (not OAuth2), your authentication server returns the username and password. No OAuth2 setup is required.

This is useful for:
- Self-hosted email servers
- Email providers that don't support OAuth2
- Centralized password management across multiple EmailEngine instances
- Dynamic credential rotation

### Step 2: Build Authentication Server

Create an HTTP endpoint that returns credentials for accounts.

#### Example: Password-Based Authentication Server

For IMAP/SMTP accounts using regular password authentication:

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

For OAuth2 accounts (Gmail, Outlook) where you manage tokens externally:

```javascript
const express = require("express");
const app = express();

// Your token storage (e.g., database, Redis)
const tokenStore = {
  user123: {
    email: "user@gmail.com",
    accessToken: "ya29.a0AWY7Ckl...",
    refreshToken: "1//0gDj5...",
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

  // Check if token is expired
  if (new Date(tokens.expiresAt) <= new Date()) {
    // Token expired, refresh it
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
  // Implement token refresh logic for your provider
  // This is provider-specific (Google, Microsoft, etc.)
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

#### Example: Combined Authentication Server

Handle both password and OAuth2 accounts in a single server:

```javascript
const express = require("express");
const app = express();

// Account types: 'password' or 'oauth2'
const accountStore = {
  // Password-based account
  user123: {
    type: "password",
    email: "user@example.com",
    password: "secretpassword",
  },
  // OAuth2 account
  user456: {
    type: "oauth2",
    email: "user@gmail.com",
    accessToken: "ya29.a0AWY7Ckl...",
    refreshToken: "1//0gDj5...",
    expiresAt: "2024-01-15T10:30:00Z",
  },
};

app.get("/authenticate", async (req, res) => {
  const { account, proto } = req.query;

  if (!account) {
    return res.status(400).json({ error: "Missing account parameter" });
  }

  const accountData = accountStore[account];

  if (!accountData) {
    return res.status(404).json({ error: "Account not found" });
  }

  if (accountData.type === "password") {
    // Return password credentials
    return res.json({
      user: accountData.email,
      pass: accountData.password,
    });
  }

  if (accountData.type === "oauth2") {
    // Check if token needs refresh
    if (new Date(accountData.expiresAt) <= new Date()) {
      const newTokens = await refreshAccessToken(accountData.refreshToken);
      accountStore[account] = { ...accountData, ...newTokens };
      return res.json({
        user: accountData.email,
        accessToken: newTokens.accessToken,
      });
    }

    // Return OAuth2 credentials
    return res.json({
      user: accountData.email,
      accessToken: accountData.accessToken,
    });
  }

  res.status(400).json({ error: "Unknown account type" });
});

// ... refreshAccessToken function same as above ...

app.listen(3001, () => {
  console.log("Authentication server running on port 3001");
});
```

:::tip Reference Implementation
See the [test implementation on GitHub](https://github.com/postalsys/emailengine/blob/master/examples/auth-server.js) for a complete example.
:::

#### Response Format

Your authentication server must return one of the following:

**For OAuth2 accounts:**

```json
{
  "user": "user@example.com",
  "accessToken": "current-valid-token"
}
```

**For password-based accounts:**

```json
{
  "user": "user@example.com",
  "pass": "secretpassword"
}
```

**Fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `user` | Yes | Email address or username for authentication |
| `accessToken` | Conditional | OAuth2 access token (required if `pass` not provided) |
| `pass` | Conditional | Password (required if `accessToken` not provided) |

**Important:** For OAuth2, EmailEngine expects the access token to be valid immediately. If it's expired, authentication will fail. Your server must handle token refresh before returning.

The response is validated before it is used. `user` is required and capped at 256 characters, `pass` at 256, and `accessToken` at 16384. Any other field is discarded, so returning an expiry or a refresh token alongside them is harmless but has no effect: EmailEngine asks again when it next needs a credential rather than tracking the lifetime itself.

### Step 3: Configure EmailEngine

Set the authentication server URL in EmailEngine settings using the [Update Settings API endpoint](/docs/api/post-v-1-settings):

```bash
curl -X POST https://emailengine.example.com/v1/settings \
  -H "Authorization: Bearer YOUR_EMAILENGINE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "authServer": "https://myservice.com/authenticate"
  }'
```

This tells EmailEngine where to fetch credentials (passwords or access tokens).

### Step 4: Register Accounts

#### For Password-Based IMAP/SMTP

Register accounts where your authentication server returns passwords:

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

#### For OAuth2 IMAP/SMTP (Outlook Example)

Register accounts where your authentication server returns OAuth2 tokens:

```bash
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_EMAILENGINE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user123",
    "name": "John Doe",
    "email": "john@outlook.com",
    "imap": {
      "useAuthServer": true,
      "host": "outlook.office365.com",
      "port": 993,
      "secure": true
    },
    "smtp": {
      "useAuthServer": true,
      "host": "smtp-mail.outlook.com",
      "port": 587,
      "secure": false
    }
  }'
```

**Key Points:**

- Set `useAuthServer: true` in both `imap` and `smtp` sections
- No credentials provided (EmailEngine fetches them from your server)
- Specify IMAP/SMTP host and port normally
- Works with both password and OAuth2 - EmailEngine uses whatever your server returns

#### For Gmail API or MS Graph API

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
      "provider": "<app-id>",
      "auth": {
        "user": "john@gmail.com"
      }
    }
  }'
```

**Key Points:**

- Set `useAuthServer: true` in `oauth2` section
- Specify `provider` as your OAuth2 app ID in EmailEngine
- `auth.user` should match the email address

:::info OAuth2 App Still Required
Even when using an authentication server with Gmail API or MS Graph API, you must still create the OAuth2 application in EmailEngine. EmailEngine uses the application information for reference (scopes, endpoints, etc.) but doesn't use it to manage tokens - it fetches them from your authentication server instead.
:::

## Authentication Flow

### When EmailEngine Needs to Authenticate

EmailEngine calls your authentication server when:

1. **Initial connection** - When account is first registered
2. **Reconnection** - When connection is lost and needs to be re-established
3. **Token expiration** - When current token expires (IMAP/SMTP sessions)
4. **API operations** - Before each Gmail API or MS Graph API call

### Request from EmailEngine

```http
GET https://myservice.com/authenticate?account=user123&proto=imap
Host: myservice.com
User-Agent: emailengine/2.79.3 (+https://emailengine.app)
```

A GET request carrying the account ID and the protocol it is authenticating. `proto` is `imap`, `smtp`, or `api`.

If the configured `authServer` URL contains a username and password, EmailEngine strips them from the URL and sends them as an HTTP Basic `Authorization` header instead, so `https://ee:s3cret@auth.example.com/authenticate` authenticates the caller without putting the credentials in your access logs.

### Your Server's Response

**Success (200 OK):**

```json
{
  "user": "john@outlook.com",
  "accessToken": "EwBIA8l6..."
}
```

**Account Not Found (404 Not Found):**

```json
{
  "error": "Account not found"
}
```

**Server Error (500 Internal Server Error):**

```json
{
  "error": "Failed to retrieve token"
}
```

### EmailEngine's Behavior

**On Success:**

- Uses the provided access token to authenticate IMAP/SMTP or API connection
- Proceeds with email operations

**On Failure:**

- Account enters error state
- Retries periodically
- Logs error for debugging

An account that carries `useAuthServer` but also has stored credentials is a special case. If no `authServer` is configured, EmailEngine logs a warning and uses the stored credentials rather than failing the account, which keeps long-lived instances working where the flag was set years ago and never had an effect.

## Advanced Patterns

### Multiple EmailEngine Instances

If you have multiple EmailEngine instances:

```bash
# Each instance uses the same authentication server
curl -X POST https://ee1.company.com/v1/settings \
  -H "Authorization: Bearer TOKEN1" \
  -H "Content-Type: application/json" \
  -d '{ "authServer": "https://auth.company.com/authenticate" }'

curl -X POST https://ee2.company.com/v1/settings \
  -H "Authorization: Bearer TOKEN2" \
  -H "Content-Type: application/json" \
  -d '{ "authServer": "https://auth.company.com/authenticate" }'
```

All instances share the same centralized token management.

### Per-Account Authentication URLs

For different OAuth2 providers or account types:

Store authentication URLs per account and use a router:

```javascript
const authEndpoints = {
  user123: "https://auth.company.com/google",
  user456: "https://auth.company.com/microsoft",
};

app.get("/authenticate", (req, res) => {
  const { account } = req.query;
  const endpoint = authEndpoints[account];

  if (!endpoint) {
    return res.status(404).json({ error: "Account not found" });
  }

  // Proxy to appropriate endpoint
  fetch(`${endpoint}?account=${account}`)
    .then((r) => r.json())
    .then((data) => res.json(data))
    .catch((err) => res.status(500).json({ error: err.message }));
});
```

### Health Checks

Implement health check endpoint:

```javascript
app.get("/health", (req, res) => {
  res.json({
    status: "ok",
    timestamp: new Date().toISOString(),
  });
});
```

Monitor this endpoint to ensure authentication server is running.

## See Also

- [Hosted authentication](/docs/accounts/hosted-authentication) - Letting EmailEngine own the OAuth2 flow instead
- [OAuth2 setup](/docs/accounts/oauth2-setup) - Registering the provider application this still needs
- [Managing accounts](/docs/accounts/managing-accounts) - Registering and updating the accounts that use the server
- [Security](/docs/deployment/security) - Protecting the endpoint that hands out credentials
