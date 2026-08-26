---
title: Setting Up Gmail API
sidebar_label: API Setup
sidebar_position: 2
description: Configure EmailEngine to use Gmail REST API as the email backend with Cloud Pub/Sub webhooks
---

# Setting Up Gmail API

This guide shows you how to configure EmailEngine to use Gmail REST API as the email backend instead of IMAP/SMTP. With this setup, EmailEngine uses direct Gmail API calls for all operations and receives change notifications via Google Cloud Pub/Sub.

:::info IMAP vs Gmail API
The [IMAP/SMTP guide](./gmail-imap) shows how to configure EmailEngine to access Gmail over OAuth2 using IMAP and SMTP. That setup uses the OAuth2 tokens **only to authenticate IMAP/SMTP sessions**.

This guide covers using **Gmail REST API directly** as the email backend. EmailEngine does not open IMAP or SMTP sessions and instead performs all operations via Gmail REST API calls.
:::

## Why Use Gmail API Instead of IMAP?

**Use Gmail API when:**

- You want better Gmail category handling
- Google's verification process requires limited OAuth2 scopes
- You want to avoid IMAP connection limits

**Use IMAP/SMTP when:**

- You need the full `https://mail.google.com/` scope and can justify it to Google
- Your organization restricts Cloud Pub/Sub permissions
- You need raw SMTP features (e.g., custom envelope-from)
- You want to avoid additional GCP setup complexity
- You're migrating existing IMAP-based integrations

:::info OAuth2 Scope Requirements
**IMAP/SMTP** requires the full `https://mail.google.com/` scope. **Gmail API** can use more granular scopes such as `gmail.modify` (default), `gmail.readonly`, `gmail.send`, and `gmail.labels`.

Google requires public apps to request only the minimum scopes needed. EmailEngine supports all Gmail scope combinations - see the [Gmail API Scopes Reference](./gmail-api-scopes) for detailed configuration options, setup instructions, and Google verification requirements.
:::

## Overview of Setup Steps

Setting up Gmail API requires more steps than IMAP because you need:

1. A regular OAuth2 app for user authentication
2. A service account for managing Cloud Pub/Sub
3. Cloud Pub/Sub API enabled for webhook notifications

## Step 1: Create a Google Cloud Project

