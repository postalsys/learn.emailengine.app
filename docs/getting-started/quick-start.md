---
title: Quick Start Guide
sidebar_position: 1
description: From an empty install to a sent message and a webhook, one step at a time
---

# Quick Start Guide

This guide installs EmailEngine, connects an email account to it, sends a message, and shows the webhooks that arrive. Every command runs against a local instance.

## Step 1: Install EmailEngine

### Option A: Download Binary (Quickest)

```bash
# Download latest release
wget https://go.emailengine.app/emailengine.tar.gz
tar xzf emailengine.tar.gz
chmod +x emailengine

# Start EmailEngine (default Redis database is 8)
./emailengine --dbs.redis="redis://127.0.0.1:6379/8"
```

### Option B: Using Docker

```bash
# Pull the latest version
docker pull postalsys/emailengine:v2

# Run EmailEngine
docker run -p 3000:3000 \
  --env EENGINE_REDIS="redis://host.docker.internal:6379/8" \
  postalsys/emailengine:v2
```

EmailEngine will start on **http://localhost:3000**

:::warning Production Security
For production deployments, you **must** configure `EENGINE_SECRET` to encrypt stored credentials. Without this setting, account passwords and OAuth tokens are stored unencrypted in Redis.

Generate the secret once, store it permanently, and pass the same value to EmailEngine on every start:

```bash
# Generate the secret once and persist it
echo "EENGINE_SECRET=$(openssl rand -hex 32)" >> .env

# Pass the persisted value to the container on every start
docker run -p 3000:3000 \
  --env-file .env \
  --env EENGINE_REDIS="redis://host.docker.internal:6379/8" \
  postalsys/emailengine:v2
```

See [Credential Security](/docs/support/security-faq) for details on how EmailEngine protects your data.
:::

## Step 2: Access the Web Interface

1. Open **http://localhost:3000** in a browser. A fresh instance has no admin password, so the dashboard opens without a login and shows an "Authentication not enabled" card
2. Click **Enable authentication** on that card and set a password. From then on the admin interface asks for it

Until a password is set, the admin interface refuses to issue access tokens, which is what the next step needs.

![EmailEngine Web Interface](/img/screenshots/01-dashboard-main.png)
_EmailEngine dashboard showing system statistics and account overview_

## Step 3: Generate an API Access Token

You need an access token to authenticate API requests.

### Via CLI (Recommended):

Use the EmailEngine CLI to generate a full-access token:

```bash
emailengine tokens issue -d "Development" -s "*" --dbs.redis="redis://127.0.0.1:6379/8"
8bf639ec7c051c3963498c6757b6813bd331afeb677886d4473190fae66c9fab
```

Save your token as an environment variable:

```bash
export EMAILENGINE_TOKEN="8bf639ec7c051c3963498c6757b6813bd331afeb677886d4473190fae66c9fab"
```

The CLI writes the token straight into Redis, so `--dbs.redis` must name the same database EmailEngine runs on. A `*` token reaches every account and every endpoint, including settings, which is what the rest of this guide needs.

### Via Web Interface:

1. Open **Integrations** > **Access Tokens** in the sidebar
2. Click **Create access token**
3. Provide a description (for example, "Development")
4. Select the scope (`*` for full access)
5. Click **Generate a token**
6. Copy the token immediately: it is shown only once

:::info What the API can mint
The [Create Access Token API](/docs/api/post-v-1-tokens) needs either an `account`, which binds the token to that one mailbox, or a `permissions` record naming what the token may do. It will not mint an unrestricted token, so the CLI and the admin interface remain the way to get one. See [Access Tokens](/docs/api-reference/access-tokens).
:::

## Step 4: Add Your First Email Account

EmailEngine supports multiple account types. Choose the method that fits your needs:

### Option A: Hosted Authentication Form (Recommended)

The easiest way to add accounts is using EmailEngine's built-in hosted authentication form. EmailEngine handles the entire OAuth2 flow for you.

**Benefits:**

- No need to obtain OAuth2 tokens manually
- EmailEngine manages token refresh automatically
- Works with Gmail, Outlook, and other OAuth2 providers
- User-friendly authentication experience

**How to use:**

1. Your application generates an authentication form URL via the API:

   ```bash
   curl -XPOST "http://localhost:3000/v1/authentication/form" \
     -H "Authorization: Bearer ${EMAILENGINE_TOKEN}" \
     -H "Content-Type: application/json" \
     -d '{
       "account": "user123",
       "redirectUrl": "https://your-app.com/callback"
     }'
   ```

2. Redirect the user's browser to the generated form URL

3. User chooses authentication method:

   ![Account type selection page with OAuth2 and IMAP options](/img/screenshots/03-account-type-selection.png)
   _Account type selection page showing OAuth2 provider buttons and standard IMAP option_

   - **OAuth2 Provider** (Gmail/Outlook): Clicks provider button, follows OAuth2 flow
   - **IMAP/SMTP**: Enters email/password, EmailEngine auto-detects server settings

