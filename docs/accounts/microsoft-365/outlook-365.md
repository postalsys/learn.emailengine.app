---
title: Setting Up Outlook and Microsoft 365 (Delegated Access)
sidebar_label: Delegated Access (OAuth2)
sidebar_position: 1
description: Complete guide to setting up Outlook.com and Microsoft 365 accounts with OAuth2 authentication
---

# Setting Up Outlook and Microsoft 365 (Delegated Access)

This guide shows you how to set up Outlook OAuth2 authentication with EmailEngine using **delegated access**, where each user signs in interactively through Microsoft's OAuth2 consent flow. This works with Outlook.com, Hotmail.com, and Microsoft 365 accounts. You can use either IMAP/SMTP or Microsoft Graph API as the email backend.

:::tip Looking for Application Access?
If you need to access Microsoft 365 mailboxes **without interactive user login** (e.g., for automated systems, helpdesk integrations, or centralized management), see [Outlook Application Access (Client Credentials)](./outlook-client-credentials) instead. Application access lets an admin grant access to any mailbox in the organization using a single set of credentials.
:::

## Overview

EmailEngine supports two types of Outlook OAuth2 integrations:

- **Delegated access** (this page) - Each user signs in and grants access to their own mailbox. Supports both IMAP/SMTP and MS Graph API backends. Works with personal Microsoft accounts and Microsoft 365.
- **Application access** ([separate guide](./outlook-client-credentials)) - An admin grants the application access to any mailbox in the organization. Uses MS Graph API only. Microsoft 365 only, no personal accounts.

### Email Backend Options

With delegated access, you can choose between two backend options:

**IMAP/SMTP Backend:**

- Standard email protocols
- Works like any other email account
- Simpler setup

**Microsoft Graph API Backend:**

- Native Microsoft 365 integration
- Better performance for high-volume accounts
- Access to Microsoft-specific features
- Supports shared mailboxes natively

This guide covers both options.

## Choosing IMAP/SMTP vs MS Graph API

| Feature                         | IMAP/SMTP        | MS Graph API           |
| ------------------------------- | ---------------- | ---------------------- |
| **Setup Complexity**            | Simple           | Moderate               |
| **Performance**                 | Good             | Excellent              |
| **Search Capabilities**         | Full text search | Very limited           |
| **Shared Mailboxes**            | Limited support  | Native support         |
| **Outlook Categories**          | Not available    | Supported via `labels` |
| **Connection Protocol**         | IMAP/SMTP        | REST API               |
| **Microsoft-Specific Features** | Limited          | Full access            |
| **Works with other providers**  | Yes              | No                     |

:::warning MS Graph API Search Limitations
MS Graph API has **significantly more limited search capabilities** compared to IMAP. IMAP supports full-text search across message headers and body content, while MS Graph API search is much more restricted. If your application requires advanced message search functionality, use IMAP/SMTP instead.
:::

**Recommendation:**

- Use **IMAP/SMTP** for simple setups, for compatibility, and when you need IMAP's search
- Use **MS Graph API** for shared mailboxes and Microsoft 365 enterprise features (but be aware of search limitations)

## Step 1: Create Azure AD Application

