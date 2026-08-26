---
title: Outlook Application Access (Client Credentials)
sidebar_label: Application Access (Client Credentials)
sidebar_position: 2
description: Setting up Microsoft 365 application-level access using client credentials for EmailEngine
---

# Outlook Application Access (Client Credentials)

This guide shows you how to set up application-level access to Microsoft 365 mailboxes using the OAuth2 client credentials flow. This allows EmailEngine to access any mailbox in your organization without interactive user logins. The `outlookService` provider that implements it is available since EmailEngine v2.65.0.

## Overview

### What is Application Access?

Application access uses the OAuth2 **client credentials** grant to authenticate as the application itself, rather than on behalf of a signed-in user. The application receives its own identity and permissions, and can access any mailbox that the admin has authorized.

### Application Access vs Delegated Access

EmailEngine supports two types of Outlook OAuth2 integrations:

| Feature | Application Access | Delegated Access |
|---|---|---|
| **Authentication** | App authenticates as itself (client credentials) | User signs in interactively |
| **User Interaction** | None required | Each user must complete OAuth2 login |
| **Permissions** | Application permissions with admin consent | Delegated permissions with user consent |
| **Mailbox Access** | Any mailbox in the organization | Only the signed-in user's mailbox |
| **Email Backend** | MS Graph API only | IMAP/SMTP or MS Graph API |
| **Account Setup** | REST API only | Hosted auth form or REST API |
| **Tenant Requirement** | Specific tenant ID required | Can use "common", "organizations", or tenant ID |
| **Best For** | Automated systems, shared mailboxes, compliance | User-facing apps, individual accounts |

For delegated access setup, see [Setting Up Outlook and Microsoft 365](./outlook-365).

### When to Use Application Access

**Best For:**

- Enterprise Microsoft 365 deployments with centralized management
- Accessing shared mailboxes or multiple user mailboxes
- Automated workflows without user interaction (helpdesk, compliance, archiving)
- Service integrations where interactive login is not possible
- Systems that need to monitor or process email across an organization

**Not Suitable For:**

- Personal Microsoft accounts (@hotmail.com, @outlook.com, @live.com)
- Applications where users should individually control access
- Scenarios requiring IMAP/SMTP protocols (application access uses MS Graph API only)

## Benefits and Limitations

### Benefits

**No User Consent Required:**

- Access granted by admin once for the entire organization
- No OAuth2 login flow for each user
- No interactive login for any mailbox in the tenant

**Centralized Management:**

- Admin controls all access from Azure portal
- Easy to revoke for the entire organization
- Audit trail via Azure AD sign-in logs

**Scalability:**

- Add new mailboxes without re-authentication
- Access any user's mailbox with the same app credentials
- Ideal for enterprise deployments

### Limitations

**Microsoft 365 Only:**

- Does not work with personal Microsoft accounts
- Requires a paid Microsoft 365 subscription

**Requires Admin Consent:**

- Must have Azure AD admin privileges
- Cannot be delegated to non-admin users

**MS Graph API Only:**

- Application access uses the MS Graph API backend exclusively
- IMAP/SMTP protocols are not available with client credentials

**Organization-Scoped:**

- Only works for mailboxes in your Microsoft 365 organization
- Cannot access external accounts

**Client Secret Expiration:**

- The Azure portal issues client secrets with a lifetime of up to 24 months
- Must be rotated before expiration to avoid service disruption

## Step 1: Create Azure AD Application

