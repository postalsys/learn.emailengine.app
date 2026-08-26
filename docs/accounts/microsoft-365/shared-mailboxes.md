---
title: Shared Mailboxes (Microsoft 365)
sidebar_label: Shared Mailboxes
sidebar_position: 3
description: Configure and manage Microsoft 365 shared mailboxes with EmailEngine using application access, delegated access, or direct access
keywords:
  - Microsoft 365
  - shared mailboxes
  - Office 365
  - delegated access
  - application access
  - client credentials
  - OAuth2
---

# Shared Mailboxes (Microsoft 365)

Microsoft 365 shared mailboxes are mailboxes not bound to a specific user. Multiple users can access them using their own credentials. EmailEngine supports three approaches for accessing shared mailboxes.

## Choosing an Approach

| Feature | Application Access | Delegated Access | Direct Access |
|---|---|---|---|
| **Setup complexity** | Simple | Moderate | Simple |
| **User login required** | No | Once (main account) | Once per mailbox |
| **Adding new mailboxes** | API call only | API call only | Re-authentication |
| **Admin consent** | Required (once) | Not required | Not required |
| **Email backend** | MS Graph API only | IMAP/SMTP or MS Graph | IMAP/SMTP or MS Graph |
| **Personal accounts** | Not supported | Supported | Supported |
| **Main account usable** | N/A | Yes | No |
| **Best for** | Enterprise, automation | Multi-mailbox with user consent | Testing, single mailbox |

### Application Access (Recommended)

Add any mailbox in your organization - including shared mailboxes - without interactive user login. Uses the OAuth2 client credentials flow with MS Graph API. An Azure AD admin grants the application access to all mailboxes once, and you add accounts by specifying the email address.

**Best for:**

- Enterprise Microsoft 365 deployments
- Automated systems (helpdesk, CRM, compliance)
- Managing many shared mailboxes
- Scenarios where interactive login is not possible

### Delegated Access

Add a main user account with interactive OAuth2 login, then add shared mailboxes that reference the main account's credentials. No additional authentication is needed for each shared mailbox.

**Best for:**

- Environments where admin consent is not available
- Setups where users must individually authorize access
- Mixed environments with personal Microsoft accounts

### Direct Access

Add each shared mailbox individually with its own OAuth2 authentication. A user who has access to the shared mailbox signs in and grants access.

**Best for:**

- Testing and evaluation
- Single shared mailbox setups
- Quick prototyping

:::info Note
With direct access, the authenticating user's credentials are stored with the shared mailbox account. The same user can still have their own primary account in EmailEngine, and the same mailbox can be added as multiple EmailEngine accounts if needed - each EmailEngine account keeps its own copy of the OAuth2 tokens.
:::

## Application Access Setup (Recommended)

Application access is the simplest way to add shared mailboxes. You need an Azure AD application configured with application permissions (client credentials). See [Outlook Application Access (Client Credentials)](./outlook-client-credentials) for the full setup guide.

Once your application is configured in EmailEngine, add a shared mailbox with a single API call:

```bash
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_EMAILENGINE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "shared-support",
    "name": "Support Mailbox",
    "email": "support@company.com",
    "oauth2": {
      "provider": "AAABkTn2CRQAAAAB",
      "auth": { "user": "support@company.com" }
    }
  }'
```

Replace `AAABkTn2CRQAAAAB` with your OAuth2 application ID from EmailEngine (find it under **Integrations > OAuth2 Apps** or via `GET /v1/oauth2`).

No delegation configuration, no interactive login, no extra scopes. The application permissions grant direct access to all mailboxes in the organization.

### Adding Multiple Shared Mailboxes

```bash
# Support mailbox
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "shared-support",
    "email": "support@company.com",
    "oauth2": {
      "provider": "AAABkTn2CRQAAAAB",
      "auth": { "user": "support@company.com" }
    }
  }'

# Sales mailbox
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "shared-sales",
    "email": "sales@company.com",
    "oauth2": {
      "provider": "AAABkTn2CRQAAAAB",
      "auth": { "user": "sales@company.com" }
    }
  }'

# Info mailbox
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "shared-info",
    "email": "info@company.com",
    "oauth2": {
      "provider": "AAABkTn2CRQAAAAB",
      "auth": { "user": "info@company.com" }
    }
  }'
```

