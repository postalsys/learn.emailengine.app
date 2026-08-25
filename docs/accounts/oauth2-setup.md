---
title: OAuth2 Setup Guide
sidebar_position: 7
description: Understanding OAuth2 authentication and setting up OAuth2 applications with EmailEngine
---

# OAuth2 Setup Guide

This guide explains OAuth2 authentication concepts and how to set up OAuth2 applications with EmailEngine. For provider-specific instructions, see the dedicated guides for [Gmail](./gmail/gmail-imap), [Outlook](./microsoft-365/outlook-365), [Mail.ru](./mail-ru), or [Google Service Accounts](./gmail/google-service-accounts).

## What is OAuth2?

OAuth2 is an authorization framework that allows applications to access user data without requiring passwords. Instead of storing passwords, your application receives temporary access tokens that can be refreshed automatically.

### Benefits of OAuth2

**Better Security:**

- No password storage required
- Tokens can be revoked without changing passwords
- Scoped permissions (request only what you need)
- Works with accounts that have two-factor authentication enabled

**Better User Experience:**

- Users authenticate directly with their provider (Google, Microsoft)
- Familiar consent screens
- Users can review and revoke access anytime
- No need to share passwords with third parties

**Better Compliance:**

- Meets modern security standards
- Supports enterprise security policies
- Audit trail of access grants
- Automatic token expiration

## OAuth2 Flow Overview

Understanding the OAuth2 flow helps troubleshoot issues:

```mermaid
flowchart TD
    A[Your Application] -->|1. generates authentication URL| B[EmailEngine]
    B -->|2. redirects user| C[Provider consent screen]
    C -->|3. user approves| D[Provider redirects to EmailEngine]
    D --> E[4. EmailEngine exchanges code for tokens]
    E --> F[5. EmailEngine stores tokens and connects to mailbox]
    F -->|6. redirects user to redirectUrl| G[Your Application]
    F --- H[Ongoing: auto-refreshes tokens before expiry]
```

EmailEngine handles the token exchange and refresh automatically. You just need to:

- Set up the OAuth2 application with the provider (one-time)
- Direct users to the EmailEngine authentication URL
- EmailEngine handles token exchange, storage, and refresh

## Supported Providers in EmailEngine

EmailEngine supports OAuth2 for:

### Gmail / Google Workspace

**Protocol Options:**

- **IMAP/SMTP with OAuth2** - Standard protocols with OAuth2 authentication
- **Gmail API** - Native Gmail REST API (faster, more features)

**Account Types:**

- **Internal Apps** - Google Workspace organization only
- **Service Accounts** - Google Workspace with domain-wide delegation
- **Public Apps** - Any Gmail user (requires security audit)

[Gmail IMAP OAuth2 Setup →](./gmail/gmail-imap)
[Gmail API Setup →](./gmail/gmail-api)
[Google Service Accounts →](./gmail/google-service-accounts)

### Outlook / Microsoft 365

**Protocol Options:**

- **IMAP/SMTP with OAuth2** - Standard protocols with OAuth2 authentication
- **Microsoft Graph API** - Native Microsoft 365 API

**Account Types:**

- **Single Tenant** - Your organization only
- **Multi-Tenant** - Any Microsoft 365 organization
- **Personal Accounts** - Outlook.com, Hotmail.com, Live.com
- **Combined** - Organizations + personal accounts

**Application Access (Client Credentials):**

- **Application permissions** - App authenticates as itself, no user login needed
- Uses `client_credentials` grant type with MS Graph API only
- Requires Azure AD admin consent and a specific tenant ID
- Best for automated systems accessing mailboxes without user interaction

[Outlook OAuth2 Setup →](./microsoft-365/outlook-365)
[Outlook Application Access →](./microsoft-365/outlook-client-credentials)

### Mail.ru

**Protocol Options:**

- **IMAP/SMTP with OAuth2** - Standard protocols with OAuth2 authentication

**Required Scopes:**

- `userinfo` - Basic user profile
- `mail.imap` - IMAP access

[Mail.ru OAuth2 Setup →](./mail-ru)

## Setting Up OAuth2 in EmailEngine

### Overview of Steps

1. **Create OAuth2 app** with the provider (Google Cloud Console or Azure AD)
2. **Configure consent screen** and permissions
3. **Get credentials** (Client ID and Client Secret)
4. **Add to EmailEngine** via Integrations → OAuth2 Apps
5. **Test** by adding an account

### OAuth2 Application Types

EmailEngine supports different OAuth2 application types:

**Gmail:**

- Gmail (IMAP/SMTP)
- Gmail API
- Gmail Service Accounts

**Outlook:**

- Outlook (delegated) - IMAP/SMTP or MS Graph API with user login
- Outlook (application) - MS Graph API with client credentials (no user login)

Each type has its own configuration page in EmailEngine.

### Required Information

When configuring OAuth2 in EmailEngine, you'll need:

**From Provider (Google/Microsoft):**

