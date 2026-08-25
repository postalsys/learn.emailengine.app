---
title: Setting Up Mail.ru with OAuth2
sidebar_position: 6
description: Complete guide to setting up Mail.ru accounts with OAuth2 authentication for IMAP and SMTP access
---

# Setting Up Mail.ru with OAuth2

This guide shows you how to set up Mail.ru OAuth2 authentication for IMAP and SMTP access with EmailEngine. EmailEngine will use these credentials to access Mail.ru accounts via the standard IMAP/SMTP protocols with OAuth2 authentication.

## Overview

When you enable OAuth2 for Mail.ru:

- EmailEngine uses **IMAP** for reading emails and **SMTP** for sending
- Users authenticate once via OAuth2, and EmailEngine automatically refreshes tokens
- No passwords are stored
- Works with Mail.ru accounts that have 2FA enabled

### Required OAuth2 Scopes

Mail.ru OAuth2 requires the following scopes:

- `userinfo` - Access to basic user profile information
- `mail.imap` - Access to IMAP functionality

## Step 1: Register an OAuth2 Application

1. Go to the [Mail.ru OAuth2 Developer Portal](https://oauth.mail.ru/app/)
2. Sign in with your Mail.ru account
3. Click **Create Application** (or similar button)
4. Fill in the application details:
   - **Application Name**: Your application name (e.g., "EmailEngine Integration")
   - **Redirect URI**: Your EmailEngine callback URL (e.g., `https://emailengine.example.com/oauth`)
   - **Scopes**: Select `userinfo` and `mail.imap`

5. After creating the application, note down:
   - **Client ID** (Application ID)
   - **Client Secret**

:::warning Keep Credentials Secure
Never commit your Client ID and Client Secret to version control. Store them securely using environment variables or a secrets manager.
:::

## Step 2: Configure EmailEngine

### Option A: Via Web UI

1. Open the EmailEngine admin dashboard
2. Navigate to **Integrations** > **OAuth2 Apps**
3. Click **Add New Application**
4. Select **Mail.ru** as the provider
5. Enter your credentials:
   - **Client ID**: Your Mail.ru application ID
   - **Client Secret**: Your Mail.ru client secret
   - **Redirect URL**: Must match what you configured in Mail.ru
6. Save the configuration

### Option B: Via API

Register your Mail.ru OAuth2 application credentials:

```bash
curl -XPOST "https://emailengine.example.com/v1/oauth2" \
  -H "Authorization: Bearer <your-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "provider": "mailRu",
    "clientId": "YOUR_MAIL_RU_CLIENT_ID",
    "clientSecret": "YOUR_MAIL_RU_CLIENT_SECRET",
    "redirectUrl": "https://your-domain/oauth"
  }'
```

**Response:**

```json
{
  "app": "AAABkQw...",
  "provider": "mailRu",
  "enabled": true
}
```

## Step 3: Connect Mail.ru Accounts

### Option A: Using Hosted Authentication Form

Generate an authentication link for users:

```bash
curl -XPOST "https://emailengine.example.com/v1/authentication/form" \
  -H "Authorization: Bearer <your-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user-mailru-account",
    "type": "mailRu",
    "name": "User Mail.ru Account",
    "redirectUrl": "https://your-app.com/callback"
  }'
```

The `type` field pre-selects an account type so the user skips the type-selection screen. Set it to your Mail.ru OAuth2 application's ID (the legacy key `mailRu` also resolves to the built-in Mail.ru app). There is no separate `provider` field on this endpoint - omit `type` entirely to let the user choose from all configured account types.

**Response:**

```json
{
  "url": "https://emailengine.example.com/accounts/new?data=..."
}
```

Direct users to the returned URL. They will:
1. Be redirected to Mail.ru's OAuth2 consent screen
2. Grant permission to your application
3. Be redirected back to EmailEngine, which stores the tokens
4. Finally redirect to your specified `redirectUrl`

### Option B: Direct API Registration

If you already have OAuth2 tokens from Mail.ru, register the account directly:

```bash
curl -XPOST "https://emailengine.example.com/v1/account" \
  -H "Authorization: Bearer <your-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "mailru-user",
    "name": "User Name",
    "email": "user@mail.ru",
    "oauth2": {
      "provider": "AAABkQw...",
      "accessToken": "ACCESS_TOKEN_FROM_MAILRU",
      "refreshToken": "REFRESH_TOKEN_FROM_MAILRU",
      "expires": "2024-12-31T23:59:59.000Z",
      "auth": {
        "user": "user@mail.ru"
      }
    }
  }'
```

`provider` is the application ID returned when you registered the app in step 2, and `auth.user` is required.

## Step 4: Verify Connection

Check that the account connected successfully:

```bash
curl "https://emailengine.example.com/v1/account/mailru-user" \
  -H "Authorization: Bearer <your-token>"
```

**Expected response:**

```json
{
  "account": "mailru-user",
  "name": "User Name",
  "email": "user@mail.ru",
  "state": "connected",
  "type": "mailRu",
  "oauth2": {
    "provider": "AAABkQw...",
    "auth": { "user": "user@mail.ru" }
  }
}
```

The `state` should be `"connected"` once EmailEngine establishes the IMAP connection.

## Token Management

EmailEngine automatically handles OAuth2 token refresh for Mail.ru accounts. You can also manually manage tokens if needed.

### Get Current Access Token

Retrieve the current OAuth2 access token for API integrations:

```bash
curl "https://emailengine.example.com/v1/account/mailru-user/oauth-token" \
  -H "Authorization: Bearer <your-token>"
```

**Response:**

```json
{
  "account": "mailru-user",
  "user": "user@mail.ru",
  "provider": "mailRu",
  "app": "AAABkQw...",
  "accessToken": "current-access-token",
  "registeredScopes": ["userinfo", "mail.imap"],
  "expires": "2024-01-15T12:00:00.000Z",
  "cached": true
}
```

### Reconnect the Account

EmailEngine renews the access token on its own when it expires, so there is nothing to trigger by hand. What you can do is rebuild the connection, which discards the cached token along with it:

```bash
curl -XPUT "https://emailengine.example.com/v1/account/mailru-user/reconnect" \
  -H "Authorization: Bearer <your-token>" \
  -H "Content-Type: application/json" \
  -d '{"reconnect": true}'
```

The body is required. `reconnect` defaults to `false`, so a `PUT` with an empty body is accepted and does nothing.

## Troubleshooting

### Authentication Errors

If you see authentication errors:

1. **Verify credentials**: Ensure Client ID and Client Secret are correct
2. **Check redirect URL**: The redirect URL in EmailEngine must exactly match the one registered in Mail.ru
3. **Verify scopes**: Ensure your Mail.ru application has `userinfo` and `mail.imap` scopes enabled
4. **Check token expiry**: If tokens expired and refresh failed, re-authenticate the user

### Connection Issues

If accounts show as disconnected:

1. **Check account state**: Use `GET /v1/account/{account}` to see the current state
2. **View logs**: Check `GET /v1/logs/{account}` for detailed error messages
3. **Reconnect**: Try `PUT /v1/account/{account}/reconnect` to force reconnection

### Common Error Messages

| Error | Cause | Solution |
|-------|-------|----------|
| `Token request failed` | Invalid credentials or expired code | Re-authenticate the user |
| `Empty response from API` | Mail.ru API temporarily unavailable | Retry the operation |
| `Invalid JSON response` | Unexpected response format | Check Mail.ru service status |

## IMAP/SMTP Server Details

For reference, Mail.ru uses these server settings (handled automatically by EmailEngine when using OAuth2):

| Protocol | Server | Port | Security |
|----------|--------|------|----------|
| IMAP | imap.mail.ru | 993 | SSL/TLS |
| SMTP | smtp.mail.ru | 465 | SSL/TLS |

## See Also

- [OAuth2 Setup Guide](/docs/accounts/oauth2-setup) - General OAuth2 concepts
- [OAuth2 Token Management](/docs/accounts/oauth2-token-management) - Managing OAuth2 tokens
- [Account Management](/docs/accounts) - Overview of all account types
- [Hosted Authentication](/docs/accounts/hosted-authentication) - Using hosted auth forms
