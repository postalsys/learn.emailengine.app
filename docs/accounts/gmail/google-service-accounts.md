---
title: Google Service Accounts
sidebar_label: Service Accounts
sidebar_position: 5
description: Setting up Google Service Accounts with domain-wide delegation for Gmail access
---

# Google Service Accounts

A service account lets a Google Workspace admin grant EmailEngine access to any mailbox in the organization without asking each user to consent. This guide covers setting one up with domain-wide delegation.

:::tip Two authentication methods
EmailEngine supports two ways to authenticate a service account:

- **Service account key** - the classic flow with a JSON key file containing a private RSA key. Covered in Steps 1-8 below.
- **Workload Identity Federation (keyless)** - your runtime (GKE, EKS, AKS, or any OIDC source) provides a short-lived token that Google trusts; no long-lived key is stored. See [Alternative: Workload Identity Federation](#alternative-workload-identity-federation-keyless).

Both methods use the same domain-wide delegation setup. Only the signing step differs.
:::

## Overview

### What are Service Accounts?

Service accounts are a special type of Google account intended for applications rather than users. With domain-wide delegation, a service account can act on behalf of any user in your Google Workspace organization.

### How EmailEngine Uses Service Accounts

EmailEngine supports service accounts for two distinct purposes, selected via the **Base scopes** option when configuring the service account:

**1. Direct Email Access (IMAP and SMTP)**

Service accounts with domain-wide delegation can access any mailbox in your Google Workspace organization via IMAP and SMTP protocols. This is the primary use case covered in this guide.

- Select **IMAP and SMTP** as the base scope
- Requires `https://mail.google.com/` scope in domain-wide delegation
- Service account impersonates users to access their mailboxes

:::info IMAP/SMTP Only
When using service accounts for email access, EmailEngine connects via IMAP and SMTP protocols. The Gmail API cannot be used as a backend for service account-based email access in EmailEngine.
:::

**2. Push Notifications for Standard Gmail Accounts (Cloud Pub/Sub)**

Service accounts can also be used to manage Gmail Push Notification subscriptions (Pub/Sub) for regular Gmail OAuth2 accounts. In this scenario:

- Select **Cloud Pub/Sub** as the base scope
- Requires `Pub/Sub Admin` role in Google Cloud (not domain-wide delegation)
- Individual users authenticate with standard OAuth2 (not service accounts)
- A single service account manages Pub/Sub topic subscriptions for all accounts
- Email updates are delivered via push notifications instead of polling
- The service account only handles the notification infrastructure, not email access

For push notification setup, see [Setting Up Gmail API](/docs/accounts/gmail/gmail-api).

### Key Concepts

**Two-Legged OAuth2:**

- No user interaction required
- No consent screens
- Admin grants access once for entire domain
- EmailEngine can access any user's mailbox

**Domain-Wide Delegation:**

- Requires Google Workspace (not free Gmail)
- Requires super admin privileges
- Admin authorizes specific scopes for the service account
- Service account can impersonate any user in the domain

### When to Use Service Accounts

**Best For:**

- Enterprise Google Workspace deployments
- Centralized email management
- Accessing multiple user mailboxes from one application
- Automated workflows without user interaction
- Systems integration and monitoring

**Not Suitable For:**

- Free Gmail accounts (Google Workspace only)
- Public-facing applications (users outside your organization)
- Applications where users should control access

## Benefits and Limitations

### Benefits

**No User Consent Required:**

- Access granted by admin once
- No OAuth2 flow for each user
- No interactive login for any mailbox in the domain

**Centralized Management:**

- Admin controls all access
- Easy to revoke for entire organization
- Audit trail of service account usage

**Scalability:**

- Add new users without re-authentication
- Access any user's mailbox with same credentials
- Ideal for enterprise deployments

### Limitations

**Google Workspace Only:**

- Does not work with free @gmail.com accounts
- Requires paid Google Workspace subscription

**Requires Admin Access:**

- Must be Google Workspace super admin
- Cannot be delegated to non-admin users

**Organization-Scoped:**

- Only works for users in your organization
- Cannot access external Gmail accounts

**Credential Security:**

- Service account credentials provide access to all users
- Must be stored very securely
- Compromised credentials = access to entire organization's email
- For deployments where storing long-lived JSON keys is unacceptable, use [Workload Identity Federation](#alternative-workload-identity-federation-keyless) instead

## Step 1: Create a Google Cloud Project

Go to [Google Cloud Console](https://console.cloud.google.com/) and create a new project.

<!-- Shows: Project selector in Google Cloud Console -->

Click the project selector and then **New Project**.

<!-- Shows: New project button -->

<!-- Shows: Project creation form -->

Fill in the project name (e.g., "EmailEngine Service Account").

<!-- Shows: Project being created, then "Select project" button -->

Wait for the project to be created, then click **Select project** to switch to it.

## Step 2: Configure OAuth Consent Screen

Even though service accounts don't show consent screens to users, the form allows us to configure required project details.

:::info Consent Screen vs Domain-Wide Delegation
The OAuth consent screen and domain-wide delegation are **separate configurations**:

- **OAuth consent screen** (this step): Configures project metadata and documents which scopes your project uses. Required to complete project setup, but service accounts bypass this screen during authentication.
- **Domain-wide delegation** (Step 4): The actual authorization that grants your service account permission to access user mailboxes. This is configured in the Google Admin Console.

Service accounts authenticate using JWT tokens, not the OAuth consent flow, so users never see the consent screen.
:::

Navigate to **APIs & Services** → **OAuth consent screen**.

<!-- Shows: Navigation to OAuth consent screen -->

### Select User Type

<!-- Shows: Internal vs External selection -->

Select **Internal** and click **Create**.

:::note User Type Selection
The "Internal" vs "External" choice primarily affects standard OAuth2 flows where users see consent screens. Since service accounts authenticate using JWT tokens and bypass the consent screen entirely, this setting has no direct impact on service account functionality. Select **Internal** for simplicity since service accounts are designed for organization-internal use.
:::

### Fill in App Information

<!-- Shows: OAuth consent form fields -->

Fill in the required fields:

- **App name**: "EmailEngine Service Account" (or your app name)
- **User support email**: Your email address
- **Developer contact information**: Your email address

These fields are required but won't be shown to users since service accounts don't use consent screens.

Click **Save and continue**.

### Configure Scopes (Optional)

<!-- Shows: Scopes configuration page -->

Click **Add or remove scopes**.

For service accounts, the actual scope authorization happens in the Google Admin Console's domain-wide delegation settings (Step 4), not here. However, adding scopes here documents what your project uses and is required if you later add standard OAuth2 clients.

<!-- Shows: Manually adding scope -->

If desired, scroll to the end of the scopes list and manually add:

```
https://mail.google.com/
```

Click **Add to table** then **Update**.

<!-- Shows: Scope listed in restricted scopes -->

The scope should now appear in the "Restricted scopes" section.

:::info
This step configures project-level scope documentation. The actual permissions for your service account are granted separately through domain-wide delegation in the Admin Console (Step 4). You can skip adding scopes here and proceed directly.
:::

Click **Save and continue** to finish consent screen setup.

## Step 3: Create Service Account

Navigate to **APIs & Services** → **Credentials** and click the **Manage service accounts** link (bottom right corner).

<!-- Shows: Credentials page with "Manage service accounts" link -->

Click **Create service account**.

<!-- Shows: Service accounts page with create button -->

### Configure Service Account

<!-- Shows: Service account creation form -->

**Service account name**: Choose a descriptive name (e.g., "emailengine-service")

**Service account ID**: Auto-generated, but you can customize

**Description**: Optional description

Click **Create and continue**.

### Grant Role

<!-- Shows: Role selection with "Owner" highlighted -->

**Role**: Select **Owner**

:::info Role Selection
The "Owner" role is used here for simplicity. For production, you may want to use a more restrictive role, but "Owner" has been tested and confirmed working.
:::

Click **Continue**.

### Complete Setup

<!-- Shows: Final step, grant users access (can be left empty) -->

Leave the optional fields empty and click **Done**.

## Step 4: Enable Domain-Wide Delegation

Now we need to authorize the service account to access user mailboxes.

### Get Client ID

From the service accounts list, copy the **OAuth2 Client ID** for your service account.

<!-- Shows: Service account list with Client ID column -->

:::tip Finding Client ID
The Client ID is a long numeric string in the service account listing. You'll need this for domain-wide delegation setup.
:::

### Open Google Admin Console

Open [Google Admin Console](https://admin.google.com/) for your domain and search for "API Controls".

<!-- Shows: Google Admin search for "API Controls" -->

### Manage Domain-Wide Delegation

Scroll down to find the "Domain-wide delegation" section and click **Manage domain-wide delegation**.

<!-- Shows: Domain-wide delegation section in Security settings -->

Click **Add new**.

<!-- Shows: API clients list with "Add new" button -->

### Authorize API Client

<!-- Shows: API client authorization form -->

**Client ID**: Paste the Client ID you copied from the service account

**OAuth scopes**: Enter the required scope for IMAP and SMTP access:

```
https://mail.google.com/
```

Click **Authorize**.

:::important This is the Actual Authorization
This step in the Admin Console is where the service account actually receives permission to access user mailboxes. The scope entered here determines what the service account can do. The OAuth consent screen configuration (Step 2) is separate and primarily for project documentation.
:::

## Step 5: Generate Service Account Key

:::tip Prefer a keyless setup?
If your clients prohibit long-lived service-account JSON keys for compliance reasons, you can skip generating a key entirely. See [Alternative: Workload Identity Federation](#alternative-workload-identity-federation-keyless) below. The federation flow replaces only the signing key - every other step in this guide stays the same.
:::

Return to the Google Cloud Console service accounts page.

Open the context menu for your service account (three dots) and click **Manage keys**.

<!-- Shows: Service account context menu with "Manage keys" option -->

Click **Add Key** → **Create new key**.

<!-- Shows: Add Key dropdown menu -->

Select **JSON** as the key type and click **Create**.

<!-- Shows: Key type selection dialog -->

The browser will automatically download the key file as a `.json` file.

:::danger Store Key Securely
This key file provides access to all email accounts in your organization. Store it very securely:

- Use secret management systems (AWS Secrets Manager, Azure Key Vault, etc.)
- Never commit to version control
- Restrict file permissions
- Rotate keys periodically
  :::

### Key File Format

The downloaded JSON file contains:

```json
{
  "type": "service_account",
  "project_id": "service-test-338412",
  "private_key_id": "83fd56801b0e46d21ad88300b73d3727e6d46961",
  "private_key": "-----BEGIN PRIVATE KEY-----\nMIIEvQIBADANBgkqhkiG9w...\n",
  "client_email": "emailengine-2lo-test@service-....iam.gserviceaccount.com",
  "client_id": "103965568215821627203",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://oauth2.googleapis.com/token",
  "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
  "client_x509_cert_url": "https://www.googleapis.com/robot/.../x509/emaile..."
}
```

The important fields for EmailEngine are:

- `client_id`: Maps to "Service client" in EmailEngine
- `client_email`: Maps to "Client Email" in EmailEngine
- `private_key`: Maps to "Secret service key" in EmailEngine
- `project_id`: Maps to "Google Cloud Project ID" in EmailEngine (used for Gmail Push Notifications)

## Step 6: Enable Gmail API

Enabling the Gmail API is not needed for the IMAP and SMTP base scope - EmailEngine authenticates service account users via JWT and IMAP XOAUTH2 using the email address you provide. The Gmail API must be enabled if the application uses the Gmail API base scope.

Navigate to **APIs & Services** → **Dashboard** and click **Enable APIs and Services**.

<!-- Shows: Dashboard with "Enable APIs and Services" button -->

Search for "mail" to find Gmail API.

<!-- Shows: API Library search results -->

<!-- Shows: Gmail API details page -->

Click **Enable**.

<!-- Shows: Gmail API enabled confirmation -->

## Step 7: Configure EmailEngine

Now configure EmailEngine to use the service account.

### Add Gmail Service Account Application

<!-- Shows: EmailEngine OAuth2 configuration page -->

1. Open EmailEngine dashboard
2. Navigate to **Integrations** > **OAuth2 Apps**
3. Click the **Create OAuth2 app** dropdown
4. Select **Gmail Service Accounts**

### Upload Credentials File

<!-- Shows: Service account configuration form in EmailEngine -->

**Credentials file**: Use the file input to select the service account key JSON file

EmailEngine will automatically extract and populate:

- Service client (from `client_id`)
- Client Email (from `client_email`)
- Secret service key (from `private_key`)
- Google Cloud Project ID (from `project_id`)

### Select Base Scopes

The **Base scopes** selection determines how EmailEngine uses this service account:

| Option | Purpose | Required Scope/Role |
|--------|---------|---------------------|
| **IMAP and SMTP** | Direct email access via IMAP/SMTP protocols | `https://mail.google.com/` (domain-wide delegation) |
| **Cloud Pub/Sub** (beta) | Webhook management for Gmail API accounts | `Pub/Sub Admin` role in Google Cloud |

**For direct email access** (the primary use case in this guide), select **IMAP and SMTP**. This allows the service account to access any mailbox in your Google Workspace organization via IMAP and SMTP protocols.

**For push notification management**, select **Cloud Pub/Sub**. This is used when you have regular Gmail OAuth2 accounts (where users authenticate individually) but want a single service account to manage Pub/Sub topic subscriptions for receiving email change notifications. See [Setting Up Gmail API](/docs/accounts/gmail/gmail-api) for this use case.

**Enable this app**: Check if you want it available immediately

Click **Register app** to save.

:::tip Automatic Field Population
When you upload the JSON key file, EmailEngine automatically fills in all required fields. You don't need to copy and paste individual values.
:::

:::info JSON Key File After Setup
Once EmailEngine has been configured with the service account credentials, the JSON key file is no longer needed. EmailEngine stores the credentials in its database. You can delete the JSON file from your local system to reduce security risk.

If you need to configure the same service account in another EmailEngine instance in the future, generate a fresh key from Google Cloud Console rather than reusing an old key file. This follows the security best practice of not storing or sharing key files.
:::

:::warning Enable Field Encryption
By default, EmailEngine stores credentials in cleartext in Redis. To protect sensitive data like service account private keys, you should enable field encryption. See [Setting Up Encryption](/docs/advanced/encryption) for configuration instructions.
:::

### Find Your App ID

After registering the app, note the **App ID** displayed in the OAuth2 application list. This is a base64url-encoded identifier (e.g., `AAABkTn2CRQAAAAB`) that you'll need when adding accounts via the API.

You can also find the App ID by calling the [List OAuth2 Apps API endpoint](/docs/api/get-v-1-oauth-2):

```bash
curl https://emailengine.example.com/v1/oauth2 \
  -H "Authorization: Bearer YOUR_EMAILENGINE_TOKEN"
```

### Verify the Setup

Before adding accounts, use the **Verify setup** button on the OAuth2 app's page (or [`POST /v1/oauth2/{app}/verify`](/docs/api/post-v-1-oauth-2-app-verify)) to test the whole authentication chain. The verifier checks the service account configuration, performs the token exchange, and - if you provide a test mailbox address - validates domain-wide delegation with a live IMAP login or Gmail API probe. Each step is reported as passed, failed, or skipped with a hint for fixing failures, so problems like a missing delegation scope surface immediately instead of when the first account fails to connect.

## Alternative: Workload Identity Federation (Keyless)

Workload Identity Federation (WIF) lets EmailEngine authenticate as a Google service account without storing a long-lived JSON key. Instead, the workload it runs on (a GKE pod, an EKS pod, a VM with managed identity, or any OIDC-emitting source) presents a short-lived token that Google's Security Token Service exchanges for a federated access token. EmailEngine then asks Google to sign the JWT bearer assertion server-side via the IAM Credentials API.

The end result is identical to the key-based flow: a signed JWT is exchanged at `oauth2.googleapis.com/token` for an IMAP XOAUTH2 access token. Only the signing step changes.

### Why use WIF

- **No JSON keys to rotate, leak, or revoke.** The signing key never leaves Google.
- **Compliance.** Many security frameworks now flag long-lived service-account keys; WIF removes them entirely.
- **Audit trail.** Every `signJwt` call is recorded in Google Cloud audit logs against the impersonated service account.

### When to use WIF

WIF is only useful when EmailEngine runs in an environment that can produce a credential Google trusts:

- **Google Kubernetes Engine (GKE)** with Workload Identity bound to a Google IAM service account
- **Amazon EKS** with IAM Roles for Service Accounts (IRSA) projecting a service-account token
- **Azure Kubernetes Service (AKS)** with workload identity / managed identity
- **Self-managed Kubernetes** with a projected service-account token volume
- **Any OIDC identity provider** exposing tokens over HTTP

If you self-host EmailEngine on a plain VM or on the EmailEngine Hosted service, stick with the key-based flow described above.

### What stays the same

Steps 1-4 and 6 of this guide are unchanged. You still:

- Create the Google Cloud project
- Configure the OAuth consent screen
- Create the service account
- Enable domain-wide delegation in the Google Workspace admin console with the same scopes (`https://mail.google.com/`)
- Enable the Gmail API

Instead of Step 5 (generating and downloading a JSON key), federation replaces the signing key with a chain you set up once: enable two APIs, create a workload identity pool and provider, grant the pool one IAM binding on the service account, and point EmailEngine at an external account credential configuration that reads your workload's identity token. The following subsections walk through each part.

### Enable the required APIs

In the Google Cloud project, enable the two APIs the federation flow calls:

- **Security Token Service API** (`sts.googleapis.com`) - exchanges the workload's OIDC token for a federated Google access token.
- **IAM Service Account Credentials API** (`iamcredentials.googleapis.com`) - signs the JWT bearer assertion (`signJwt`).

```bash
gcloud services enable sts.googleapis.com iamcredentials.googleapis.com --project=PROJECT_ID
```

### Create a Workload Identity Pool and provider

Create a workload identity pool and an OIDC provider that trusts the OIDC issuer of the environment EmailEngine runs in (for GKE, the cluster's OIDC issuer URL; for EKS, the cluster OIDC provider URL; for any other source, its issuer).

```bash
# Create the pool
gcloud iam workload-identity-pools create POOL_ID \
  --project=PROJECT_ID \
  --location=global \
  --display-name="EmailEngine WIF pool"

# Create an OIDC provider in the pool
gcloud iam workload-identity-pools providers create-oidc PROVIDER_ID \
  --project=PROJECT_ID \
  --location=global \
  --workload-identity-pool=POOL_ID \
  --issuer-uri="https://ISSUER_OF_YOUR_WORKLOAD" \
  --attribute-mapping="google.subject=assertion.sub"
```

:::note Attribute mapping
`google.subject=assertion.sub` maps the `sub` claim of the workload's OIDC token to the federated principal that the IAM binding below references. Add an `--attribute-condition` if you want to restrict which workload identities the pool accepts.
:::

You will also need the project **number** (not the ID) - it is part of the pool audience used in the binding and the credential configuration:

```bash
gcloud projects describe PROJECT_ID --format="value(projectNumber)"
```

### Grant the pool access to the service account

This is the one federation-specific binding the service account needs. EmailEngine calls `iamcredentials.googleapis.com signJwt` **as the federated pool principal** - it does not first impersonate the service account - so that principal must hold **Service Account Token Creator** (`roles/iam.serviceAccountTokenCreator`) on the target service account:

```bash
gcloud iam service-accounts add-iam-policy-binding GSA_EMAIL \
  --project=PROJECT_ID \
  --role=roles/iam.serviceAccountTokenCreator \
  --member="principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL_ID/*"
```

The `principalSet://.../*` member grants every identity from the pool. To scope it to a single workload identity, use a `principal://.../subject/SUBJECT_VALUE` member instead, where `SUBJECT_VALUE` is the `sub` claim of the workload's OIDC token.

:::danger Workload Identity User is not sufficient
A common mistake - including in Google's own "Grant access" helper - is to grant `roles/iam.workloadIdentityUser`. That role permits `generateAccessToken` but **not** `signJwt`, so EmailEngine's token acquisition fails with a `403` on `signJwt` (surfaced as the `ESignJwt` error). Because EmailEngine signs the assertion directly via `signJwt`, the pool principal must have **`roles/iam.serviceAccountTokenCreator`** on the service account. Do not grant the service account this role on itself - the federated pool principal is the caller, not the service account.
:::

### Generate the external account credential configuration

Use `gcloud iam workload-identity-pools create-cred-config` to produce the JSON file. The exact arguments depend on your workload identity provider, but the output always conforms to the same external-account schema.

**Example for GKE:**

```bash
gcloud iam workload-identity-pools create-cred-config \
  projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL_ID/providers/PROVIDER_ID \
  --service-account=GSA_EMAIL \
  --credential-source-file=/var/run/secrets/tokens/gcp-ksa/token \
  --output-file=external-account.json
```

The resulting `external-account.json` looks like:

```json
{
  "type": "external_account",
  "audience": "//iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL_ID/providers/PROVIDER_ID",
  "subject_token_type": "urn:ietf:params:oauth:token-type:jwt",
  "token_url": "https://sts.googleapis.com/v1/token",
  "service_account_impersonation_url": "https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/GSA_EMAIL:generateAccessToken",
  "credential_source": {
    "file": "/var/run/secrets/tokens/gcp-ksa/token",
    "format": { "type": "text" }
  }
}
```

**Example for an OIDC provider exposing tokens over HTTP** (Azure IMDS, custom OIDC issuer, etc.):

```json
{
  "type": "external_account",
  "audience": "//iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL_ID/providers/PROVIDER_ID",
  "subject_token_type": "urn:ietf:params:oauth:token-type:id_token",
  "token_url": "https://sts.googleapis.com/v1/token",
  "service_account_impersonation_url": "https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/GSA_EMAIL:generateAccessToken",
  "credential_source": {
    "url": "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=AUDIENCE",
    "headers": { "Metadata": "true" },
    "format": { "type": "json", "subject_token_field_name": "access_token" }
  }
}
```

:::info Supported credential sources
EmailEngine supports the `file` and `url` credential sources. The `executable` and AWS sigv4 `aws4_request` variants are not implemented; EKS users should use the projected service-account token at the standard mount path, which uses the `file` source.
:::

### Mount the projected token (GKE)

Your pod spec must mount a projected service-account token volume at the path referenced in `credential_source.file`:

```yaml
spec:
  serviceAccountName: KSA_NAME
  containers:
    - name: emailengine
      volumeMounts:
        - name: gcp-ksa
          mountPath: /var/run/secrets/tokens/gcp-ksa
          readOnly: true
  volumes:
    - name: gcp-ksa
      projected:
        sources:
          - serviceAccountToken:
              path: token
              audience: https://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL_ID/providers/PROVIDER_ID
              expirationSeconds: 3600
```

The Kubernetes kubelet rotates this token automatically before expiry; EmailEngine re-reads the file on every cold token refresh.

### Configure EmailEngine

In the EmailEngine OAuth2 app form:

1. Select **Gmail Service Accounts** as the provider.
2. Choose **Workload Identity Federation (keyless)** as the authentication method.
3. Fill in the **Google Cloud Project ID**, **Client Email**, and **Service client** fields with the same values you would use for the key-based flow. The "Load configuration from the external-account JSON file" button auto-fills the client email from the impersonation URL.
4. Paste the contents of `external-account.json` into the **External account configuration** textarea, or use the file upload button.
5. Click **Register app**.

EmailEngine validates the JSON shape on save and rejects unsupported credential sources or malformed configurations up front.

:::important Client Email must match the impersonated service account
In federation mode EmailEngine issues the JWT bearer assertion as the service account named in the configuration's `service_account_impersonation_url`. The **Client Email** field must be that same service account address. If they differ, EmailEngine rejects the configuration with `EServiceAccountMismatch` rather than failing later with an opaque `invalid_grant`.
:::

The authentication method (service account key vs Workload Identity Federation) is fixed once the application is created. To switch methods, create a new application.

### Verify the setup

After registering the app, use the built-in setup verifier to confirm every step works before pointing real accounts at it. On the OAuth2 application page click **Verify setup**, or call the API:

```bash
curl -X POST https://emailengine.example.com/v1/oauth2/APP_ID/verify \
  -H "Authorization: Bearer YOUR_EMAILENGINE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "account": "user@yourdomain.com" }'
```

The verifier runs the real authentication chain and reports each step as `ok`, `fail`, or `skip` with an actionable hint:

1. **Configuration** - the external account JSON is structurally valid.
2. **Read OIDC subject token** - the credential source (file or url) is readable.
3. **STS token exchange** - the pool, provider, audience, and issuer line up.
4. **Sign assertion (signJwt)** - the pool principal can sign as the service account. This is what catches the Token Creator binding mistake described above.
5. **Domain-wide delegation token** - Google issues an access token for the impersonated user (requires the `account` field).
6. **Mailbox access** - a live, read-only IMAP login confirms the token works and that IMAP access is enabled for the domain.

Steps 1-4 run without an `account`; supply a mailbox address in your Workspace domain to also exercise steps 5-6. The verifier is read-only - it never modifies a mailbox or sends mail.

### Troubleshooting

When token acquisition fails, EmailEngine surfaces a stable error code that points at the failing step. The **Verify setup** action above reports the same codes per step.

| Error code | What it means | Likely fix |
|---|---|---|
| `EExternalAccountConfig` | The stored JSON is malformed, has the wrong `type`, or uses an unsupported credential source. | Re-run `gcloud iam workload-identity-pools create-cred-config` and re-paste the JSON. Ensure `type` is `external_account` and `credential_source` uses `file` or `url`. |
| `EServiceAccountMismatch` | The **Client Email** field does not match the service account in the `service_account_impersonation_url`. | Set Client Email to the same service account address the impersonation URL points at. |
| `ESubjectTokenRead` | EmailEngine could not read the subject token (file missing, URL unreachable, empty response). | Verify the projected token volume is mounted at the path in `credential_source.file`. For `url` sources, confirm the endpoint is reachable and any required headers are set. |
| `ESTSExchange` | Google STS rejected the token exchange. | Common causes: wrong `audience`, the workload's OIDC issuer does not match the provider's `issuer-uri`, the subject token's audience does not match the provider, or the pool/provider is disabled. Check the `error_description` in the EmailEngine error log. |
| `ESignJwt` | The `iamcredentials.googleapis.com signJwt` call failed. | A `403` means the federated pool principal is missing **`roles/iam.serviceAccountTokenCreator`** on the service account (see the binding above - `workloadIdentityUser` is not enough). A `429` means requests are being rate-limited, which is rare under normal token-refresh cadence. |
| `EUnknownAuthMethod` | EmailEngine received an unrecognized `authMethod` value. | Should not occur via the admin UI; indicates direct API misuse. |

If domain-wide delegation has not been authorized in the Workspace admin console for the impersonating service account, the final token exchange at `oauth2.googleapis.com/token` returns `unauthorized_client` regardless of WIF status. That error path is identical to the key-based flow.

### Hosted EmailEngine

The hosted EmailEngine service does not run inside your workload-identity boundary, so WIF is not available there. Hosted customers continue with the key-based flow.

## Step 8: Add Email Accounts

With the service account configured, you can now add email accounts without any user interaction.

### Add Account via the Admin UI

Since service-account apps authenticate without an interactive consent screen, the OAuth2 app's detail page in the EmailEngine dashboard provides an **Add account** button. It opens a dialog asking for the account name and email address and registers the account directly - no API call needed.

The button is available for email-scoped service apps (IMAP/SMTP or Gmail API base scopes). Pub/Sub-scoped service apps grant no mailbox access, so they do not offer it.

### Add Account via API

Add accounts using the [Register Account API endpoint](/docs/api/post-v-1-account):

```bash
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_EMAILENGINE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user123",
    "name": "John Doe",
    "email": "john@company.com",
    "oauth2": {
      "provider": "AAABkTn2CRQAAAAB",
      "auth": {
        "user": "john@company.com"
      }
    }
  }'
```

Replace `AAABkTn2CRQAAAAB` with your actual App ID from EmailEngine.

**Key Points:**

- No `accessToken` or `refreshToken` needed
- No passwords required
- Just specify the user email address
- Service account handles authentication automatically

**Response:**

```json
{
  "account": "user123",
  "state": "new"
}
```

The account will be created and should transition to "connected" state within moments.

### Add Multiple Accounts

You can add any user in your organization using the same App ID:

```bash
# Add sales team accounts (replace APP_ID with your actual App ID)
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "sales-1",
    "email": "alice@company.com",
    "oauth2": {
      "provider": "AAABkTn2CRQAAAAB",
      "auth": { "user": "alice@company.com" }
    }
  }'

curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "sales-2",
    "email": "bob@company.com",
    "oauth2": {
      "provider": "AAABkTn2CRQAAAAB",
      "auth": { "user": "bob@company.com" }
    }
  }'
```

### Verify Accounts

Check the accounts list in EmailEngine:

<!-- Shows: Accounts list showing Gmail OAuth2 accounts -->

Service account-based accounts appear as "OAuth2 (Gmail Service Accounts)" accounts in the list.

## Account Management

### Listing Service Account Accounts

Service account accounts appear like any other OAuth2 account:

```bash
curl https://emailengine.example.com/v1/accounts \
  -H "Authorization: Bearer YOUR_TOKEN"
```

### Updating Service Account Accounts

Update account settings using the [Update Account API endpoint](/docs/api/put-v-1-account-account):

```bash
curl -X PUT https://emailengine.example.com/v1/account/user123 \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "John Doe (Updated)",
    "subconnections": ["\\Sent"]
  }'
```

### Deleting Service Account Accounts

Delete accounts using the [Delete Account API endpoint](/docs/api/delete-v-1-account-account):

```bash
curl -X DELETE https://emailengine.example.com/v1/account/user123 \
  -H "Authorization: Bearer YOUR_TOKEN"
```

## Security Best Practices

### Key Storage

**Do:**

- Store keys in secret management systems (AWS Secrets Manager, Azure Key Vault, HashiCorp Vault)
- Encrypt key files at rest
- Use environment variables for production
- Restrict file permissions (0600)

**Don't:**

- Commit keys to version control
- Store keys in plain text
- Share keys via email or chat
- Store keys in public locations

### Access Control

**Limit Access:**

- Only admins should access service account keys
- Use separate keys for different environments (dev/staging/prod)
- Audit key usage regularly
- Rotate keys periodically (e.g., every 90 days)

**Monitor Usage:**

- Enable Google Cloud audit logs
- Monitor EmailEngine access logs
- Alert on unusual account access patterns
- Review service account activity regularly

### Scope Limitation

**Principle of Least Privilege:**

- Only grant scopes actually needed
- For IMAP access, `https://mail.google.com/` is required
- Consider separate service accounts for different purposes
- Document why each scope is needed

## Use Cases

### Enterprise Email Management

Monitor and manage all employee email accounts:

- Compliance and archiving
- Security scanning
- Backup solutions
- E-discovery

### Help Desk Integration

Access user mailboxes for support:

- Troubleshoot email issues
- Recover deleted messages
- Investigate delivery problems
- Provide user assistance

### Automated Workflows

Build automated systems:

- Invoice processing from shared mailbox
- Customer support ticket creation
- Email-based data extraction
- Notification routing

### Migration and Sync

Synchronize email data:

- Migrate between systems
- Backup to external storage
- Mirror to CRM systems
- Sync with document management

## Official Google Documentation

For the most up-to-date information, refer to Google's official documentation:

### Domain-Wide Delegation

- [Control API access with domain-wide delegation](https://support.google.com/a/answer/162106) - Google Workspace Admin Help guide for setting up domain-wide delegation
- [Domain-wide delegation best practices](https://support.google.com/a/answer/14437356) - Security recommendations and guidelines

### Service Accounts

- [Service accounts overview](https://cloud.google.com/iam/docs/service-account-overview) - Google Cloud IAM documentation
- [Best practices for using service accounts](https://cloud.google.com/iam/docs/best-practices-service-accounts) - Security and management best practices

### Gmail IMAP/SMTP OAuth

- [OAuth 2.0 Mechanism](https://developers.google.com/workspace/gmail/imap/xoauth2-protocol) - SASL XOAUTH2 protocol for IMAP/SMTP
- [IMAP, POP, and SMTP](https://developers.google.com/workspace/gmail/imap/imap-smtp) - Gmail IMAP/SMTP access documentation
- [Choose Gmail API scopes](https://developers.google.com/workspace/gmail/api/auth/scopes) - Available OAuth scopes for Gmail
