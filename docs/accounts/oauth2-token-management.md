---
title: OAuth2 Token Management
sidebar_position: 8
description: Using EmailEngine to manage OAuth2 tokens and access provider APIs
---

# OAuth2 Token Management

EmailEngine automatically manages OAuth2 tokens for registered accounts, refreshing access tokens when they expire. You can also retrieve these tokens to use with other Google or Microsoft APIs directly.

:::tip Quick Reference
**API Endpoint:** [GET /v1/account/\{account\}/oauth-token](/docs/api/get-v-1-account-account-oauthtoken) - Retrieve a valid OAuth2 access token for any account
:::

## Overview

When you register OAuth2 accounts in EmailEngine:

- EmailEngine stores OAuth2 tokens in Redis
- An access token that has expired, or expires within the next 30 seconds, is refreshed before it is used: when a connection is opened, when a Gmail API or MS Graph request is made, and when you request the token through the API
- Because access tokens are short-lived, a connected account exercises its refresh token regularly, which keeps the provider's inactivity timers from running out
- You never need to handle token refresh logic
- Tokens can be retrieved for use with other APIs via the [OAuth2 Token API](/docs/api/get-v-1-account-account-oauthtoken)

:::warning Token Encryption
OAuth2 tokens (including sensitive refresh tokens and client secrets) are stored **encrypted in Redis** only if you configure the `EENGINE_SECRET` environment variable. Without encryption enabled, credentials are stored in **cleartext**. For production deployments, always enable encryption by setting a strong encryption secret. [See encryption documentation](/docs/advanced/encryption)
:::

EmailEngine can therefore serve as the OAuth2 token store for the rest of your application, not only for email access.

## Use Cases

### Email-Only Access

Most applications only need EmailEngine for email operations:

- Reading emails via webhooks
- Sending emails via REST API
- Managing folders and messages

EmailEngine handles all OAuth2 complexity transparently.

### Multi-Service Access

Some applications need to access multiple Google/Microsoft services:

**Example with Gmail:**

- EmailEngine accesses Gmail for email operations
- Your app accesses Google Calendar API
- Your app accesses Google Drive API
- All using the same OAuth2 tokens

**Example with Outlook:**

- EmailEngine accesses Outlook for email operations
- Your app accesses Microsoft Calendar API
- Your app accesses OneDrive API
- All using the same OAuth2 tokens

Instead of implementing separate OAuth2 flows, you can use EmailEngine as your central OAuth2 manager:

```mermaid
flowchart LR
    A[User] -->|authenticates once| B[EmailEngine]
    B -->|email operations| C[Gmail/Outlook]
    B -->|provides tokens| D[Your App]
    D -->|calendar API| E[Google Calendar / MS Calendar]
    D -->|storage API| F[Google Drive / OneDrive]
```

1. Use EmailEngine for OAuth2 authentication
2. Request additional scopes during setup
3. Retrieve tokens from EmailEngine using the [OAuth2 Token API](/docs/api/get-v-1-account-account-oauthtoken)
4. Use tokens to call provider APIs directly

## Setting Up Multi-Service Access

### Step 1: Configure OAuth2 Scopes

When creating your OAuth2 application (Google Cloud Console or Azure AD), request all scopes you need.

#### Google Example

In Google Cloud Console, add all required scopes:

| Scope | Purpose |
|-------|---------|
| `https://mail.google.com/` | Email access |
| `https://www.googleapis.com/auth/calendar` | Calendar access |
| `https://www.googleapis.com/auth/postmaster.readonly` | Postmaster tools |
| `https://www.googleapis.com/auth/drive.readonly` | Google Drive (read-only) |

#### Microsoft Example

In Azure AD, add all required permissions:

| Permission | Purpose |
|------------|---------|
| `Mail.ReadWrite` | Email access |
| `Mail.Send` | Sending emails |
| `Calendars.ReadWrite` | Calendar access |
| `Files.Read` | OneDrive access |
| `offline_access` | Token refresh capability |

:::warning Microsoft Scope Compatibility
Additional OAuth2 scopes with Microsoft accounts are only supported when using the **MS Graph API backend** (Mail.\* scopes). If you configure EmailEngine to use IMAP/SMTP (IMAP.AccessAsUser.All, SMTP.Send scopes), those access tokens are valid **only for IMAP/SMTP** and cannot be used with other Microsoft Graph APIs like Calendars or Files.
:::