4. After authentication, user is redirected back to your `redirectUrl`

5. EmailEngine handles token management and maintains the connection automatically

:::info OAuth2 Setup Required
OAuth2 provider buttons (Gmail, Outlook) only appear if OAuth2 apps are configured in EmailEngine. Set these up via **Integrations > OAuth2 Apps** in the web interface, or use the [OAuth2 Apps API](/docs/api/post-v-1-oauth-2). See [Gmail OAuth2 Setup](/docs/accounts/gmail/gmail-imap) and [Outlook OAuth2 Setup](/docs/accounts/microsoft-365/outlook-365) for detailed configuration guides.
:::

:::tip Admin Testing
For testing during development, admins can use the web interface: navigate to **Accounts > Add an account**. This is a convenience wrapper for the same authentication form.
:::

![Accounts List](/img/screenshots/02-accounts-list.png)
_Accounts page showing empty account list with "Add an account" button_

![Add Account Form](/img/screenshots/04-account-add-form.png)
_Auto-detected IMAP/SMTP configuration showing server settings, ports, and TLS options_

**When to use the hosted form:**

- Quick account setup during development
- End-user account authentication in your application
- When you don't need programmatic control over OAuth2 tokens

:::tip Using in Your Application
You can redirect users to the hosted authentication form from your application. The form will redirect users to the OAuth2 provider's permission page (which must open in a main window, not an iframe due to security restrictions). After authentication, users are redirected back to your application. See the [Authentication Form API](/docs/api/post-v-1-authentication-form) for details.
:::

### Option B: API with OAuth2 Tokens (Advanced)

For programmatic control or special requirements, register accounts directly via API with OAuth2 tokens you've obtained.

**When to use direct API:**

- Automated account provisioning
- Service account authentication
- Custom OAuth2 flows
- When you need to manage tokens yourself

**Gmail Example:**

**Prerequisites:** You need to set up OAuth2 credentials in Google Cloud Console. [See detailed guide →](/docs/accounts/gmail/gmail-imap)

```bash
curl -XPOST "http://localhost:3000/v1/account" \
  -H "Authorization: Bearer ${EMAILENGINE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "my-gmail",
    "name": "Your Name",
    "email": "you@gmail.com",
    "oauth2": {
      "provider": "AAABlf_0iLgAAAAQ",
      "refreshToken": "1//0gExampleRefreshTokenFromGoogle",
      "auth": {
        "user": "you@gmail.com"
      }
    }
  }'
```

:::info Provider ID
The `provider` value should be your OAuth2 application ID from EmailEngine, which is a base64url encoded string like `AAABlf_0iLgAAAAQ`. Find this in **Integrations > OAuth2 Apps**.
:::

**Outlook Example:**

**Prerequisites:** You need to register an app in Azure AD. [See detailed guide →](/docs/accounts/microsoft-365/outlook-365)

```bash
curl -XPOST "http://localhost:3000/v1/account" \
  -H "Authorization: Bearer ${EMAILENGINE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "my-outlook",
    "name": "Your Name",
    "email": "you@outlook.com",
    "oauth2": {
      "provider": "AAABlf_0iLgAAAAQ",
      "refreshToken": "M.C546_ExampleRefreshTokenFromMicrosoft",
      "auth": {
        "user": "you@outlook.com"
      }
    }
  }'
```

### Option C: Generic IMAP/SMTP (Any provider)

This works with any email provider that supports IMAP and SMTP:

```bash
curl -XPOST "http://localhost:3000/v1/account" \
  -H "Authorization: Bearer ${EMAILENGINE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "my-account",
    "name": "Your Name",
    "email": "you@example.com",
    "imap": {
      "host": "imap.example.com",
      "port": 993,
      "secure": true,
      "auth": {
        "user": "you@example.com",
        "pass": "your-password"
      }
    },
    "smtp": {
      "host": "smtp.example.com",
      "port": 465,
      "secure": true,
      "auth": {
        "user": "you@example.com",
        "pass": "your-password"
      }
    }
  }'
```

### Common Provider Settings

| Provider    | IMAP Host             | IMAP Port | SMTP Host           | SMTP Port |
| ----------- | --------------------- | --------- | ------------------- | --------- |
| Gmail       | imap.gmail.com        | 993       | smtp.gmail.com      | 465       |
| Outlook.com | outlook.office365.com | 993       | smtp.office365.com  | 587       |
| Yahoo       | imap.mail.yahoo.com   | 993       | smtp.mail.yahoo.com | 465       |
| Fastmail    | imap.fastmail.com     | 993       | smtp.fastmail.com   | 465       |

**Response:**