Go to [Azure Portal](https://portal.azure.com/) and navigate to **Microsoft Entra ID** → **App Registrations**.

![Navigate to Azure AD App Registrations](/img/outlook/out001.gif)

Click **New registration**.

![Click New registration button](/img/outlook/out002.gif)

## Step 2: Configure Application Registration

![Application registration form](/img/outlook/out003.gif)

### Application Name

Choose a name that users will see in the authorization form. Make it clear and recognizable (e.g., "EmailEngine" or "YourApp Email Integration").

### Supported Account Types

Choose who can use this application:

**Personal Microsoft accounts only:**

- Only free @hotmail.com, @outlook.com, @live.com accounts
- Select: "Accounts in any organizational directory and personal Microsoft accounts"

**Single tenant (organization only):**

- Only accounts from your Microsoft 365 organization
- Select: "Accounts in this organizational directory only"

**Multi-tenant (any organization):**

- Any Microsoft 365 organization account
- Does NOT include personal Microsoft accounts
- Select: "Accounts in any organizational directory"

**Multi-tenant + personal accounts:**

- Both Microsoft 365 and personal accounts
- Most flexible option
- Select: "Accounts in any organizational directory and personal Microsoft accounts"

:::tip Recommended Setting
For maximum compatibility, select **"Accounts in any organizational directory and personal Microsoft accounts"** unless you have specific requirements.
:::

### Microsoft Cloud Environments

EmailEngine supports multiple Microsoft cloud environments for government and regional deployments. Select the appropriate cloud when creating an Outlook OAuth2 application to use the correct endpoints.

| Cloud | Value | Use Case |
|-------|-------|----------|
| **Azure Global** | `global` | Standard Microsoft 365 (default) |
| **GCC High** | `gcc-high` | US Government L4 |
| **DoD** | `dod` | US Department of Defense L5 |
| **Azure China** | `china` | China (operated by 21Vianet) |

#### Cloud Environment Details

**Azure Global (default)**

The standard commercial Microsoft 365 environment used by most organizations:

- **Entra ID Endpoint**: `https://login.microsoftonline.com`
- **MS Graph API**: `https://graph.microsoft.com`
- **IMAP Host**: `outlook.office365.com`
- **SMTP Host**: `smtp.office365.com`
- **Azure Portal**: [portal.azure.com](https://portal.azure.com)

**GCC High (US Government L4)**

For US government agencies and contractors requiring FedRAMP High + DoD SRG Impact Level 4 compliance:

- **Entra ID Endpoint**: `https://login.microsoftonline.us`
- **MS Graph API**: `https://graph.microsoft.us`
- **IMAP Host**: `outlook.office365.us`
- **SMTP Host**: `smtp.office365.us`
- **Azure Portal**: [portal.azure.us](https://portal.azure.us)

**DoD (US Department of Defense L5)**

For US Department of Defense requiring Impact Level 5 (ITAR and DoD SRG):

- **Entra ID Endpoint**: `https://login.microsoftonline.us`
- **MS Graph API**: `https://dod-graph.microsoft.us`
- **IMAP Host**: `outlook-dod.office365.us`
- **SMTP Host**: `outlook-dod.office365.us`
- **Azure Portal**: [portal.azure.us](https://portal.azure.us)

**Azure China (21Vianet)**

Microsoft Azure operated by 21Vianet in China for compliance with Chinese regulations:

- **Entra ID Endpoint**: `https://login.chinacloudapi.cn`
- **MS Graph API**: `https://microsoftgraph.chinacloudapi.cn`
- **IMAP Host**: `partner.outlook.cn`
- **SMTP Host**: `partner.outlook.cn`
- **Azure Portal**: [portal.azure.cn](https://portal.azure.cn)

#### Configuring Cloud Environment via API

When creating an Outlook OAuth2 application via API, specify the `cloud` parameter:

```bash
curl -X POST "https://emailengine.example.com/v1/oauth2" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Outlook GCC High",
    "provider": "outlook",
    "enabled": true,
    "cloud": "gcc-high",
    "clientId": "YOUR_CLIENT_ID",
    "clientSecret": "YOUR_CLIENT_SECRET",
    "redirectUrl": "https://emailengine.example.com/oauth",
    "authority": "organizations"
  }'
```

Valid values for `cloud`:

- `global` (default if not specified)
- `gcc-high`
- `dod`
- `china`

#### OAuth2 Scopes by Cloud

Each cloud environment uses different scope URLs. EmailEngine automatically uses the correct scopes based on the selected cloud:

**IMAP/SMTP Scopes:**

| Cloud | IMAP Scope | SMTP Scope |
|-------|------------|------------|
| Global | `https://outlook.office.com/IMAP.AccessAsUser.All` | `https://outlook.office.com/SMTP.Send` |
| GCC High | `https://outlook.office365.us/IMAP.AccessAsUser.All` | `https://outlook.office365.us/SMTP.Send` |
| DoD | `https://outlook.office365.us/IMAP.AccessAsUser.All` | `https://outlook.office365.us/SMTP.Send` |
| China | `https://partner.outlook.cn/IMAP.AccessAsUser.All` | `https://partner.outlook.cn/SMTP.Send` |

**MS Graph API Scopes:**

| Cloud | Mail.ReadWrite | Mail.Send |
|-------|----------------|-----------|
| Global | `https://graph.microsoft.com/Mail.ReadWrite` | `https://graph.microsoft.com/Mail.Send` |
| GCC High | `https://graph.microsoft.us/Mail.ReadWrite` | `https://graph.microsoft.us/Mail.Send` |
| DoD | `https://dod-graph.microsoft.us/Mail.ReadWrite` | `https://dod-graph.microsoft.us/Mail.Send` |
| China | `https://microsoftgraph.chinacloudapi.cn/Mail.ReadWrite` | `https://microsoftgraph.chinacloudapi.cn/Mail.Send` |

:::warning Government Cloud Registration
For government clouds (GCC High, DoD) and sovereign clouds (China), you must:

1. Register your Azure AD application in the corresponding Azure portal for that cloud environment
2. Your organization must have a subscription in that cloud environment
3. Use the correct Azure portal URL for app registration

You cannot use an app registered in the global Azure portal for government or China cloud environments.
:::

### Redirect URI

**Platform:** Select **Web**

**URI:** Your EmailEngine URL with `/oauth` path:

- `http://localhost:3000/oauth` (for local testing)
- `https://emailengine.example.com/oauth` (for production)

:::warning Redirect URL Must Match Exactly
The redirect URL must match exactly what you'll configure in EmailEngine later. Mismatches will cause OAuth failures.
:::

Click **Register** to create the application.

## Step 3: Copy Application (Client) ID

![Application overview page with client ID](/img/outlook/out004.gif)

On the application overview page, find and copy the **Application (client) ID**. You'll need this for EmailEngine configuration.

Keep this page open - you'll come back to it.

## Step 4: Configure API Permissions

Click **API permissions** in the left menu.

![API permissions page](/img/outlook/out005.gif)

By default, only `User.Read` permission exists. Click **Add a permission**.

![Adding permission button](/img/outlook/out006.gif)

Select **Microsoft Graph** and then **Delegated permissions**.

:::danger Select Delegated Permissions, Not Application Permissions
Make sure you select the **Delegated permissions** tab, not **Application permissions**. Application permissions are used for [application access (client credentials)](./outlook-client-credentials), which is a different setup flow. Delegated permissions allow the app to act on behalf of a signed-in user.
:::

### For IMAP/SMTP Backend

Add these permissions:

![Searching for and selecting IMAP permissions](/img/outlook/out007.gif)

- `IMAP.AccessAsUser.All` - For IMAP access
- `SMTP.Send` - For sending emails via SMTP
- `offline_access` - For token refresh

:::important offline_access Required
The `offline_access` scope is required in both cases. It allows EmailEngine to renew access tokens in the background without user interaction.
:::

### For MS Graph API Backend

Add these permissions instead:

- `Mail.ReadWrite` - For reading and managing emails
- `Mail.Send` - For sending emails
- `offline_access` - For token refresh

:::warning Don't Mix Scopes
You cannot use both IMAP/SMTP and Mail.\* scopes together. Choose one backend:

- **IMAP/SMTP scopes**: From Outlook API
- **Mail.\* scopes**: From MS Graph API

Pick one approach and stick with it.
:::


Verify all required permissions are listed, then continue to the next step.

## Step 5: Create Client Secret

Click **Certificates & secrets** in the left menu.

![Certificates & secrets page](/img/outlook/out008.gif)

Click **New client secret**.

![New client secret](/img/outlook/out009.gif)

### Configure Secret

**Description:** Give it a meaningful name (e.g., "EmailEngine Secret")

**Expires:** Choose an expiration period you're comfortable with

- 6 months (default)
- 12 months
- 24 months
- Custom

:::tip Secret Expiration
When the secret expires, your application will stop working until you generate a new secret and update EmailEngine. Choose a longer expiration for production, but set a reminder to rotate it before expiry.
:::

Click **Add**.

### Copy Secret Value

:::danger Copy Secret Now
The secret value is only shown once. Copy it immediately - you cannot retrieve it later. If you lose it, you'll need to generate a new secret.
:::

Copy the value from the **Value** column (NOT the "Secret ID").

## Step 6: Enable IMAP/SMTP (If Using IMAP Backend)

:::info Only for IMAP/SMTP Backend
Skip this step if you're using MS Graph API as your backend.
:::

If you're managing a Microsoft 365 organization and EmailEngine cannot connect via IMAP/SMTP, you may need to enable these protocols manually.

![Enabling IMAP and SMTP in Microsoft 365 admin center](/img/outlook/enable-imap-smtp.gif)

1. Navigate to [https://admin.microsoft.com/](https://admin.microsoft.com/)
2. Go to **Users** → **Active users**
3. Select a user
4. Navigate to **Mail** settings
5. Enable **IMAP** and **SMTP AUTH**

You may need to do this for each user or configure organization-wide settings.

## Step 7: Configure EmailEngine

Now configure EmailEngine with your Azure application credentials.

### Add Outlook OAuth2 Application

![Creating Outlook OAuth2 application in EmailEngine](/img/outlook/out010.gif)

1. Open EmailEngine dashboard
2. Navigate to **Integrations** > **OAuth2 Apps**
3. Click **Create OAuth2 app**
4. Select **Outlook (delegated)** from the dropdown

### Configure OAuth2 Settings

![Outlook OAuth2 configuration form](/img/outlook/out011.gif)

**Application name:** Give it a descriptive name (e.g., "Outlook OAuth2")

**Enable this app:** Check this box (otherwise it won't appear in authentication forms)

**Azure Application Id:** Paste the Application (client) ID from Azure

**Client Secret:** Paste the secret value you copied earlier

**Redirect URL:** Must match exactly what you entered in Azure:

- Example: `http://localhost:3000/oauth`

**Supported account types:** Choose based on your Azure configuration:

- `common` - Organizations and personal accounts (most flexible)
- `consumers` - Personal Microsoft accounts only (@hotmail.com, @outlook.com, @live.com)
- `organizations` - Microsoft 365 organizations only
- `<directory-id>` - Specific organization only (use Directory/Tenant ID like `f8cdef31-a31e-4b4a-93e4-5f571e91255a`)

:::tip Account Type Mapping
| Azure Setting | EmailEngine Value |
|---------------|-------------------|
| Any org + personal | `common` |
| Personal accounts only | `consumers` |
| Any organization | `organizations` |
| Single organization | Use Directory ID |
:::

**Base scope:** Select based on your Azure permissions:

- **IMAP and SMTP** - If you added `IMAP.AccessAsUser.All` and `SMTP.Send`
- **MS Graph API** - If you added `Mail.ReadWrite` and `Mail.Send`

:::important Base Scope Must Match Azure Permissions
The base scope you select here must match the permissions you configured in Azure. Mismatches will cause authentication failures.
:::

Click **Register app** to save.

## Step 8: Test the Setup

Add an Outlook account to test the OAuth2 flow.

### Via Hosted Authentication Form

![Using hosted authentication form](/img/outlook/out012.gif)

1. In EmailEngine, click **Add account**
2. Click **Sign in with Microsoft**
3. Complete the OAuth2 consent flow
4. EmailEngine will store the credentials and connect

The account should enter "connected" state within moments.

### Via API

Generate an authentication form URL:

```bash
curl -X POST https://emailengine.example.com/v1/authentication/form \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user123",
    "email": "user@outlook.com",
    "redirectUrl": "https://myapp.com/settings"
  }'
```

Direct the user to the returned URL.

[Learn more about hosted authentication →](/docs/accounts/hosted-authentication)

### Direct API Account Registration

If you already have OAuth2 tokens:

```bash
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user123",
    "name": "John Doe",
    "email": "user@outlook.com",
    "oauth2": {
      "provider": "AAABlf_0iLgAAAAQ",
      "accessToken": "EwBIA8l6...",
      "refreshToken": "M.R3_BAY..."
    }
  }'
```

:::info Provider ID
The `provider` field should be the **OAuth2 application ID** from EmailEngine, which is a base64url encoded string like `AAABlf_0iLgAAAAQ`. You can find this ID in **Integrations → OAuth2 Apps** in the EmailEngine interface. This is NOT the Application (client) ID from Azure AD.
:::

[See full API documentation →](/docs/api/post-v-1-account)

## Shared Mailboxes

Microsoft 365 shared mailboxes are mailboxes not bound to a specific user. Multiple users can access them using their own credentials.

:::tip Recommended: Application Access
For shared mailboxes, [Outlook Application Access (Client Credentials)](./outlook-client-credentials) is the simplest approach - no interactive login, no extra scopes, just an API call per mailbox.
:::

With delegated access, EmailEngine also supports shared mailboxes through:

- **Delegated access** - Add a main account, then add shared mailboxes that reference its credentials
- **Direct access** - Add a shared mailbox with its own OAuth2 authentication

For detailed setup instructions covering all three approaches, see the [Shared Mailboxes guide](./shared-mailboxes).

## Performance Considerations

### IMAP/SMTP Limits

Microsoft enforces connection and rate limits:

**Connection Limits:**

- 15 concurrent IMAP connections per account
- 30 SMTP connections per minute

**Rate Limits:**

- Throttling on excessive operations
- May temporarily block account

**Best Practices:**

- Use sub-connections sparingly
- Implement path filtering
- Monitor account states for throttling

### MS Graph API Limits

Microsoft Graph has separate quotas:

**Throttling Limits:**

- Varies by license type and tenant
- Typically higher than IMAP limits

**Best Practices:**

- Implement exponential backoff
- Use batch requests where possible
- Monitor for throttling responses

[Learn more about performance tuning →](/docs/advanced/performance-tuning)

## Production Considerations

### Admin Consent for Organization Apps

For single-tenant or organization apps, an admin can grant consent for all users:

1. In Azure AD, go to your app registration
2. Click **API permissions**
3. Click **Grant admin consent for [Organization]**

This pre-approves the app for all users in the organization.

### Token Management

EmailEngine automatically:

- Refreshes access tokens when they expire during API requests
- Makes regular API requests (at least daily) even for idle accounts, keeping refresh tokens active
- Stores refresh tokens securely
- Re-authenticates on token failures

You can:

- Retrieve current access tokens for other Microsoft API calls
- Monitor token status via account state
- Revoke access by deleting the account

:::warning Refresh Token Expiration
Microsoft refresh tokens can expire under certain conditions:

- **90 days of inactivity** - If not used to obtain new access tokens
- **Maximum ~1 year** - Rolling lifetime limit even if used regularly (may vary by tenant configuration)
- **User revokes consent** - Via https://myapps.microsoft.com
- **Admin revokes app** - Via Azure AD Enterprise Applications
- **Password change** - Invalidates existing refresh tokens
- **Policy changes** - New MFA or conditional access policies invalidate old tokens
- **"Sign out everywhere"** - User action that kills all refresh tokens

EmailEngine keeps tokens active by making regular API requests, but if an account is deleted from EmailEngine and re-added later, a new consent flow is required.
:::

:::danger Client Secret Expiration
Microsoft OAuth2 **client secrets expire** (90 days to 2 years max). When expired, **all accounts** using that OAuth2 app fail immediately. Monitor expiration dates in Azure AD and rotate secrets before they expire.
:::

[Learn more about OAuth2 token management →](../oauth2-token-management)
