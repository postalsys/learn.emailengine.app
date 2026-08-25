---
title: Setting Up Gmail API
sidebar_label: API Setup
sidebar_position: 2
description: Configure EmailEngine to use Gmail REST API as the email backend with Cloud Pub/Sub webhooks
---

# Setting Up Gmail API

This guide shows you how to configure EmailEngine to use Gmail REST API as the email backend instead of IMAP/SMTP. With this setup, EmailEngine uses direct Gmail API calls for all operations and receives change notifications via Google Cloud Pub/Sub.

:::info IMAP vs Gmail API
In a [previous guide](./gmail-imap), we showed how to configure EmailEngine to access Gmail over OAuth2 using IMAP and SMTP. That setup uses Gmail OAuth2 **only for generating access tokens** to authenticate IMAP/SMTP sessions.

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

Click the hamburger menu (top-left) → **APIs & Services** → **Enabled APIs & Services**.

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

Click hamburger menu → **APIs & Services** → **OAuth consent screen**.

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

Click **Add or remove scopes** and find `gmail.modify` from the list.

![Adding required scope](/img/external/BONjtoR9p6.gif)


Check `gmail.modify` and click **Update**.

:::important Required Scope
The `gmail.modify` scope is required for Gmail API access. This is different from the `https://mail.google.com/` scope used for IMAP/SMTP.

If Google's verification process determines you need different scopes (e.g., `gmail.readonly`, `gmail.send`, `gmail.labels`), see the [Custom Scopes section](#using-custom-scopes) below.
:::

![Saving consent screen configuration](/img/external/THYy7q5W6Z.gif)


Scroll down and click **Save and continue** to finish consent screen setup.

## Step 4: Create OAuth Credentials for User Authentication

Navigate to **APIs & Services** → **Credentials**.

![Navigating to credentials page](/img/external/7bDFveWih1.gif)


Click **Create credentials** → **OAuth client ID**.

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

Keep these separate - you'll need both!
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


Click **Add key** → **Create new key** → **JSON format**.

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

**Credentials file:** Select the **service account** JSON file (not the user OAuth credentials!)

**Base scope:** Select **Cloud Pub/Sub**

:::warning Use Correct Credentials File
Make sure you're uploading the **service account** credentials file, not the user OAuth credentials file. You can tell them apart:

- **Service account**: Filename uses service account name prefix
- **User OAuth**: Filename starts with `client_secret_`
  :::

Click **Register app** to save.

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

**Credentials file:** Select the **user OAuth credentials** file (`client_secret_...json`)

**Redirect URL:** Verify this matches exactly what you entered in Google Cloud Console

**Base scope:** Select **Gmail API**

**Service Account for managing webhook Pub/Sub:** Select the service account app you created earlier

:::important Link Service Account
This selector is marked as optional in the UI, but you should select the service account application you created in Step 6. It tells EmailEngine which credentials to use for managing Pub/Sub resources - without it, accounts using this app will not receive real-time notifications or webhooks.
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
- In Google Cloud Console → Pub/Sub, you should see:
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
- Not all email use cases will be approved

**Alternatives:**

- Use Internal apps (Google Workspace only)
- Use IMAP/SMTP with app passwords
- Consider if narrower scopes work for your use case

### Managing Pub/Sub Resources

EmailEngine automatically:

- Creates one Pub/Sub topic and subscription pair per Pub/Sub service account app, shared by all accounts that use it (created when the app is registered)
- Registers a Gmail watch (`users.watch`) for each account, pointing at the shared topic
- Renews each Gmail watch at least daily (watches expire after about 7 days)
- Recreates the Pub/Sub subscription automatically if Google deletes it due to inactivity (default TTL 31 days, configurable via `gmailSubscriptionTtl`)
- Deletes the topic and subscription when the OAuth2 app is deleted - not when individual accounts are removed

You can:

- Monitor Pub/Sub usage in Google Cloud Console
- Set up billing alerts for Pub/Sub costs
- View subscription status in EmailEngine

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
- Makes regular API requests (at least daily) even for idle accounts, keeping refresh tokens active
- Stores refresh tokens securely
- Re-authenticates on token failures

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

[Learn more about OAuth2 token management →](../oauth2-token-management)