```json
{
  "account": "my-account",
  "state": "new"
}
```

### Provider-Specific Notes

#### Gmail

- **Account passwords do not work:** Google no longer accepts the account password over IMAP. Either enable 2-step verification and generate an app password, or use OAuth2
- **OAuth2 refreshes itself:** EmailEngine renews the access token from the refresh token, so nothing expires on your side
- **OAuth2 verification:** A Google Cloud OAuth2 app used by accounts outside your own organization has to pass Google's verification
- [Complete Gmail setup →](/docs/accounts/gmail/gmail-imap)

#### Outlook/Microsoft 365

- **OAuth2:** Microsoft has retired basic authentication for Exchange Online, so an app registration in Microsoft Entra is needed
- **Permissions:** The registration needs the IMAP and SMTP permissions, or the Graph API permissions when using the Graph backend
- [Complete Outlook setup →](/docs/accounts/microsoft-365/outlook-365)

#### Yahoo Mail

- **App password required:** Generate one in the Yahoo account security settings, then use it with the IMAP/SMTP settings in the table above

#### ProtonMail

- **Bridge required:** Proton Mail exposes IMAP and SMTP only through the Proton Mail Bridge application, which listens on localhost with its own ports and credentials. EmailEngine connects to the Bridge, not to Proton's servers

## Step 5: Wait for Initial Sync

EmailEngine performs an initial sync of your mailbox before it's ready to use.

### Option A: Using Webhooks (Recommended)

Once the account completes its initial sync, EmailEngine automatically sends an `accountInitialized` webhook event:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "event": "accountInitialized",
  "account": "my-account",
  "date": "2025-01-15T10:23:45.678Z",
  "data": {
    "initialized": true
  }
}
```

**Benefits:**

- No polling required
- Instant notification when account is ready
- Efficient for production applications

### Option B: Polling Account Status

Alternatively, poll the [get account API](/docs/api/get-v-1-account-account) until `state` shows `"connected"`:

```bash
# Check account status
curl "http://localhost:3000/v1/account/my-account" \
  -H "Authorization: Bearer ${EMAILENGINE_TOKEN}"
```

**Response when ready:**

```json
{
  "account": "my-account",
  "name": "Your Name",
  "email": "you@example.com",
  "state": "connected",
  "lastError": null
}
```

Key fields to check:

- `state`: `"connected"` is the healthy steady state. `"init"`, `"connecting"` and `"syncing"` mean it is still on its way there; `"authenticationError"` and `"connectError"` mean it is not going to get there without your help. [Account States](/docs/accounts/managing-accounts#account-states) describes all nine
- `lastError`: `null` when the last connection attempt succeeded
- `authFailureDisabledAt` (since v2.79.4): normally absent. When set, EmailEngine switched syncing off after repeated authentication failures, and the state reads `"unset"` until working credentials are supplied

### Initial Sync Duration

Sync time depends on:

- **Number of mailbox folders** - More folders take longer to scan
- **Number of messages** - Each message needs to be indexed
- **Indexer type** - `imapIndexer` on the account, or the global setting, chooses between `full` (detects new, changed and deleted messages) and `fast` (detects new messages only)

## Step 6: Send Your First Email

Once the account is connected, send a test email using the [submit API](/docs/api/post-v-1-account-account-submit):

```bash
curl -XPOST "http://localhost:3000/v1/account/my-account/submit" \
  -H "Authorization: Bearer ${EMAILENGINE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "to": [{
      "name": "Test Recipient",
      "address": "recipient@example.com"
    }],
    "subject": "Test email from EmailEngine",
    "text": "Hello! This is my first email sent through EmailEngine.",
    "html": "<p>Hello! This is my first email sent through <strong>EmailEngine</strong>.</p>"
  }'
```

**Response:**

```json
{
  "response": "Queued for delivery",
  "messageId": "<unique-id@example.com>",
  "sendAt": "2025-01-15T10:25:30.123Z",
  "queueId": "abc123def456"
}
```

The email is now queued for delivery. EmailEngine will:

1. Submit it to the SMTP server
2. Retry automatically if delivery fails
3. Send webhooks for delivery status

## Step 7: Set Up Webhooks

Webhooks notify your application about email events in real-time.

### Configure Webhook URL

#### Via Web Interface:

1. Go to **Configuration** > **Webhooks**
2. Enable webhooks and enter your webhook URL (for example, `https://your-app.com/webhooks`)
3. Select the events you want to receive
4. Save settings

![Webhooks Configuration](/img/screenshots/05-webhooks-config.png)
_Webhooks configuration page in EmailEngine settings_

#### Via API:

Use the [update settings API](/docs/api/post-v-1-settings). Three settings are involved: the URL, the enable switch, and `webhookEvents`, which is an allowlist with no default. Leaving it unset delivers nothing; `["*"]` delivers every event:

```bash
curl -XPOST "http://localhost:3000/v1/settings" \
  -H "Authorization: Bearer ${EMAILENGINE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "webhooks": "https://your-app.com/webhooks",
    "webhooksEnabled": true,
    "webhookEvents": ["*"]
  }'
```

The response lists the keys that changed:

```json
{
  "updated": ["webhooks", "webhooksEnabled", "webhookEvents"]
}
```

### Test Webhooks with Webhook.site

For testing, use [webhook.site](https://webhook.site) to see webhook payloads:

1. Go to https://webhook.site - you'll get a unique URL
2. Copy the URL (e.g., `https://webhook.site/abc-123-def`)
3. Set it as your webhook URL in EmailEngine
4. Send a test email (Step 6)
5. Check webhook.site to see the delivery notifications

### What Arrives

After the test email from Step 6, a [messageSent](/docs/webhooks/messagesent) event reaches the endpoint once the SMTP server accepts the message, and a [messageNew](/docs/webhooks/messagenew) arrives for each message that lands in a watched folder from then on. Every payload has the same envelope, with the event-specific fields under `data`:

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "my-account",
  "date": "2025-01-15T10:25:31.000Z",
  "event": "messageSent",
  "data": {
    "messageId": "<unique-id@example.com>",
    "queueId": "abc123def456",
    "response": "250 2.0.0 OK: queued as A1B2C3",
    "envelope": {
      "from": "you@example.com",
      "to": ["recipient@example.com"]
    }
  }
}
```

A delivery that fails produces [messageDeliveryError](/docs/webhooks/messagedeliveryerror) for each attempt, and [messageFailed](/docs/webhooks/messagefailed) once the retries are exhausted. [Webhook events](/docs/reference/webhook-events) lists every event with its payload.

## Step 8: List Incoming Messages

Retrieve messages from the mailbox using the [list messages API](/docs/api/get-v-1-account-account-messages):

```bash
curl "http://localhost:3000/v1/account/my-account/messages?path=INBOX&pageSize=10" \
  -H "Authorization: Bearer ${EMAILENGINE_TOKEN}"
```

**Response:**

```json
{
  "total": 1234,
  "page": 0,
  "pages": 124,
  "messages": [
    {
      "id": "AAAAAQAACnA",
      "uid": 1234,
      "subject": "Meeting tomorrow",
      "from": {
        "name": "John Doe",
        "address": "john@example.com"
      },
      "date": "2025-01-15T10:30:00.000Z",
      "flags": ["\\Seen"],
      "labels": []
    }
  ]
}
```

## Step 9: Read a Specific Message

Read a message with the [get message API](/docs/api/get-v-1-account-account-message-message). Body content is left out unless `textType` asks for it, because fetching it costs a round trip to the mail server:

```bash
curl "http://localhost:3000/v1/account/my-account/message/AAAAAQAACnA?textType=*" \
  -H "Authorization: Bearer ${EMAILENGINE_TOKEN}"
```

## Step 10: Search Messages

Search for messages matching specific criteria using the [search messages API](/docs/api/post-v-1-account-account-search):

```bash
curl -XPOST "http://localhost:3000/v1/account/my-account/search?path=INBOX" \
  -H "Authorization: Bearer ${EMAILENGINE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "search": {
      "from": "john@example.com",
      "subject": "meeting"
    }
  }'
```

## Next Steps

### Set Up OAuth2

- **[Gmail OAuth2 Setup](/docs/accounts/gmail/gmail-imap)** - Complete guide with screenshots
- **[Outlook OAuth2 Setup](/docs/accounts/microsoft-365/outlook-365)** - Azure AD configuration
- **[OAuth2 Token Management](/docs/accounts/oauth2-token-management)** - Use tokens for other APIs

### Production Deployment

- **[Docker Installation](/docs/installation/docker)** - Production Docker setup
- **[Environment Variables](/docs/configuration/environment-variables)** - Configuration Reference
- **[Performance Tuning](/docs/advanced/performance-tuning)** - Optimize for high volume
- **[Security Best Practices](/docs/deployment/security)** - Secure your deployment

### Integration Examples

- **[CRM Integration](/docs/integrations/crm)** - Complete architecture guide
- **[AI/ChatGPT Integration](/docs/integrations/ai-chatgpt)** - Email summaries and analysis
- **[PHP Integration](/docs/integrations/php)** - Using the PHP library

## See Also

- [Managing accounts](/docs/accounts/managing-accounts) - The full account lifecycle beyond the first one
- [Webhooks overview](/docs/webhooks/overview) - Every event, and how delivery and retries work
- [API Reference](/docs/api-reference) - Authentication, conventions, and error handling
- [Troubleshooting](/docs/troubleshooting) - What to check when an account will not connect
- [Support](/docs/support) - Support channels and what a subscription covers