- **Client ID** - Identifies your application
- **Client Secret** - Authenticates your application
- **Redirect URI** - Where users return after consent

**For EmailEngine:**

- **Application Name** - Internal identifier
- **Base Scope** - Protocol to use (IMAP/SMTP or API)
- **Enabled** - Whether app shows in authentication forms

### Configuration in EmailEngine

Navigate to **Integrations** > **OAuth2 Apps** in EmailEngine dashboard.

![OAuth2 configuration page](/img/oauth2-setup/01-oauth2-config-empty.png)

**Add New Application:**

1. Click the **Add application** dropdown and select the provider type (Gmail or Outlook)

![Add application dropdown menu](/img/oauth2-setup/02-oauth2-add-app-menu.png)

2. Fill in the application details:

![Outlook OAuth2 form](/img/oauth2-setup/03-oauth2-outlook-form-empty.png)

- **Name**: Internal identifier (e.g., "Production Outlook")
- **Enabled**: Check to make available to users
- **Client ID**: From provider console (Azure AD Application ID or Google Client ID)
- **Client Secret**: From provider console
- **Redirect URL**: Your EmailEngine URL + `/oauth`
- **Base Scope**: Choose protocol (IMAP/SMTP or MS Graph API)
- **Authority** (Outlook): Tenant configuration (`common`, `organizations`, `consumers`, or specific tenant ID)

![Filled OAuth2 form](/img/oauth2-setup/04-oauth2-outlook-form-filled.png)

3. Click **Register app** to save

![OAuth2 app created](/img/oauth2-setup/05-oauth2-app-created.png)

The application now appears in your OAuth2 configuration list:

![OAuth2 configuration with app](/img/oauth2-setup/06-oauth2-config-with-app.png)

### Credentials File Upload

Both Google and Microsoft allow downloading credentials as JSON files:

**Google credentials file** (starts with `client_secret_`):

```json
{
  "web": {
    "client_id": "123456789.apps.googleusercontent.com",
    "client_secret": "abcdef123456",
    "redirect_uris": ["http://localhost:3000/oauth"]
  }
}
```

**Microsoft credentials** (entered manually):

- Application (client) ID
- Client secret value

EmailEngine can auto-fill fields from Google's JSON file.

### Verifying the App Setup

After configuring an OAuth2 app, use the **Verify setup** button on the app's page in the EmailEngine dashboard (or the [`POST /v1/oauth2/{app}/verify`](/docs/api/post-v-1-oauth-2-app-verify) API endpoint) to test the configuration before connecting accounts.

The verifier runs the real provider authentication chain step by step and reports each step as passed, failed, or skipped, with an actionable hint for failures:

- **Service-account apps (Gmail/Outlook)** - checks the credentials, token exchange, and domain-wide delegation. Provide a test mailbox address to also perform a live IMAP login or Gmail/MS Graph API probe.
- **Regular (3-legged) apps** - performs a configuration-only check, since user consent is required for a full token exchange.

The verifier is read-only: it does not modify any mailbox and does not send mail. This makes misconfigurations visible immediately instead of surfacing only when the first account fails to connect.

## OAuth2 Scopes

Scopes define what your application can access.

### Gmail Scopes

**For IMAP/SMTP:**

| Scope                      | Purpose                              |
| -------------------------- | ------------------------------------ |
| `https://mail.google.com/` | Full IMAP and SMTP access (required) |

**For Gmail API:**

| Scope          | Purpose                                                              |
| -------------- | -------------------------------------------------------------------- |
| `gmail.modify` | Full Gmail API access (read, write, delete, but not admin functions) |

**Narrower Scopes (if required by Google):**

| Scope            | Access                       | Notes                                              |
| ---------------- | ---------------------------- | -------------------------------------------------- |
| `gmail.readonly` | Read-only access             | Must be combined with `gmail.labels` (see below)    |
| `gmail.send`     | Send emails only             | Cannot read emails                                  |
| `gmail.labels`   | List and manage labels       | Required with `gmail.readonly` for EmailEngine      |

:::warning gmail.readonly requires gmail.labels
The `gmail.labels` scope is required for EmailEngine to list labels (mailboxes) properly. Always combine `gmail.readonly` with `gmail.labels` for read-only access to work.
:::

See the [Gmail API Scopes Reference](/docs/accounts/gmail/gmail-api-scopes) for detailed setup instructions and EmailEngine feature availability for each scope combination.

### Outlook Scopes

**For IMAP/SMTP:**

| Scope                   | Purpose                                   |
| ----------------------- | ----------------------------------------- |
| `IMAP.AccessAsUser.All` | Read and manage emails via IMAP           |
| `SMTP.Send`             | Send emails via SMTP (enabled by default) |
| `offline_access`        | Allow token refresh (required)            |

If your application doesn't need sending capabilities, you can disable `SMTP.Send` via the **Disabled scopes** field in EmailEngine's OAuth2 app settings.