Go to [Azure Portal](https://portal.azure.com/) and navigate to **Microsoft Entra ID** > **App Registrations**.

Click **New registration**.

### Application Name

Choose a descriptive name (e.g., "EmailEngine Application Access").

### Supported Account Types

Select **"Accounts in this organizational directory only"** (single tenant). Application access requires a specific tenant, so multi-tenant configurations are not applicable.

### Redirect URI

Leave the redirect URI **empty**. The client credentials flow does not use redirects.

Click **Register** to create the application.

### Copy Application (Client) ID

On the application overview page, copy the **Application (client) ID**. You will need this when configuring EmailEngine.

### Copy Directory (Tenant) ID

Also copy the **Directory (tenant) ID** from the overview page. Unlike delegated access which can use "common" or "organizations", application access requires the specific tenant ID.

:::warning Tenant ID Required
Application access always requires a specific tenant ID. You cannot use "common", "organizations", or "consumers" as the authority value. The tenant ID is a UUID like `f8cdef31-a31e-4b4a-93e4-5f571e91255a`.
:::

## Step 2: Configure Application Permissions

Click **API permissions** in the left menu.

By default, only the `User.Read` delegated permission exists. Remove it and add application permissions instead.

Click **Add a permission** > **Microsoft Graph** > **Application permissions**.

Add the following permissions:

- `Mail.ReadWrite` - Read and write mail in all mailboxes
- `Mail.Send` - Send mail as any user
- `User.Read.All` - Read all users' profiles (needed for mailbox resolution)

:::danger Application Permissions, Not Delegated
Make sure you select **Application permissions**, not **Delegated permissions**. These are in separate tabs when adding permissions. Application permissions apply to the app itself and require admin consent. Delegated permissions apply on behalf of a signed-in user and are used in the delegated access flow.
:::

## Step 3: Grant Admin Consent

After adding the permissions, click **Grant admin consent for [Your Organization]**.

This step is **mandatory** for application permissions. Unlike delegated permissions where users can consent individually, application permissions must be explicitly approved by an Azure AD administrator.

Verify that all three permissions show a green checkmark under the "Status" column, indicating admin consent has been granted.

## Step 4: Create Client Secret

Click **Certificates & secrets** in the left menu.

Click **New client secret**.

**Description:** Give it a meaningful name (e.g., "EmailEngine Secret")

**Expires:** Choose an expiration period. The Azure portal offers up to 24 months.

:::danger Copy Secret Immediately
The secret value is only shown once. Copy it immediately after creation - you cannot retrieve it later. If you lose it, you must generate a new secret.
:::

Copy the value from the **Value** column (not the "Secret ID").

:::tip Secret Expiration Reminder
Set a calendar reminder to rotate the client secret before it expires. When the secret expires, **all accounts** using this OAuth2 application will fail to authenticate until a new secret is configured in EmailEngine.
:::

## Step 5: Configure EmailEngine

Now configure EmailEngine to use the Azure application for mailbox access.

### Add Outlook (application) OAuth2 Application

1. Open the EmailEngine dashboard
2. Navigate to **Integrations** > **OAuth2 Apps**
3. Click **Create OAuth2 app**
4. Select **Outlook (application)**

### Configure OAuth2 Settings

**Application name:** Give it a descriptive name (e.g., "Outlook Application Access")

**Azure Application Id:** Paste the Application (client) ID from Azure

**Client Secret:** Paste the secret value you copied earlier

**Azure cloud environment:** Select your Microsoft cloud environment (default: **Azure Global**)

**Directory (tenant) ID:** Paste the Directory (tenant) ID from Azure. This must be the specific tenant UUID; the API field is `authority`.

:::info No Redirect URL
Unlike delegated access, application access does not require a redirect URL. The client credentials flow authenticates directly with Azure without browser redirects.
:::

Click **Register app** to save.

### Find Your App ID

After registering, note the **App ID** displayed in the OAuth2 application list. This is a base64url-encoded identifier (e.g., `AAABkTn2CRQAAAAB`) that you need when adding accounts via the API.

You can also find it via the API:

```bash
curl https://emailengine.example.com/v1/oauth2 \
  -H "Authorization: Bearer YOUR_EMAILENGINE_TOKEN"
```

### Verify the Setup

Before adding accounts, use the **Verify setup** button on the OAuth2 app's page (or [`POST /v1/oauth2/{app}/verify`](/docs/api/post-v-1-oauth-2-app-verify), both since v2.68.0) to test the configuration. The verifier obtains a client-credentials token and - if you provide a test mailbox address - reads that user's `id`, `mail`, and `userPrincipalName` from Microsoft Graph, which is what `User.Read.All` is granted for. Each step is reported as passed, failed, or skipped with a hint for fixing failures.

## Step 6: Add Email Accounts

The hosted authentication form is not available for application access because there is no interactive user login. Instead, add accounts either through the REST API or directly from the admin dashboard.

### Add Account via the Admin UI

The OAuth2 app's detail page in the EmailEngine dashboard provides an **Add account** button (since v2.68.0) that opens a dialog asking for the full name, the email address, and an optional account identifier, and registers the account directly - no API call needed. Leave the identifier blank to have one generated.

### Add Account via API

Add accounts using the [Register Account API endpoint](/docs/api/post-v-1-account):

```bash
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_EMAILENGINE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user123",
    "name": "John Doe",
    "email": "john@contoso.com",
    "oauth2": {
      "provider": "AAABkTn2CRQAAAAB",
      "auth": {
        "user": "john@contoso.com"
      }
    }
  }'
```

Replace `AAABkTn2CRQAAAAB` with your actual App ID from EmailEngine.

**Key points:**

- No `accessToken` or `refreshToken` needed - EmailEngine obtains tokens automatically via client credentials
- The `auth.user` field specifies which mailbox to access
- The `email` field should match the mailbox email address
- EmailEngine handles token acquisition and renewal automatically

**Response:**

```json
{
  "account": "user123",
  "state": "new"
}
```

The account should transition to the "connected" state within moments.

### Add Multiple Accounts

You can add any mailbox in your organization using the same App ID:

```bash
# Add multiple accounts (replace APP_ID with your actual App ID)
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "sales-1",
    "email": "alice@contoso.com",
    "oauth2": {
      "provider": "AAABkTn2CRQAAAAB",
      "auth": { "user": "alice@contoso.com" }
    }
  }'

curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "sales-2",
    "email": "bob@contoso.com",
    "oauth2": {
      "provider": "AAABkTn2CRQAAAAB",
      "auth": { "user": "bob@contoso.com" }
    }
  }'
```

### Verify Accounts

Check the account status:

```bash
curl https://emailengine.example.com/v1/account/user123 \
  -H "Authorization: Bearer YOUR_TOKEN"
```

The account should show `"state": "connected"` when successfully linked.

## Shared Mailboxes

Application access suits shared mailboxes: use the shared mailbox address as the `auth.user` value.

```bash
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "support-inbox",
    "email": "support@contoso.com",
    "oauth2": {
      "provider": "AAABkTn2CRQAAAAB",
      "auth": { "user": "support@contoso.com" }
    }
  }'
```

No additional delegation configuration is needed - the application permissions grant direct access to all mailboxes.

## Cloud Environments

EmailEngine supports multiple Microsoft cloud environments. Select the appropriate cloud in the **Azure cloud environment** field when creating the OAuth2 application to use the correct authentication and API endpoints. Client credentials request the single scope `{graph-endpoint}/.default`, so the cloud decides which Graph endpoint that is.

| Cloud | Value | Use Case |
|---|---|---|
| **Azure Global** | `global` | Standard Microsoft 365 (default) |
| **GCC High** | `gcc-high` | US Government L4 |
| **DoD** | `dod` | US Department of Defense L5 |
| **Azure China** | `china` | China (operated by 21Vianet) |

The Entra ID and Microsoft Graph endpoints behind each value are listed under [Microsoft cloud environments](./outlook-365#microsoft-cloud-environments) on the delegated access page.

### Configuring Cloud Environment via API

When creating the OAuth2 application via the API, specify the `cloud` parameter:

```bash
curl -X POST "https://emailengine.example.com/v1/oauth2" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Outlook Application Access (GCC High)",
    "provider": "outlookService",
    "enabled": true,
    "cloud": "gcc-high",
    "clientId": "YOUR_CLIENT_ID",
    "clientSecret": "YOUR_CLIENT_SECRET",
    "authority": "YOUR_TENANT_ID"
  }'
```

`redirectUrl` is not needed, and `baseScopes`, `extraScopes`, and `skipScopes` are ignored for this provider.

:::warning Government Cloud Registration
For government clouds (GCC High, DoD) and sovereign clouds (China), you must:

1. Register your Azure AD application in the corresponding Azure portal for that cloud environment
2. Your organization must have a subscription in that cloud environment
3. Use the correct Azure portal URL for app registration

You cannot use an app registered in the global Azure portal for government or China cloud environments.
:::

## Account Management

### Listing Accounts

Application access accounts appear like any other OAuth2 account:

```bash
curl https://emailengine.example.com/v1/accounts \
  -H "Authorization: Bearer YOUR_TOKEN"
```

### Updating Accounts

Update account settings using the [Update Account API endpoint](/docs/api/put-v-1-account-account):

```bash
curl -X PUT https://emailengine.example.com/v1/account/user123 \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "John Doe (Updated)"
  }'
```

### Deleting Accounts

Delete accounts using the [Delete Account API endpoint](/docs/api/delete-v-1-account-account):

```bash
curl -X DELETE https://emailengine.example.com/v1/account/user123 \
  -H "Authorization: Bearer YOUR_TOKEN"
```

## Security Best Practices

### Client Secret Management

**Do:**

- Store client secrets in secret management systems (Azure Key Vault, AWS Secrets Manager, HashiCorp Vault)
- Set calendar reminders for secret rotation before expiration
- Use separate Azure AD applications for different environments (dev/staging/prod)
- Monitor Azure AD sign-in logs for unusual activity

**Don't:**

- Commit secrets to version control
- Share secrets via email or chat
- Use the same secret across multiple environments
- Ignore expiration warnings

### Permission Scope

**Principle of Least Privilege:**

- `Mail.ReadWrite`, `Mail.Send`, and `User.Read.All` are the minimum required permissions
- Review permissions periodically and remove any that are no longer needed
- Consider using separate applications for read-only and read-write access patterns
- Document why each permission is needed for compliance audits

### Monitoring

- Enable Azure AD audit logs for the application
- Monitor EmailEngine account states for authentication errors
- Set up alerts for client secret expiration (Azure AD notifies admins)
- Review the list of accessed mailboxes regularly

:::warning Enable Field Encryption
By default, EmailEngine stores credentials in cleartext in Redis. To protect sensitive data like client secrets, enable field encryption. See [Setting Up Encryption](/docs/advanced/encryption) for configuration instructions.
:::

## Webhooks for Real-Time Updates

Application access uses MS Graph webhook subscriptions for real-time email notifications. EmailEngine requires two publicly reachable HTTPS endpoints:

- `{serviceUrl}/oauth/msg/notification` - Receives change notifications for messages
- `{serviceUrl}/oauth/msg/lifecycle` - Receives lifecycle events (`reauthorizationRequired`, `subscriptionRemoved`, `missed`)

EmailEngine automatically creates these subscriptions and renews them on a timer. Microsoft validates the notification URL when the subscription is created - if the endpoints are not reachable from Microsoft's servers, the subscription cannot be created. EmailEngine retries subscription creation periodically, but new-message detection will not work until the endpoints become reachable. There is no polling fallback for MS Graph accounts.

### Automatic Recovery for Missed Notifications

Microsoft Graph may occasionally fail to deliver change notifications - for example, due to transient network issues or service disruptions. When this happens, Microsoft sends a `missed` lifecycle event to inform EmailEngine that notifications were lost.

EmailEngine handles this automatically (since v2.67.0):

1. When a `missed` lifecycle event is received, EmailEngine lists messages received since two minutes before the last notification it processed, or over the last 30 minutes if it has not processed any
2. Any messages not already processed through normal notifications are synced
3. A five-minute cooldown per account prevents repeated recovery runs for the same event. **Run sync** on the account page runs the same recovery regardless of the cooldown

No configuration is required - this recovery mechanism is built in and runs automatically for all MS Graph accounts with webhook subscriptions enabled.

:::info Public HTTPS Required
MS Graph webhook subscriptions require publicly accessible HTTPS endpoints. If your EmailEngine instance is behind a firewall or on a private network, you will need to configure a reverse proxy or tunnel. Without reachable endpoints, the subscription cannot be created and new-message detection will not work - EmailEngine keeps retrying subscription creation until the endpoints become reachable. There is no polling fallback for MS Graph accounts.
:::

## Official Microsoft Documentation

For the most up-to-date information, refer to Microsoft's official documentation:

### Client Credentials Flow

- [Microsoft identity platform and the OAuth 2.0 client credentials flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant-flow) - Technical reference for the client credentials grant
- [Get access without a user](https://learn.microsoft.com/en-us/graph/auth-v2-service) - MS Graph guide for app-only access

### Application Permissions

- [Microsoft Graph permissions reference](https://learn.microsoft.com/en-us/graph/permissions-reference) - Complete list of available permissions
- [Understanding application-only access](https://learn.microsoft.com/en-us/graph/auth/auth-concepts#application-permissions) - Concepts guide for application vs delegated permissions

### Admin Consent

- [Grant tenant-wide admin consent](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/grant-admin-consent) - How to grant and manage admin consent
- [Configure admin consent workflow](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/configure-admin-consent-workflow) - Setting up approval workflows

## See Also

- [Outlook OAuth2 (delegated)](/docs/accounts/microsoft-365/outlook-365) - The interactive alternative, and when it is required
- [Shared mailboxes](/docs/accounts/microsoft-365/shared-mailboxes) - Reaching a mailbox nobody signs into
- [OAuth2 token management](/docs/accounts/oauth2-token-management) - Client secret lifetimes and what expiry costs
- [Managing accounts](/docs/accounts/managing-accounts) - Registering mailboxes once consent is granted
- [Account troubleshooting](/docs/accounts/troubleshooting) - Consent, tenant, and permission failures