Go to [Google Cloud Console](https://console.cloud.google.com/) and open the project menu in the top navbar.

![Creating a new Google Cloud project](/img/external/6V0B1AgnvU.gif)

Click the "New project" button to start.

![Naming your project](/img/external/owSQLNV1_5.gif)

Name your project (e.g., "EmailEngine Gmail API").

![Waiting for project creation](/img/external/0B4b3JeP3t.gif)

Wait for the project to be created, then select it from the project menu.

## Step 2: Enable Required APIs

Click the hamburger menu (top-left) > **APIs & Services** > **Enabled APIs & Services**.

![Navigating to APIs & Services](/img/external/v3Flo-WBVG.gif)

### Enable Gmail API

Find and enable **Gmail API** for your project.

![Enabling Gmail API](/img/external/vz7Is1SAWe.gif)

This allows EmailEngine to perform Gmail REST API requests.

### Enable Cloud Pub/Sub API

Also find and enable **Cloud Pub/Sub API**.

![Enabling Cloud Pub/Sub API](/img/external/KwfF06xSzN.gif)

:::info Why Cloud Pub/Sub?
Gmail pushes change notifications (new messages, flag changes, etc.) to Google's Pub/Sub system, not directly to EmailEngine. EmailEngine sets up a Pub/Sub topic and subscription to receive these notifications and convert them into webhooks for your application.

Without Pub/Sub, EmailEngine would not know when changes occur on the email account and couldn't send real-time webhook notifications.
:::

## Step 3: Configure OAuth Consent Screen

The consent screen is shown to users when they authorize EmailEngine.

Click hamburger menu > **APIs & Services** > **OAuth consent screen**.

![Navigating to consent screen](/img/external/0h3kuzzsCN.gif)

### Choose User Type

![Selecting user type](/img/external/mT6n2spEgt.gif)

**Internal:**

- Only accounts from your Google Workspace organization
- No verification process required
- Best for testing and enterprise apps

**External:**

- Any Gmail user can authenticate
- Requires verification for production
- Best for public applications

For this tutorial, we'll use **Internal**. For production, select **External** and complete Google's verification process.

### Fill in App Information

![Configuring consent screen details](/img/external/FIRIMzunwz.gif)

Provide:

- **App name**: "EmailEngine" (or your application name)
- **User support email**: Your email address
- **Developer contact information**: Your email address
- **Application home page**: Your EmailEngine instance URL

Click **Save and continue**.

### Configure Scopes

Click **Add or remove scopes** and find `gmail.modify` in the list.

![Adding required scope](/img/external/BONjtoR9p6.gif)

Check `gmail.modify` and click **Update**.

:::important Required Scope
The `gmail.modify` scope is required for Gmail API access. This is different from the `https://mail.google.com/` scope used for IMAP/SMTP.

If Google's verification process determines you need different scopes (e.g., `gmail.readonly`, `gmail.send`, `gmail.labels`), see the [Custom Scopes section](#using-custom-scopes) below.
:::

![Saving consent screen configuration](/img/external/THYy7q5W6Z.gif)

Scroll down and click **Save and continue** to finish consent screen setup.

## Step 4: Create OAuth Credentials for User Authentication

Navigate to **APIs & Services** > **Credentials**.

![Navigating to credentials page](/img/external/7bDFveWih1.gif)

Click **Create credentials** > **OAuth client ID**.

![Creating OAuth client ID](/img/external/dd27iNGkH0.gif)

### Configure OAuth Client

![Configuring OAuth client details](/img/external/5gMPcI0kJe.gif)

**Application type:** Web application

**Authorized JavaScript origins:**
Add your EmailEngine URL:

- `http://127.0.0.1:3000` (for local testing)
- `https://emailengine.example.com` (for production)

**Authorized redirect URIs:**
Add your EmailEngine URL with `/oauth`:

- `http://127.0.0.1:3000/oauth`
- `https://emailengine.example.com/oauth`

Click **Create**.

### Download User Credentials

![Downloading OAuth credentials](/img/external/4UhRTwH9yL.gif)

Click the **Download** button. Save this file - you'll need it to configure EmailEngine.

:::tip Credential File Names

- **User OAuth credentials**: Filename starts with `client_secret_`
- **Service account credentials**: Uses service account name prefix

Keep these separate - you need both.
:::

## Step 5: Create Service Account for Pub/Sub Management

EmailEngine needs a service account with permissions to manage Pub/Sub topics and subscriptions.

On the **Credentials** page, navigate to the **Service Account management** page.

![Navigating to service accounts](/img/external/FztCvZP6it.gif)

Click **Create Service Account**.

### Configure Service Account

![Creating service account](/img/external/M5HVdcmnY8.gif)

**Service account name:** Choose any name (e.g., "EmailEngine Pub/Sub Manager")

**Role:** Select **Pub/Sub Admin** (or a compatible role that allows managing Pub/Sub queues, topics, and IAM policies)

:::important Required Permissions
The service account must have permissions to:

- Create and delete Pub/Sub topics
- Create and delete Pub/Sub subscriptions
- Manage IAM policies for Pub/Sub resources

The "Pub/Sub Admin" role provides all necessary permissions.
:::

Click **Create** to finish.

### Generate Service Account Keys

Once created, select the service account and navigate to **Manage Keys**.

![Generating service account keys](/img/external/VtJcozUfxY.gif)

Click **Add key** > **Create new key** > **JSON format**.

The key file will automatically download. Store it securely - you cannot download it again.

## Step 6: Configure EmailEngine - Service Account

Now configure EmailEngine to use the service account for managing webhooks.

### Add Gmail Service Account Application

![Creating service account app in EmailEngine](/img/external/YvOpC3QjWZ.gif)

1. Open EmailEngine dashboard
2. Navigate to **Integrations** > **OAuth2 Apps**
3. Click the **Create OAuth2 app** dropdown
4. Select **Gmail Service Accounts**

### Configure Service Account Settings

![Configuring service account](/img/external/OfoPs4TldB.gif)

**Application name:** Give it a descriptive name (e.g., "Gmail Pub/Sub Manager")

**Load configuration from the service key file:** Select the **service account** JSON file (not the user OAuth credentials)

**Base scopes:** Select **Cloud Pub/Sub**

:::warning Use Correct Credentials File
Make sure you're uploading the **service account** credentials file, not the user OAuth credentials file. You can tell them apart:

- **Service account**: Filename uses service account name prefix
- **User OAuth**: Filename starts with `client_secret_`
  :::

Click **Register app** to save. Registering the app creates the Pub/Sub topic and subscription in your project; a setup failure is recorded on the app and shown under **Gmail Subscriptions**.

## Step 7: Configure EmailEngine - User OAuth

Now configure the user OAuth application that will authenticate Gmail accounts.

### Add Gmail OAuth2 Application

![Creating Gmail OAuth2 app](/img/external/cJspELPMDV.gif)

1. Navigate to **Integrations** > **OAuth2 Apps**
2. Click the **Create OAuth2 app** dropdown
3. Select **Gmail**

### Configure OAuth2 Settings

![Configuring Gmail OAuth2 settings](/img/external/vj8qeSQt6D.gif)

**Application name:** Give it a descriptive name (e.g., "Gmail API OAuth2")

**Enable this app:** Check this box

**Load configuration from the JSON file:** Select the **user OAuth credentials** file (the one whose name starts with `client_secret_`)

**Redirect URL:** Verify this matches exactly what you entered in Google Cloud Console

**Base scopes:** Select **Gmail API**

**Select service account to manage webhooks:** Select the service account app you created in Step 6. The selector only lists service accounts registered for the same Google Cloud project ID

:::important Link Service Account
This selector is marked as optional in the UI. It tells EmailEngine which credentials to use for managing Pub/Sub resources. Without it, EmailEngine registers no Gmail watch and instead polls each account for changes about every 10 minutes, so webhooks still fire, with up to that much delay. Send-only accounts never need it.
:::

### Configuring Limited Scopes

If Google requires you to use limited scopes during verification, you can configure EmailEngine to request only the scopes you need. The Web UI provides preset buttons (**Normal**, **Read-Only**, **Read-Only + Send**, **Send-Only**) that auto-populate the scope fields.

See the [Gmail API Scopes Reference](./gmail-api-scopes) for all supported scope combinations, what each enables in EmailEngine, and detailed setup instructions for both the Web UI and API.

Click **Register app** to save.

## Step 8: Test the Setup

Add a Gmail account to test the complete flow.

### Via Hosted Authentication Form

![Testing with hosted authentication](/img/external/5OA36VmtxU.gif)

1. In EmailEngine, click **Add account**
2. Click **Sign in with Google**
3. Complete the OAuth2 consent flow
4. EmailEngine will:
   - Store OAuth2 tokens
   - Register a Gmail watch that points to the Pub/Sub topic created when the service account app was registered
   - Start receiving webhook notifications

### Via API

Generate an authentication form URL:

```bash
curl -X POST https://emailengine.example.com/v1/authentication/form \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user123",
    "email": "user@gmail.com",
    "redirectUrl": "https://myapp.com/settings"
  }'
```

Direct the user to the returned URL.

### Verify Setup

Check the account status in EmailEngine:

- Account should enter "connected" state
- In Google Cloud Console > Pub/Sub, you should see:
  - The topic and subscription that EmailEngine created when you registered the service account app

## Using Custom Scopes

If you have a public OAuth2 application and Google requires narrower scopes than `gmail.modify`, you can configure custom scopes using the **Additional scopes** and **Disabled scopes** fields in the OAuth2 application configuration.

See the [Gmail API Scopes Reference](./gmail-api-scopes) for all supported scope combinations with setup instructions for both the Web UI and API, EmailEngine feature availability for each configuration, and Google verification requirements.

:::warning Important
You must add `gmail.modify` to **Disabled scopes** when using custom scopes. Otherwise, EmailEngine will request both `gmail.modify` AND your additional scopes, which defeats the purpose of using limited scopes.
:::

## Production Considerations

### Security Audit for Public Apps

If you need a public app for any Gmail user:

**Requirements:**

1. **Security audit**: OWASP compliance, penetration testing
2. **Use case validation**: Google may reject certain use cases
3. **Minimum scopes**: Google may require narrower scopes

**Process:**

- Expensive and time-consuming
- May require custom scope configuration
- Not all email use cases are approved

**Alternatives:**

- Use Internal apps (Google Workspace only)
- Use IMAP/SMTP with app passwords
- Consider if narrower scopes work for your use case

### Managing Pub/Sub Resources

EmailEngine automatically:

- Creates one Pub/Sub topic and subscription pair per Pub/Sub service account app, shared by all accounts that use it (created when the app is registered)
- Registers a Gmail watch (`users.watch`) for each account, pointing at the shared topic
- Renews each Gmail watch once it is more than a day old, checking hourly (watches expire after about 7 days)
- Recreates the Pub/Sub subscription automatically if Google deletes it due to inactivity (default TTL 31 days, configurable via `gmailSubscriptionTtl`)
- Deletes the topic and subscription it created when the service account app is deleted - not when individual accounts are removed. A topic or subscription that already existed under the configured name is adopted rather than created, and is left in place

You can:

- Monitor Pub/Sub usage in Google Cloud Console
- Set up billing alerts for Pub/Sub costs
- View subscription status in EmailEngine under **Integrations** > **OAuth2 Apps** > **Gmail Subscriptions**, or with [`GET /v1/pubsub/status`](/docs/api/get-v-1-pubsub-status)

See [Gmail Pub/Sub Integration](./gmail-pubsub) for the resource names, the TTL setting, and troubleshooting.

### Granular Consent and Scope Validation

Google supports **granular consent**, allowing users to selectively grant or deny individual permissions during the OAuth2 flow. EmailEngine validates that all required functional scopes were granted after the OAuth2 callback.

If a user deselects a required scope (e.g., unchecks email access), EmailEngine:

1. Detects the missing scope(s)
2. Revokes the partial token (best-effort) to prevent dangling grants
3. Shows an error page listing the missing permissions with human-readable descriptions
4. Offers a "Try Again" button to restart the OAuth2 flow

This prevents accounts from being registered with insufficient permissions, which would cause authentication errors during sync.

### Token Management

EmailEngine automatically:

- Refreshes access tokens when they expire during API requests
- Calls the Gmail API regularly even for idle accounts (the watch renewal and the fallback poll), which keeps the refresh token in use
- Stores refresh tokens in Redis, encrypted when [field encryption](/docs/advanced/encryption) is enabled
- Reports a failed refresh as an [`authenticationError`](/docs/webhooks/authenticationerror), since obtaining a new grant needs the user

An account whose refresh keeps failing for longer than [`EENGINE_MAX_IMAP_AUTH_FAILURE_TIME`](/docs/configuration/environment-variables#max-imap-auth-failure-time) (three days by default) is switched off and reports the `unset` state with a non-null `authFailureDisabledAt`. Re-authorizing it through the hosted authentication form brings it back (since v2.79.4); see [Accounts switched off after authentication failures](/docs/accounts/managing-accounts#accounts-switched-off-after-authentication-failures).

You can:

- Retrieve current access tokens for other Google API calls
- Monitor token status via account state
- Revoke access by deleting the account

:::warning Refresh Token Expiration
Google refresh tokens can expire under certain conditions:

- **6 months of inactivity** - If not used to obtain new access tokens
- **7 days** - If your OAuth app is in "Testing" mode (not published to production)
- **User revokes access** - Via Google account settings
- **Password change** - When Gmail scopes are present
- **Token limit exceeded** - Google allows ~50 refresh tokens per user/client; oldest tokens are invalidated

EmailEngine keeps tokens active by making regular API requests, but if an account is deleted from EmailEngine and re-added later, a new consent flow is required.
:::

[Learn more about OAuth2 token management >](../oauth2-token-management)

## See Also

- [Gmail over IMAP](/docs/accounts/gmail/gmail-imap) - The other Gmail backend, and how the two differ
- [Gmail API scopes](/docs/accounts/gmail/gmail-api-scopes) - Which features each scope combination supports
- [Gmail Pub/Sub](/docs/accounts/gmail/gmail-pubsub) - The push subscription this setup depends on
- [Google service accounts](/docs/accounts/gmail/google-service-accounts) - Workspace access without per-user consent
- [Account troubleshooting](/docs/accounts/troubleshooting) - When consent or sync fails