### Step 2: Configure Additional Scopes in EmailEngine

When setting up the OAuth2 application in EmailEngine, add extra scopes to the **Additional scopes** field.

**Google Example:**

Navigate to **Integrations** > **OAuth2 Apps**, open your Gmail app and click **Edit app**.

**Additional scopes** field:

```
https://www.googleapis.com/auth/calendar
https://www.googleapis.com/auth/postmaster.readonly
```

**Microsoft Example:**

The same field, with scopes from `https://graph.microsoft.com/`:

```
https://graph.microsoft.com/Calendars.ReadWrite
https://graph.microsoft.com/Files.Read
```

Adding a delegated permission in Azure AD is not enough on its own: EmailEngine only requests the scopes it is configured with, so a permission the user is never asked to consent to does not end up in the token. The app must use the **MS Graph API** base scope; a Microsoft token is issued for one resource, so `graph.microsoft.com` scopes cannot be combined with the IMAP/SMTP base scope (see [Additional scopes](/docs/accounts/oauth2-setup#additional-scopes)).

### Step 3: Enable OAuth2 Token API Endpoint

For security, the OAuth2 token API endpoint is **disabled by default**.

**To enable via Web UI:**

1. Navigate to **Configuration** > **Security**
2. Check **Allow OAuth2 Token Access via API**
3. Click **Save**

**To enable via environment variable:**

Set `EENGINE_ENABLE_OAUTH_TOKENS_API=true` when starting EmailEngine. The variable seeds the setting on an instance where it has never been saved; once the checkbox has been saved either way, the stored value wins.

:::warning Security Consideration
The OAuth2 token endpoint returns access tokens that can access user data. Only enable this if you need it, and ensure your EmailEngine API is properly secured with strong access tokens and appropriate access controls.

This setting **cannot be changed via the API** - it must be configured through the web interface or environment variable.
:::

### Step 4: Enable Required APIs

Make sure the APIs you want to use are enabled in the provider console.

**Google Cloud Console:**

Navigate to **APIs & Services** > **Enabled APIs and services**.

Search for and enable required APIs (e.g., "Google Calendar API", "Gmail Postmaster API").

**Azure AD:**

Permissions added in Azure AD are automatically linked to their respective APIs.

### Step 5: Add Accounts

Add accounts via hosted authentication or API. Users will be asked to grant all configured permissions during consent.

:::important Add Accounts After Scope Configuration
If you add accounts before configuring all scopes, those accounts will be missing the required permissions. You'll need to have users re-authenticate to grant the new permissions.
:::

## Retrieving OAuth2 Tokens

### Get Current Access Token

Use the [OAuth2 Token API endpoint](/docs/api/get-v-1-account-account-oauthtoken) to retrieve a currently valid access token:

```bash
curl https://emailengine.example.com/v1/account/example/oauth-token \
  -H "Authorization: Bearer YOUR_EMAILENGINE_TOKEN"
```

**Response:**

```json
{
  "account": "example",
  "user": "user@example.com",
  "accessToken": "ya29.a0AVA9y1sXQ....CP1A",
  "provider": "gmail",
  "app": "AAABhaBPHscAAAAI",
  "registeredScopes": ["https://www.googleapis.com/auth/postmaster.readonly", "https://mail.google.com/"],
  "expires": "2022-07-08T14:25:27.780Z",
  "cached": true
}
```

**Response Fields:**

- `account` - Account ID in EmailEngine
- `user` - Email address of the account
- `accessToken` - Currently valid OAuth2 access token. Absent when an [authentication server](/docs/accounts/authentication-server) supplied password credentials instead of a token
- `provider` - OAuth2 provider type (`gmail`, `gmailService`, `outlook`, `outlookService`, `mailRu`)
- `app` - ID of the OAuth2 application the account was registered through. Omitted for the legacy applications whose ID equals the provider name
- `registeredScopes` - List of scopes this token has access to
- `expires` - When the access token expires (ISO 8601)
- `cached` - `false` when this request refreshed the token, `true` when the stored token was still valid

:::tip Token Validity
A stored token that has expired, or expires within the next 30 seconds, is refreshed before the response is sent. If the refresh fails, the endpoint answers 403 with the error code `OauthRenewError` and the provider's rejection under `tokenRequest`.
:::

### Token Lifetime

#### Google (Gmail API)

**Access tokens:**

- Expire after 1 hour
- EmailEngine refreshes them automatically using the refresh token

**Refresh tokens:**

- Long-lived but can be invalidated under certain conditions
- EmailEngine keeps them active by regular use

**Conditions that invalidate Google refresh tokens:**

| Condition                             | Explanation                                                                                             |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| **User revokes your app**             | Immediate invalidation when user removes permissions via Google account settings                        |
| **Not used for 6 months**             | Auto-expired due to inactivity                                                                          |
| **Gmail password changed**            | Token invalidated when Gmail scopes are present                                                         |
| **Too many refresh tokens issued**    | Google keeps at most 100 refresh tokens per account per client ID; the oldest is invalidated when a new one is issued past that limit |
| **Consent was time-bounded**          | If user granted time-based access, it expires accordingly                                               |
| **OAuth consent screen in "Testing"** | For external apps in Testing mode, refresh tokens expire after 7 days. Move to Production to avoid this |

:::warning Testing Mode Expiration
If your Google OAuth app is in "Testing" status, refresh tokens expire after **7 days**. Publish your app to "In production" status to get long-lived refresh tokens.
:::

#### Microsoft (Graph API / Outlook)

**Access tokens:**

- Expire after 1 hour
- EmailEngine refreshes them automatically using the refresh token

**Refresh tokens:**

- Each refresh returns a new refresh token, which EmailEngine stores in place of the old one, so a token in regular use has no fixed lifetime
- EmailEngine keeps them active by regular use

**Conditions that invalidate Microsoft refresh tokens:**

| Condition                                   | Explanation                                                                                |
| ------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **User revokes consent**                    | User removes app permissions via https://myapps.microsoft.com                              |
| **Admin revokes or disables the app**       | Admin can revoke the service principal in Azure AD Enterprise Applications                 |
| **Password change / account disabled**      | Changing password or disabling account invalidates existing refresh tokens                 |
| **Conditional access or MFA policy change** | New tenant policies (MFA, location requirements) invalidate old refresh tokens             |
| **Refresh token inactive for 90 days**      | If not used to get new access tokens for 90 days, it expires                               |
| **Application permission changes**          | Changing requested scopes or app registration requires re-consent, invalidating old tokens |
| **User signs out of all sessions**          | "Sign out everywhere" action kills all refresh tokens                                      |

:::tip EmailEngine Keeps Tokens Active
EmailEngine automatically uses refresh tokens to obtain new access tokens, which resets the 90-day inactivity timer for Microsoft and the 6-month timer for Google. As long as accounts remain connected in EmailEngine, tokens stay active.
:::

:::danger Microsoft Client Secret Expiration
Microsoft OAuth2 **client secrets expire** and must be renewed regularly. The expiry is chosen when the secret is created in Azure AD, and the portal allows at most **24 months**. When a client secret expires, EmailEngine can no longer refresh access tokens, causing **all accounts using that OAuth2 app to fail** immediately.

**To prevent service disruption:**
1. Monitor client secret expiration in Azure AD: **Certificates & secrets** section
2. Generate a new client secret **before** the current one expires
3. Update EmailEngine configuration with the new client secret
4. Azure AD allows multiple active secrets simultaneously, so you can add the new secret before removing the old one

**If the secret expires:**
- Every account bound to that OAuth2 app fails authentication and stops syncing
- Replacing the secret is enough to recover them. Refresh tokens are bound to the application registration rather than to the secret, so the users do not have to authorize again
- An outage that runs longer than [`EENGINE_MAX_IMAP_AUTH_FAILURE_TIME`](/docs/configuration/environment-variables#max-imap-auth-failure-time), three days by default, is the exception: those accounts are [switched off](/docs/accounts/managing-accounts#accounts-switched-off-after-authentication-failures) and stay off after the secret is fixed. Use **Resume syncing** on each account page, or re-enable them through the API
- No stored data is lost either way
:::

#### Token Lifetime Summary

| Provider      | Access Token | Refresh Token (Active)                                        | Refresh Token (Inactive) |
| ------------- | ------------ | ------------------------------------------------------------- | ------------------------ |
| **Google**    | 1 hour       | No fixed lifetime; invalidated by the conditions listed above | Expires after 6 months   |
| **Microsoft** | 1 hour       | Replaced on every refresh; invalidated by the conditions above | Expires after 90 days    |

## Using Tokens with Provider APIs

All examples below use the [OAuth2 Token API](/docs/api/get-v-1-account-account-oauthtoken) to retrieve access tokens.

### Google API Example

Retrieve token from EmailEngine using the [OAuth2 Token API](/docs/api/get-v-1-account-account-oauthtoken):

```bash
curl https://emailengine.example.com/v1/account/example/oauth-token \
  -H "Authorization: Bearer YOUR_EMAILENGINE_TOKEN"
```

Use the access token with Google APIs:

```bash
curl https://gmailpostmastertools.googleapis.com/v1/domains \
  -H "Authorization: Bearer ya29.a0AVA9y1sXQ....CP1A"
```

**Response:**

```json
{
  "domains": [
    {
      "name": "domains/example.com",
      "createTime": "2020-01-15T12:30:00Z",
      "permission": "OWNER"
    }
  ]
}
```

### Google Calendar Example

```bash
# Get token
TOKEN=$(curl -s https://emailengine.example.com/v1/account/example/oauth-token \
  -H "Authorization: Bearer YOUR_EMAILENGINE_TOKEN" \
  | jq -r '.accessToken')

# List calendars
curl https://www.googleapis.com/calendar/v3/users/me/calendarList \
  -H "Authorization: Bearer $TOKEN"
```

### Microsoft Graph Example

```bash
# Get token
curl https://emailengine.example.com/v1/account/example/oauth-token \
  -H "Authorization: Bearer YOUR_EMAILENGINE_TOKEN"

# Use with Microsoft Graph
curl https://graph.microsoft.com/v1.0/me/calendars \
  -H "Authorization: Bearer EwBIA8l6..."
```

### Example: Creating a Calendar Event

Using EmailEngine's OAuth2 tokens to create a Google Calendar event:

```javascript
// Get current access token from EmailEngine
const tokenResponse = await fetch("https://emailengine.example.com/v1/account/user123/oauth-token", {
  headers: {
    Authorization: "Bearer YOUR_EMAILENGINE_TOKEN",
  },
});

const { accessToken } = await tokenResponse.json();

// Create calendar event
const eventResponse = await fetch("https://www.googleapis.com/calendar/v3/calendars/primary/events", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${accessToken}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    summary: "Team Meeting",
    start: {
      dateTime: "2024-01-15T10:00:00+02:00",
      timeZone: "Europe/Tallinn",
    },
    end: {
      dateTime: "2024-01-15T11:00:00+02:00",
      timeZone: "Europe/Tallinn",
    },
    attendees: [{ email: "colleague@example.com" }],
  }),
});

const event = await eventResponse.json();
console.log("Event created:", event.htmlLink);
```

## Token Refresh

EmailEngine handles token refresh automatically:

**When EmailEngine Refreshes Tokens:**

- Before opening an IMAP or SMTP connection, when the stored token has expired or expires within 30 seconds
- Before a Gmail API or MS Graph request, on the same condition
- When you request a token via the API endpoint, on the same condition

**What EmailEngine Does:**

1. Uses the refresh token to obtain a new access token
2. Stores the new token and its expiry time in Redis, encrypted when `EENGINE_SECRET` is set
3. Proceeds with the connection or request using the new token

A refresh that the provider rejects is reported as an [`authenticationError`](/docs/webhooks/authenticationerror) webhook; a transient network failure is retried rather than treated as a rejection.

**Your Responsibility:**

- Retrieve tokens when you need them via the API endpoint; the token returned is valid at the time of the response

:::tip Caching Tokens
You can cache a token in your application until its `expires` time. Requesting it from EmailEngine on demand is also fine: when the stored token is still valid the response comes from Redis, with `cached` set to `true`.
:::

## See Also

- [OAuth2 setup](/docs/accounts/oauth2-setup) - Registering the provider application and its scopes
- [Gmail API scopes](/docs/accounts/gmail/gmail-api-scopes) - What each Google scope combination allows
- [Authentication server](/docs/accounts/authentication-server) - Keeping token management outside EmailEngine entirely
- [authenticationError webhook](/docs/webhooks/authenticationerror) - Being told when a token stops working
- [Managing accounts](/docs/accounts/managing-accounts) - Re-enabling an account that was parked