**For MS Graph API:**

| Scope            | Purpose                        |
| ---------------- | ------------------------------ |
| `Mail.ReadWrite` | Read and manage emails         |
| `Mail.Send`      | Send emails                    |
| `offline_access` | Allow token refresh (required) |

:::info offline_access Scope
The `offline_access` scope allows EmailEngine to refresh access tokens in the background without user interaction.

- **Gmail**: Enabled by default, no configuration needed
- **Outlook**: Must be explicitly requested (EmailEngine handles this automatically)
  :::

### Additional Scopes

You can add extra scopes if you want to use OAuth2 tokens for other APIs:

**Example - Add Google Calendar access:**

```
https://www.googleapis.com/auth/calendar
```

Configure this in **Additional scopes** field in EmailEngine.

:::warning Microsoft Additional Scopes
For Microsoft accounts, additional scopes only work when using **MS Graph API** as the base scope. If you're using **IMAP/SMTP**, additional scopes won't be usable because IMAP/SMTP uses the Outlook API, not MS Graph API. You can still add the scopes, but you won't be able to make API requests to those endpoints.
:::

[Learn more about using tokens for other APIs →](./oauth2-token-management)

### Disabled Scopes

If Google/Microsoft requires narrower scopes, you can disable the default wide scope:

**Disabled scopes** field:

```
https://mail.google.com/
```

This removes the wide scope from consent requests.

## Account Types and Tenant Configuration

### Gmail Account Types

**Internal Apps:**

- Only for Google Workspace organizations
- No security audit required
- Limited to organization's domain

**External Apps (Development):**

- Limited to 100 manually whitelisted users
- Grants expire after 7 days
- Not suitable for production

**External Apps (Production):**

- Available to any Gmail user
- Requires thorough security audit (expensive, time-consuming)
- Google may reject broad scopes like `https://mail.google.com/`

### Outlook Account Types

Configure via **Supported account types** field:

- `common` - Organizations + personal accounts (most flexible)
- `consumers` - Personal accounts only (@outlook.com, @hotmail.com)
- `organizations` - Microsoft 365 organizations only
- `<directory-id>` - Specific organization only (use Directory/Tenant ID)

**Mapping to Azure:**

| Azure Setting       | EmailEngine Value |
| ------------------- | ----------------- |
| Any org + personal  | `common`          |
| Personal only       | `consumers`       |
| Any organization    | `organizations`   |
| Single organization | Use Directory ID  |

## Redirect URLs

The redirect URL is where users return after granting consent.

### Format

```
{emailengine-url}/oauth
```

**Examples:**

- `http://localhost:3000/oauth` (development)
- `https://ee.company.com/oauth` (production)
- `https://your-domain.com/oauth` (custom domain)

### Requirements

- Must be HTTPS in production (HTTP allowed for localhost)
- Must match exactly between provider and EmailEngine
- Case-sensitive
- No trailing slashes
- Must be publicly accessible (for user redirects)

## Advanced OAuth2 Features

### Authentication Server

Delegate OAuth2 management to an external server:

```json
{
  "authServer": "https://your-auth-server.com/authenticate"
}
```

Use this when:

- You already manage OAuth2 in your app
- Want centralized token management
- Need custom authentication flows

[Learn more about authentication servers →](./authentication-server)

### Pre-filled Email

When generating authentication URLs, you can pre-fill the user's email:

```bash
curl -X POST https://emailengine.example.com/v1/authentication/form \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user123",
    "email": "user@gmail.com",
    "redirectUrl": "https://myapp.com/settings"
  }'
```

### Delegated Access (Shared Mailboxes)

For Microsoft 365 shared mailboxes:

```bash
curl -X POST https://emailengine.example.com/v1/authentication/form \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "shared-support",
    "email": "support@company.com",
    "delegated": true,
    "redirectUrl": "https://myapp.com/settings"
  }'
```

[Learn more about shared mailboxes →](./microsoft-365/outlook-365#shared-mailboxes)

### Service Accounts (Google Workspace)

Access any user's mailbox without individual consent:

```json
{
  "oauth2": {
    "provider": "AAABhaBPHscAAAAI",
    "auth": {
      "user": "user@company.com"
    }
  }
}
```

`provider` is the ID of the service-account application you registered in EmailEngine.

Requires domain-wide delegation setup.

[Learn more about service accounts →](./gmail/google-service-accounts)

## See Also

- [Gmail OAuth2 over IMAP](/docs/accounts/gmail/gmail-imap) - The Google Cloud side, step by step
- [Outlook OAuth2](/docs/accounts/microsoft-365/outlook-365) - The Azure side, step by step
- [Gmail API scopes](/docs/accounts/gmail/gmail-api-scopes) - Which scope combination supports which feature
- [OAuth2 token management](/docs/accounts/oauth2-token-management) - Using the stored tokens against other provider APIs
- [Hosted authentication](/docs/accounts/hosted-authentication) - Sending users through the consent flow