All mailboxes use the same OAuth2 application. EmailEngine obtains and renews tokens automatically.

## Delegated Access Setup

Delegated access requires a main user account authenticated via OAuth2. Shared mailboxes then reference this main account for credentials.

### Prerequisites

1. **Azure AD OAuth2 application** configured for delegated access - see [Outlook OAuth2 Setup (Delegated Access)](./outlook-365)
2. **Shared mailbox permissions** - the main user must have access to the shared mailbox in Microsoft 365 admin center
3. **OAuth2 app registered in EmailEngine** under **Integrations > OAuth2 Apps**

### Additional Scopes for MS Graph API

If using the MS Graph API backend, your OAuth2 application needs additional scopes for shared mailbox access.

**Step 1: Add scopes in Azure Portal**

1. Navigate to your app registration in [Azure Portal](https://portal.azure.com/)
2. Go to **API Permissions** > **Add a permission** > **Microsoft Graph** > **Delegated permissions**
3. Add these scopes:

| Scope | Purpose |
|---|---|
| `User.ReadBasic.All` | Resolve shared mailbox identity |
| `Mail.ReadWrite.Shared` | Read and write mail in shared mailboxes |
| `Mail.Send.Shared` | Send mail from shared mailboxes |

**Step 2: Add scopes in EmailEngine**

1. In EmailEngine, open the OAuth2 app under **Integrations** > **OAuth2 Apps** and click **Edit app**
2. Expand the **Additional scopes** section
3. Add the same scopes:
   ```
   User.ReadBasic.All
   Mail.ReadWrite.Shared
   Mail.Send.Shared
   ```
4. Save the changes

**Step 3: Refresh OAuth2 grant**

Existing accounts need to re-authenticate to pick up the new permissions. Either:

- **Re-add the account** - Delete and re-add the main account in EmailEngine
- **Generate new auth link** - Use the [Authentication Form API](/docs/api/post-v-1-authentication-form) with the existing account ID to generate a new authentication URL. The user must open this link and grant the new permissions.

:::info IMAP/SMTP Backend
If using the IMAP/SMTP backend, no additional scopes are needed. Shared mailbox access works out of the box.
:::

### Step 1: Add the Main User Account

Add the main user account via the hosted authentication form:

```bash
curl -X POST https://emailengine.example.com/v1/authentication/form \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "my-account",
    "name": "John Doe",
    "email": "john@company.com",
    "redirectUrl": "https://myapp.com/settings",
    "type": "AAABiCtT7XUAAAAF"
  }'
```

The user completes the OAuth2 login, and the account is added to EmailEngine.

### Step 2: Add Shared Mailboxes

Add shared mailboxes that reference the main account:

```bash
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "shared-support",
    "name": "Support Mailbox",
    "email": "support@company.com",
    "oauth2": {
      "provider": "AAABhaBPHscAAAAH",
      "auth": {
        "delegatedUser": "support@company.com",
        "delegatedAccount": "my-account"
      }
    }
  }'
```

**Key fields:**

- `oauth2.provider` - EmailEngine OAuth2 application ID - use the same application the main account uses. The API rejects a body that sets `delegatedUser` or `delegatedAccount` without it (HTTP 400)
- `oauth2.auth.delegatedUser` - Email address or Microsoft 365 user ID of the shared mailbox
- `oauth2.auth.delegatedAccount` - EmailEngine account ID of the main user (from Step 1)

EmailEngine uses the main account's OAuth2 tokens to access the shared mailbox. No additional authentication is required. You can add multiple shared mailboxes referencing the same main account. Delegation through another account is available since v2.49.0.

## Direct Access Setup

Direct access adds the shared mailbox as a standalone account. A user who has permissions to the shared mailbox authenticates via OAuth2.

### Via Hosted Authentication Form

```bash
curl -X POST https://emailengine.example.com/v1/authentication/form \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "shared-support",
    "name": "Support Mailbox",
    "email": "support@company.com",
    "delegated": true,
    "redirectUrl": "https://myapp.com/settings",
    "type": "AAABiCtT7XUAAAAF"
  }'
```

**Fields:**

- `account` - Account ID for EmailEngine
- `email` - Email address of the shared mailbox
- `delegated` - Must be `true`, indicates the authenticating user is not the mailbox owner
- `type` - OAuth2 application ID from EmailEngine

The user signs in with their **own Microsoft 365 account** (not the shared mailbox). This account must have access to the shared mailbox in Microsoft 365.

### Via Direct API

If you manage OAuth2 tokens externally:

```bash
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "shared-support",
    "name": "Support Mailbox",
    "email": "support@company.com",
    "oauth2": {
      "provider": "AAABlf_0iLgAAAAQ",
      "auth": {
        "user": "admin@company.com",
        "delegatedUser": "support@company.com"
      },
      "accessToken": "ACCESS_TOKEN_FROM_MICROSOFT",
      "refreshToken": "REFRESH_TOKEN_FROM_MICROSOFT"
    }
  }'
```

**Key fields:**

- `oauth2.auth.user` - Email of the user whose credentials are used (the user with access)
- `oauth2.auth.delegatedUser` - Email or Microsoft 365 user ID of the shared mailbox

## Email Address vs UPN Mismatch

In Microsoft 365, a shared mailbox may have a public email address that differs from its User Principal Name (UPN). This commonly happens when:

- The organization uses a custom domain for email (e.g., `shared@contoso.com`)
- The UPN uses the default Microsoft domain (e.g., `sharedmailbox@contoso.onmicrosoft.com`)

Set the `email` field to the public address and use `delegatedUser` (or `auth.user` for application access) to specify the actual UPN.

### With Application Access

```bash
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "shared-support",
    "email": "shared@contoso.com",
    "oauth2": {
      "provider": "AAABkTn2CRQAAAAB",
      "auth": { "user": "sharedmailbox@contoso.onmicrosoft.com" }
    }
  }'
```

### With Delegated Access

```bash
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "shared-support",
    "email": "shared@contoso.com",
    "oauth2": {
      "provider": "AAABhaBPHscAAAAH",
      "auth": {
        "delegatedUser": "sharedmailbox@contoso.onmicrosoft.com",
        "delegatedAccount": "my-account"
      }
    }
  }'
```

The `provider` value is the same EmailEngine OAuth2 application ID the main account uses - the API requires it when setting delegation fields.

### With Direct Access

:::warning Hosted Form Limitation
The hosted authentication form (`/v1/authentication/form`) does not support UPN mismatch. Use the `/v1/account` endpoint with the `authorize` flag instead.
:::

Using the OAuth2 authorization flow:

```bash
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "shared-support",
    "name": "Support Mailbox",
    "email": "shared@contoso.com",
    "oauth2": {
      "authorize": true,
      "provider": "AAABlf_0iLgAAAAQ",
      "redirectUrl": "https://myapp.com/settings",
      "auth": {
        "delegatedUser": "sharedmailbox@contoso.onmicrosoft.com"
      }
    }
  }'
```

The response includes a `redirect` URL. Direct the user there to complete OAuth2 authentication.

### How It Works

- `email` - The public email address (`shared@contoso.com`), used in EmailEngine for display and as the From address
- `delegatedUser` or `auth.user` - The UPN (`sharedmailbox@contoso.onmicrosoft.com`), used internally to access the mailbox via Microsoft Graph or IMAP

## Backend-Specific Considerations

### IMAP/SMTP Backend

Available with delegated access and direct access only (application access uses MS Graph API exclusively).

**IMAP access** works out of the box for shared mailboxes.

**SMTP limitations:**

- Shared mailboxes cannot authenticate directly via SMTP
- EmailEngine authenticates as the main user but sets the From address to the shared mailbox email
- Outlook SMTP saves sent emails to the main account's Sent Items folder
- EmailEngine uploads a copy via IMAP to the shared mailbox's Sent Items folder to compensate

### MS Graph API Backend

Available with all three approaches. Provides the best shared mailbox support.

- No SMTP authentication workarounds needed
- Emails are sent and managed directly as the shared mailbox
- Sent emails saved only in the shared mailbox's Sent Items (no duplicates)

## Verifying Shared Mailbox Access

After adding a shared mailbox, verify it's working:

### Check Account Status

```bash
curl https://emailengine.example.com/v1/account/shared-support \
  -H "Authorization: Bearer YOUR_TOKEN"
```

The account should show `"state": "connected"`. For delegated access accounts the `type` field is `"delegated"`; if the chain to the credential-holding account cannot be resolved, it is `"invalid"` and `delegationError` carries the reason.

### Test Sending Email

```bash
curl -X POST https://emailengine.example.com/v1/account/shared-support/submit \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "to": [{"address": "test@example.com"}],
    "subject": "Test from shared mailbox",
    "text": "This is a test email"
  }'
```

## Delegated Send Access

Beyond shared mailboxes, delegation can be used for scenarios where one account sends emails through another account's credentials. This is a separate feature from shared mailboxes but uses the same `delegatedAccount` mechanism.

**Use cases:**

- Multiple sender identities that share one authenticated account
- Service accounts where several addresses send through a common Microsoft 365 account
- Organizations with centralized email sending infrastructure

:::warning OAuth2 Parent Required
The parent account referenced by `delegatedAccount` must be an OAuth2 account. EmailEngine resolves the parent's OAuth2 tokens and application when sending - a password-based IMAP/SMTP account cannot be used as a delegation parent.
:::

### How Delegation Works Internally

When you configure a delegated account, EmailEngine:

1. **Resolves the delegation chain** - Follows `delegatedAccount` references up to 20 hops (with loop detection)
2. **Loads credentials from the parent account** - Uses the OAuth2 tokens from the referenced account
3. **Sets the identity from the delegated account** - Uses the `email` and `delegatedUser` fields for the From address and mailbox user

### Configuring Delegated Send Access

**Step 1: Create the parent account with OAuth2 credentials**

Add a Microsoft 365 account via the hosted authentication form (the user completes the OAuth2 login):

```bash
curl -X POST https://emailengine.example.com/v1/authentication/form \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "relay-sender",
    "name": "Relay Sender",
    "email": "relay@company.com",
    "redirectUrl": "https://myapp.com/settings",
    "type": "AAABhaBPHscAAAAH"
  }'
```

**Step 2: Create delegated accounts that send through the parent**

```bash
# Sales team sends through the relay account
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "sales-sender",
    "name": "Sales Team",
    "email": "sales@company.com",
    "oauth2": {
      "provider": "AAABhaBPHscAAAAH",
      "auth": {
        "delegatedUser": "sales@company.com",
        "delegatedAccount": "relay-sender"
      }
    }
  }'
```

The `provider` value is the same EmailEngine OAuth2 application ID the parent account uses - the API requires it when setting delegation fields.

**Step 3: Send emails from delegated accounts**

```bash
# Uses relay-sender credentials but sends from sales@company.com
curl -X POST https://emailengine.example.com/v1/account/sales-sender/submit \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "to": [{"address": "customer@example.com"}],
    "subject": "Sales Inquiry",
    "text": "Thank you for your interest. A sales representative will contact you shortly."
  }'
```

### Troubleshooting Delegation Issues

**"Missing account data for delegated account"** - The referenced `delegatedAccount` does not exist. Verify the parent account ID is correct.

**"Delegation looping detected"** - Circular reference in the delegation chain (e.g., A references B, B references C, C references A). Break the loop by ensuring one account has actual credentials.

**"Too many delegation hops"** - The delegation chain exceeds 20 hops. Simplify by referencing the credential-holding account directly.

**Parent account authentication errors** - If the parent account has authentication issues (expired tokens, changed password), all delegated accounts fail. Monitor the parent account's state and fix authentication promptly. A parent that EmailEngine has [switched off after repeated authentication failures](/docs/accounts/managing-accounts#accounts-switched-off-after-authentication-failures) has to be re-authorized before its delegated accounts work again. In v2.79.3 and v2.79.4 the delegated accounts are switched off as well, and re-authorizing the parent lifts only the parent, so each delegated account then needs **Resume syncing** on its account page. v2.79.5 leave delegated accounts alone, since the failing credential is the parent's, and re-authorizing the parent brings them back with it.

## See Also

- [Outlook Application Access (Client Credentials)](./outlook-client-credentials) - Setting up application-level access
- [Outlook OAuth2 Setup (Delegated Access)](./outlook-365) - Setting up delegated OAuth2
- [Account Management](/docs/accounts/managing-accounts) - Managing accounts in EmailEngine
- [Send-Only Accounts](/docs/accounts/imap-smtp#send-only-accounts) - SMTP-only configurations
