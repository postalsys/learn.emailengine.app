---
title: Configuration Options Reference
description: Complete reference of all configuration options and settings
sidebar_position: 3
---

# Configuration Reference

Every EmailEngine configuration option, whether it is set at startup or through the settings API.

:::info Configuration Methods
EmailEngine can be configured via:

1. **Environment variables** (highest priority)
2. **Command-line arguments**
3. **Configuration file** (TOML format)
   :::

## Configuration Types

### Startup Configuration

Loaded at startup. Requires restart to apply changes.

- HTTP port, workers, Redis connection
- Secret for encryption
- License key
- Logging settings
- TLS settings

### Runtime Configuration

Can be updated via Settings API or web interface without restart.

- Service URL
- Webhook URLs and settings
- Email sending limits
- SMTP gateways
- Proxy settings
- OAuth2 applications (configured via API, not environment variables)

## Core Settings

### Server Configuration

#### Port

**Environment:** `EENGINE_PORT`
**Command line:** `--api.port=3000`
**Config file:** `api.port`
**Default:** `3000`

HTTP port for the web interface and API.

```bash
# Environment
export EENGINE_PORT=8080

# Command line
emailengine --api.port=8080
```

```toml
# Config file (config.toml)
[api]
port = 8080
```

#### Host

**Environment:** `EENGINE_HOST`
**Command line:** `--api.host=0.0.0.0`
**Config file:** `api.host`
**Default:** `127.0.0.1`

IP address to bind to. Use `0.0.0.0` to listen on all interfaces.

```bash
EENGINE_HOST=0.0.0.0
```

:::warning Security
Only use `0.0.0.0` if behind a reverse proxy. Otherwise, bind to `127.0.0.1`.
:::

#### Service URL

**Runtime config:** Settings API or web interface
**Setting name:** `serviceUrl`

Base URL for generating links (OAuth redirects, webhook URLs, authentication forms). This is a **runtime setting**, not an environment variable.

```bash
# Configure via API
curl -X POST https://emailengine.example.com/v1/settings \
  -H "Authorization: Bearer TOKEN" \
  -d '{"serviceUrl": "https://emailengine.example.com"}'

# Or via EENGINE_SETTINGS environment variable at startup
EENGINE_SETTINGS='{"serviceUrl":"https://emailengine.example.com"}'
```

### Workers

#### Account Worker Threads

**Environment:** `EENGINE_WORKERS`
**Command line:** `--workers.imap=4`
**Config file:** `workers.imap`
**Default:** `4`

Number of account worker threads for processing accounts. These threads sync all account types - IMAP, Gmail API, and Outlook (Microsoft Graph) - not just IMAP.

```bash
EENGINE_WORKERS=8
```

**Recommendation:** Set to number of CPU cores for optimal performance.

```bash
# Auto-detect CPU cores (Linux)
EENGINE_WORKERS=$(nproc)
```

#### API Worker Threads

**Environment:** `EENGINE_WORKERS_API`
**Command line:** `--workers.api=1`
**Config file:** `workers.api`
**Default:** `1`

Number of API/HTTP worker threads that serve the REST API and admin UI. Values above `1` require `SO_REUSEPORT` so the workers can share the listen port, which is only available on **Linux with Node.js 23.1 or newer**. On macOS, Windows, or older Node.js, EmailEngine falls back to a single API worker and shows a notice on the Workers page (`/admin/internals`).

```bash
# Linux + Node.js >= 23.1
EENGINE_WORKERS_API=4
```

#### Webhook Workers

**Environment:** `EENGINE_WORKERS_WEBHOOKS`
**Config file:** `workers.webhooks`
**Default:** `1`

Number of webhook delivery worker threads.

#### Submit Workers

**Environment:** `EENGINE_WORKERS_SUBMIT`
**Config file:** `workers.submit`
**Default:** `1`

Number of email submission worker threads.

### Redis Configuration

#### Connection URL

**Environment:** `EENGINE_REDIS`
**Command line:** `--dbs.redis=redis://localhost:6379/8`
**Config file:** `dbs.redis`
**Default:** `redis://127.0.0.1:6379/8`

Redis connection URL. The default database number is 8.

```bash
# Default (database 8)
EENGINE_REDIS=redis://localhost:6379/8

# With password
EENGINE_REDIS=redis://:password@localhost:6379/8

# With username and password
EENGINE_REDIS=redis://username:password@localhost:6379/8

# With different database number
EENGINE_REDIS=redis://localhost:6379/5

# TLS connection
EENGINE_REDIS=rediss://localhost:6379/8

# IPv6 only
EENGINE_REDIS=redis://localhost:6379/8?family=6
```

#### Redis Key Prefix

**Environment:** `EENGINE_REDIS_PREFIX`
**Default:** Empty

Prefix for all Redis keys. Useful when sharing a Redis instance.

```bash
EENGINE_REDIS_PREFIX=ee1:
```

### Security

#### Secret

**Environment:** `EENGINE_SECRET`
**Command line:** `--service.secret=...`
**Config file:** `service.secret`
**Required:** Yes (for production)

Secret used for encrypting session tokens AND account credentials (passwords, OAuth tokens). This is a single secret that serves both purposes.

**Generate and save:**

```bash
# Generate a 64-character hex secret
openssl rand -hex 32

# Add to environment or .env file
EENGINE_SECRET=your-generated-secret-here
```

:::danger Required for Production
EmailEngine will start without this secret in development mode, but **all sensitive data will be stored unencrypted**. Always set a strong secret in production.
:::

:::warning Secret Management

- Keep a secure backup of your secret
- If lost, encrypted credentials cannot be recovered
- Changing the secret requires re-encrypting existing data using the `encrypt` command
  :::

**Re-encrypting data after secret change:**

```bash
emailengine encrypt --dbs.redis="redis://url" --service.secret="new-secret" --decrypt="old-secret"
```

### Logging

#### Log Level

**Environment:** `EENGINE_LOG_LEVEL`
**Command line:** `--log.level=info`
**Config file:** `log.level`
**Default:** `trace`

Logging verbosity level.

**Levels:** `silent`, `fatal`, `error`, `warn`, `info`, `debug`, `trace`

```bash
# Production
EENGINE_LOG_LEVEL=info

# Development
EENGINE_LOG_LEVEL=trace

# Minimal
EENGINE_LOG_LEVEL=warn
```

#### Log Raw

**Environment:** `EENGINE_LOG_RAW`
**Config file:** `log.raw`
**Default:** `false`

Log raw IMAP/SMTP protocol traffic and OAuth2 API requests.

```bash
EENGINE_LOG_RAW=true
```

:::danger Security Warning - Contains Unmasked Credentials
When enabled, logs contain **unmasked sensitive data**:

- **IMAP/SMTP traffic**: Base64-encoded protocol data includes plaintext passwords in AUTH commands
- **OAuth2 API requests**: Access tokens, refresh tokens, and client secrets are logged without filtering
- **All credentials are visible**: Nothing is redacted or masked

**Never enable in production.** Use only for local debugging, and delete logs immediately after troubleshooting.
:::

:::warning Performance
Enabling this creates very large log files due to the volume of protocol data.
:::

## OAuth2 Configuration

OAuth2 applications (Gmail, Outlook, Mail.ru) are configured via the **Settings API or web interface**, not environment variables.

### Configuring OAuth2 via API

**Endpoint:** `POST /v1/oauth2` - [API Reference](/docs/api/post-v-1-oauth-2)

**Supported providers:**

- `gmail` - Gmail with 3-legged OAuth2 (user consent)
- `gmailService` - Gmail with service account (2-legged OAuth2)
- `outlook` - Microsoft 365 / Outlook (delegated access)
- `outlookService` - Microsoft 365 via client credentials (application access)
- `mailRu` - Mail.ru

```bash
# Create Gmail OAuth2 application (3-legged OAuth2)
curl -X POST https://emailengine.example.com/v1/oauth2 \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Gmail",
    "provider": "gmail",
    "clientId": "123456789.apps.googleusercontent.com",
    "clientSecret": "GOCSPX-xxxxxxxxxxxxx",
    "baseScopes": "imap",
    "enabled": true
  }'

# Create Outlook OAuth2 application
curl -X POST https://emailengine.example.com/v1/oauth2 \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Outlook",
    "provider": "outlook",
    "clientId": "12345678-1234-1234-1234-123456789012",
    "clientSecret": "your-azure-secret",
    "authority": "common",
    "enabled": true
  }'

# Create Gmail Service Account application (2-legged OAuth2)
curl -X POST https://emailengine.example.com/v1/oauth2 \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Gmail Service Account",
    "provider": "gmailService",
    "serviceClient": "123456789012345678901",
    "serviceClientEmail": "myapp@project-123.iam.gserviceaccount.com",
    "serviceKey": "-----BEGIN PRIVATE KEY-----\nMIIEv...\n-----END PRIVATE KEY-----",
    "enabled": true
  }'
```

**Key fields:**

- `name` - Display name for the application
- `provider` - One of: `gmail`, `gmailService`, `outlook`, `outlookService`, `mailRu`
- `clientId` / `clientSecret` - OAuth2 credentials (for 3-legged OAuth2)
- `serviceClient` / `serviceClientEmail` / `serviceKey` - Service account credentials (for Gmail service accounts)
- `authority` - Microsoft tenant: `common`, `organizations`, `consumers`, or tenant ID
- `baseScopes` - Connection type: `imap`, `api`, or `pubsub`
- `enabled` - Whether the application is active

### Configuring OAuth2 via Web Interface

1. Navigate to **Integrations > OAuth2 Apps**
2. Click **Register new application**
3. Select provider (Gmail, Outlook, Outlook Application Access, Gmail Service Account, or Mail.ru)
4. Enter your OAuth2 credentials
5. Click **Register application**

### Gmail Pub/Sub Subscription TTL

**Setting name:** `gmailSubscriptionTtl`
**Default:** Google default (31 days)

Controls how long a Gmail Pub/Sub subscription persists without activity before Google automatically deletes it. Configured via `POST /v1/settings` or the admin UI at **Integrations > OAuth2 Apps > Gmail Subscriptions**.

| Value | Behavior |
|-------|----------|
| Empty / not set | Google default (31 days) |
| `0` | No expiration (never expires) |
| `1`-`365` | Custom TTL in days |

```bash
curl -X POST https://emailengine.example.com/v1/settings \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"gmailSubscriptionTtl": 0}'
```

Changes take effect when a subscription is next created or updated, not immediately. See [Gmail Pub/Sub Integration](/docs/accounts/gmail/gmail-pubsub#subscription-expiration-ttl) for details.

### Related API Endpoints

- [List OAuth2 applications](/docs/api/get-v-1-oauth-2) - `GET /v1/oauth2`
- [Register OAuth2 application](/docs/api/post-v-1-oauth-2) - `POST /v1/oauth2`
- [Get OAuth2 application](/docs/api/get-v-1-oauth-2-app) - `GET /v1/oauth2/{app}`
- [Update OAuth2 application](/docs/api/put-v-1-oauth-2-app) - `PUT /v1/oauth2/{app}`
- [Delete OAuth2 application](/docs/api/delete-v-1-oauth-2-app) - `DELETE /v1/oauth2/{app}`

For detailed OAuth2 setup instructions, see:

- [Gmail OAuth2 Setup](/docs/accounts/gmail/gmail-imap)
- [Outlook OAuth2 Setup](/docs/accounts/microsoft-365/outlook-365)

## Performance Tuning

### Command Timeout

**Environment:** `EENGINE_TIMEOUT`
**Command line:** `--service.commandTimeout=10000`
**Config file:** `service.commandTimeout`
**Default:** `10000` (10 seconds)

Timeout for IMAP commands in milliseconds.

```bash
EENGINE_TIMEOUT=30000
```

### Fetch Batch Size

**Environment:** `EENGINE_FETCH_BATCH_SIZE`
**Command line:** `--service.fetchBatchSize=1000`
**Config file:** `service.fetchBatchSize`
**Default:** `1000`

Number of messages to fetch per batch during synchronization.

```bash
EENGINE_FETCH_BATCH_SIZE=500
```

**Lower values:** Slower sync, less memory usage
**Higher values:** Faster sync, more memory usage

### Download Chunk Size

**Environment:** `EENGINE_CHUNK_SIZE`
**Default:** `1000000` (1 MB)

Chunk size for streaming large message downloads.

```bash
EENGINE_CHUNK_SIZE=2000000
```

### URL Fetch Timeout

**Environment:** `EENGINE_FETCH_TIMEOUT`
**Default:** `90000` (90 seconds)

Timeout for fetching external URLs (attachments, templates) in milliseconds.

```bash
EENGINE_FETCH_TIMEOUT=120000
```

### Connection Setup Delay

**Environment:** `EENGINE_CONNECTION_SETUP_DELAY`
**Default:** `0`

Delay in milliseconds between setting up account connections. Useful for rate limiting during startup.

```bash
EENGINE_CONNECTION_SETUP_DELAY=100
```

## API Server Settings

### Max Body Size

**Environment:** `EENGINE_MAX_BODY_SIZE`
**Default:** `50MB`

Maximum request body size for API requests.

```bash
EENGINE_MAX_BODY_SIZE=100MB
```

### Max Payload Timeout

**Environment:** `EENGINE_MAX_PAYLOAD_TIMEOUT`
**Default:** `10000` (10 seconds)

Timeout for receiving request payloads.

```bash
EENGINE_MAX_PAYLOAD_TIMEOUT=30000
```

### CORS Configuration

#### CORS Origin

**Environment:** `EENGINE_CORS_ORIGIN`
**Default:** None (CORS disabled)

Allowed CORS origins. Set to `*` for all origins or specific domain.

```bash
EENGINE_CORS_ORIGIN=https://your-app.com
```

#### CORS Max Age

**Environment:** `EENGINE_CORS_MAX_AGE`
**Default:** `60`

CORS preflight cache duration in seconds.

```bash
EENGINE_CORS_MAX_AGE=3600
```

### API Authentication

#### Require API Authentication

**Environment:** `EENGINE_REQUIRE_API_AUTH`
**Default:** `true`

Whether API requests require authentication tokens.

```bash
# Disable for development only
EENGINE_REQUIRE_API_AUTH=false
```

:::danger Security
Never disable API authentication in production.
:::

### TLS Configuration

#### Enable API TLS

**Environment:** `EENGINE_API_TLS`
**Default:** `false`

Enable TLS for the API server.

```bash
EENGINE_API_TLS=true
```

#### TLS Minimum Version

**Environment:** `EENGINE_TLS_MIN_VERSION`
**Default:** `TLSv1`

Minimum TLS version to accept.

```bash
EENGINE_TLS_MIN_VERSION=TLSv1.2
```

#### TLS Ciphers

**Environment:** `EENGINE_TLS_CIPHERS`
**Default:** `DEFAULT@SECLEVEL=0`

TLS cipher suites to use.

```bash
EENGINE_TLS_CIPHERS=ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384
```

#### TLS Min DH Size

**Environment:** `EENGINE_TLS_MIN_DH_SIZE`
**Default:** `1024`

Minimum DH parameter size.

```bash
EENGINE_TLS_MIN_DH_SIZE=2048
```

### Reverse Proxy Mode

**Environment:** `EENGINE_API_PROXY`
**Default:** `false`

When enabled, client IP addresses are read from the `X-Forwarded-For` header instead of the socket connection. Enable this when running EmailEngine behind a reverse proxy (Nginx, HAProxy, load balancer, etc.).

```bash
EENGINE_API_PROXY=true
```

### Trusted Proxy Addresses

**Environment:** `EENGINE_API_PROXY_ADDRESSES`
**Default:** none

Comma-separated IP addresses or CIDR ranges of your own proxies. Only requests arriving from one of these peers are allowed to set `X-Forwarded-For`, and entries contributed by these proxies are discarded when resolving the client address.

```bash
EENGINE_API_PROXY=true
EENGINE_API_PROXY_ADDRESSES=10.0.0.0/8,192.168.1.10
```

:::warning Security
`X-Forwarded-For` is a list that each proxy appends to, so the left-most entry is whatever the original caller sent. With `EENGINE_API_PROXY=true` and no proxy addresses declared, EmailEngine keeps that left-most entry, which means any client able to reach the port directly chooses the address EmailEngine records.

That is fine for logging, but it is not safe for the two controls that match on the client address: the [admin interface allowlist](/docs/deployment/security#admin-interface-access-control) (`EENGINE_ADMIN_ACCESS_ADDRESSES`) and per-token `restrictions.addresses`. If you rely on either, set `EENGINE_API_PROXY_ADDRESSES` as well, or leave `EENGINE_API_PROXY` disabled so the socket address is used.

EmailEngine logs a warning at startup when it detects an admin allowlist combined with undeclared proxies.
:::

## Webhooks

Webhook settings are configured via the **Settings API or web interface**.

### Configure Webhooks via API

```bash
# Set webhook URL and events
curl -X POST https://emailengine.example.com/v1/settings \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "webhooks": "https://your-app.com/webhooks",
    "webhooksEnabled": true,
    "webhookEvents": ["messageNew", "messageDeleted", "accountAuthenticationError"]
  }'
```

### Pre-configure Webhooks at Startup

Use `EENGINE_SETTINGS` to configure webhooks at startup:

```bash
EENGINE_SETTINGS='{"webhooks":"https://your-app.com/webhooks","webhooksEnabled":true,"webhookEvents":["*"]}'
```

### Webhook Payload Settings

Control what data is included in webhook payloads. These are runtime settings configured via `POST /v1/settings`.

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `notifyText` | boolean | `false` | Include plain text content in webhook payloads |
| `notifyTextSize` | number | none | Max text size in webhook payloads (bytes) |
| `notifyAttachments` | boolean | `false` | Include attachment data in webhook payloads |
| `notifyAttachmentSize` | number | none | Max attachment size in webhook payloads (bytes) |
| `notifyCalendarEvents` | boolean | `false` | Include calendar event data in webhook payloads |
| `notifyWebSafeHtml` | boolean | `false` | Replace the HTML body in webhook payloads with a [web-safe](/docs/receiving/web-safe-html) version, sanitized and with quoted thread history folded |
| `notifyHeaders` | string | none | Comma-separated list of email headers to include in webhook payloads |

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "notifyText": true,
    "notifyTextSize": 50000,
    "notifyAttachments": true,
    "notifyAttachmentSize": 1048576,
    "notifyCalendarEvents": true,
    "notifyHeaders": "list-unsubscribe,x-mailer"
  }'
```

## Monitoring

### Metrics Endpoint

The Prometheus metrics endpoint is available at `/metrics` on the main API server. It requires authentication with a token that has the `metrics` scope.

**Endpoint:** `http://localhost:3000/metrics`

**Authentication:** Requires API token with `metrics` scope (or `*` for full access)

```bash
# Create a metrics-only token via CLI
emailengine tokens issue -d "Prometheus" -s "metrics"

# Or create via API
curl -X POST https://emailengine.example.com/v1/tokens \
  -H "Authorization: Bearer ADMIN_TOKEN" \
  -d '{"description": "Prometheus", "scopes": ["metrics"]}'

# Access metrics
curl -H "Authorization: Bearer METRICS_TOKEN" http://localhost:3000/metrics
```

### Health Check

**Endpoint:** `/health`
**Always enabled, no authentication required**

Returns health status.

```bash
curl http://localhost:3000/health
```

## License

### License Key

**Environment:** `EENGINE_PREPARED_LICENSE`
**Command line:** `--preparedLicense=...`
**Config file:** `preparedLicense`
**Default:** 14-day trial

Production license key.

```bash
# From file
EENGINE_PREPARED_LICENSE="$(cat /path/to/license.txt)"

# Inline (PEM format)
EENGINE_PREPARED_LICENSE="-----BEGIN LICENSE-----
Application: EmailEngine
Licensed to: Your Company

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
-----END LICENSE-----"
```

:::info Trial Mode
Without a license key, EmailEngine runs in 14-day trial mode with full functionality. Activate trial via the dashboard.
:::

## Pre-configured Settings

### Prepared Settings

**Environment:** `EENGINE_SETTINGS`

JSON string of runtime settings to apply at startup.

```bash
EENGINE_SETTINGS='{"serviceUrl":"https://emailengine.example.com","webhooks":"https://your-app.com/webhooks","webhooksEnabled":true}'
```

### Prepared API Token

**Environment:** `EENGINE_PREPARED_TOKEN`
**Command line:** `--preparedToken=...`
**Config file:** `preparedToken`

Pre-configure an API access token at startup. This accepts an **exported token hash**, not the actual access token value.

**Workflow:**

1. Generate a token: `emailengine tokens issue -d "API Token" -s "*"`
2. Export the token: `emailengine tokens export -t <token-value>`
3. Use the exported string (not the original token) as `EENGINE_PREPARED_TOKEN`

```bash
# The value is an exported token hash from "emailengine tokens export"
EENGINE_PREPARED_TOKEN=hKJpZNlAMzAxZThjNTFhZjgxM2Q3MzUxNTYzYTFlM2I1NjVkYmEzZWJjMzk4ZjI4OWZjNjgzN...
```

For complete token management workflow, see [Prepared Tokens](/docs/configuration/prepared-settings/tokens).

### Prepared Admin Password

**Environment:** `EENGINE_PREPARED_PASSWORD`
**Command line:** `--preparedPassword=...`
**Config file:** `preparedPassword`

Pre-configure the admin password at startup. This accepts a **base64url-encoded password hash**, not a plain password.

**Workflow:**

1. Generate a password hash: `emailengine password -p "your-password" --hash`
2. Use the returned hash as `EENGINE_PREPARED_PASSWORD`

```bash
# Generate hash
emailengine password -p "my-secure-password" --hash
# Output: JHBia2RmMi1zaGE1MTIkaTEwMDAwMCRhYmNkZWYxMjM0NTY3ODkw...

# Use the hash (not the plain password)
EENGINE_PREPARED_PASSWORD=JHBia2RmMi1zaGE1MTIkaTEwMDAwMCRhYmNkZWYxMjM0NTY3ODkw...
```

### Disable Setup Warnings

**Environment:** `EENGINE_DISABLE_SETUP_WARNINGS`
**Default:** `false`

Disable admin password setup warnings.

```bash
EENGINE_DISABLE_SETUP_WARNINGS=true
```

## Admin Access Control

### Admin Access Addresses

**Environment:** `EENGINE_ADMIN_ACCESS_ADDRESSES`
**Default:** All addresses

Comma-separated list of IP addresses allowed to access admin interface.

```bash
EENGINE_ADMIN_ACCESS_ADDRESSES=127.0.0.1,10.0.0.0/8
```

### Disable Message Browser

**Environment:** `EENGINE_DISABLE_MESSAGE_BROWSER`
**Default:** `false`

Disable the message browser in the admin interface.

```bash
EENGINE_DISABLE_MESSAGE_BROWSER=true
```

## Locale and Branding

Runtime settings configured via `POST /v1/settings` that customize the admin UI and hosted pages.

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `locale` | string | auto | UI language/locale |
| `timezone` | string | auto | Default timezone for date/time display (IANA identifier, e.g. `Europe/Tallinn`) |
| `pageBrandName` | string | none | Custom brand name displayed in page titles |
| `templateHeader` | string | none | Custom HTML injected at the top of hosted pages (max 1 MB) |
| `templateHtmlHead` | string | none | Custom HTML injected into the `<head>` section of hosted pages (max 1 MB) |

`templateHeader` and `templateHtmlHead` are useful for adding custom branding, analytics scripts, or CSS to the hosted authentication forms and other hosted pages.

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "locale": "en",
    "timezone": "America/New_York",
    "pageBrandName": "My Company"
  }'
```

## IMAP Settings

### Disable IMAP Compression

**Environment:** `EENGINE_DISABLE_COMPRESSION`
**Default:** `false`

Disable IMAP COMPRESS extension.

```bash
EENGINE_DISABLE_COMPRESSION=true
```

### IMAP Socket Timeout

**Environment:** `EENGINE_IMAP_SOCKET_TIMEOUT`
**Default:** None (system default)

Socket timeout for IMAP connections in milliseconds.

```bash
EENGINE_IMAP_SOCKET_TIMEOUT=300000
```

### Max IMAP Auth Failure Time

**Environment:** `EENGINE_MAX_IMAP_AUTH_FAILURE_TIME`
**Default:** `3d` (3 days)

How long an account may keep failing authentication before EmailEngine stops retrying it. Once the first authentication error is older than this window, EmailEngine sets `imap.disabled` on the account, records the reason in the account's last error state, closes the connection, and sends an [`authenticationError`](/docs/webhooks/authenticationerror) webhook. This keeps a mailbox with a changed password, or an OAuth2 grant the user has revoked, from hammering the mail server or the provider's token endpoint indefinitely.

```bash
EENGINE_MAX_IMAP_AUTH_FAILURE_TIME=1d
```

The threshold covers every account type. Gmail API and Outlook API accounts have no IMAP configuration of their own, so EmailEngine writes the flag for them, and both clients check it before attempting a token refresh. Before 2.79.3 the check was gated on stored IMAP settings, which OAuth2 accounts do not have, so their revoked grants were retried indefinitely. Upgrading an instance that has collected such accounts parks every one already past the threshold on its next failed refresh, so expect a burst of `authenticationError` webhooks. Raise this value before upgrading to stage that.

:::warning A parked account stays parked
Nothing lifts the flag on its own, because a parked account is never retried. `imap.disabled` in the [Get Account](/docs/api/get-v-1-account-account) response is what tells a parked account apart from one that is merely failing, including for OAuth2 accounts that carry no other IMAP settings.
:::

For a password account, saving its IMAP settings again clears the flag: the `imap` object you send replaces the stored one, whether it comes from the account edit page in the admin interface or from [Update Account](/docs/api/put-v-1-account-account). Otherwise clear the flag on its own and ask for a reconnect:

```bash
curl -X PUT "https://emailengine.example.com/v1/account/user123" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"imap": {"partial": true, "disabled": false}}'

curl -X PUT "https://emailengine.example.com/v1/account/user123/reconnect" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"reconnect": true}'
```

`partial: true` is what keeps this from being destructive - see [Disabling and enabling accounts](/docs/accounts/managing-accounts#disabling-and-enabling-accounts). Fix the credentials first, or the account is parked again once the window passes.

### IMAP ID Extension

Runtime settings configured via `POST /v1/settings`. These customize the IMAP ID extension response ([RFC 2971](https://tools.ietf.org/html/rfc2971)) sent to mail servers, which can affect server-side behavior.

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `imapClientName` | string | none | Client name reported to IMAP servers |
| `imapClientVersion` | string | none | Client version reported to IMAP servers |
| `imapClientVendor` | string | none | Vendor name reported to IMAP servers |
| `imapClientSupportUrl` | string | none | Support URL reported to IMAP servers |

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "imapClientName": "EmailEngine",
    "imapClientVersion": "2.63.4",
    "imapClientVendor": "Postal Systems"
  }'
```

Some mail servers adjust rate limits, feature availability, or logging based on the IMAP ID values provided by the client.

## Queue Settings

### Submit Queue Concurrency

**Environment:** `EENGINE_SUBMIT_QC`
**Default:** `1`

Concurrency for email submission queue.

```bash
EENGINE_SUBMIT_QC=4
```

### Submit Delay

**Environment:** `EENGINE_SUBMIT_DELAY`
**Default:** None

Delay between email submissions in milliseconds.

```bash
EENGINE_SUBMIT_DELAY=1000
```

### Notify Queue Concurrency

**Environment:** `EENGINE_NOTIFY_QC`
**Default:** `1`

Concurrency for notification/webhook queue.

```bash
EENGINE_NOTIFY_QC=4
```

### Queue History Size

**Environment:** `EENGINE_QUEUE_REMOVE_AFTER`
**Default:** `0` (keep none)

Number of completed queue entries to retain for debugging (this is a count of jobs, not a duration). Stored internally as the `queueKeep` setting.

```bash
EENGINE_QUEUE_REMOVE_AFTER=1000
```

### Failed Entry Retention

**Environment:** `EENGINE_QUEUE_KEEP_FAILED`, `EENGINE_QUEUE_KEEP_FAILED_AGE`
**Default:** `500` entries, `604800` seconds (7 days)

Failed entries are retained independently of `EENGINE_QUEUE_REMOVE_AFTER`, because a failed job is the only record that a delivery was given up on. Setting the history size above 500 raises this floor with it.

```bash
EENGINE_QUEUE_KEEP_FAILED=2000
EENGINE_QUEUE_KEEP_FAILED_AGE=259200  # 3 days
```

See [Queue Management](/docs/advanced/queue-management#enable-job-retention).

## SMTP Proxy Server

EmailEngine can run an SMTP proxy server that accepts SMTP connections and routes them through configured accounts.

### Enable SMTP Proxy

**Environment:** `EENGINE_SMTP_ENABLED`
**Default:** `false`

```bash
EENGINE_SMTP_ENABLED=true
```

### SMTP Proxy Port

**Environment:** `EENGINE_SMTP_PORT`
**Default:** `2525`

```bash
EENGINE_SMTP_PORT=587
```

### SMTP Proxy Host

**Environment:** `EENGINE_SMTP_HOST`
**Default:** `127.0.0.1`

```bash
EENGINE_SMTP_HOST=0.0.0.0
```

### SMTP Proxy Secret

**Environment:** `EENGINE_SMTP_SECRET`
**Default:** Empty

Password for SMTP proxy authentication. If not set, API access tokens with the `smtp` scope can be used for authentication instead.

```bash
EENGINE_SMTP_SECRET=your-smtp-password
```

### SMTP PROXY Protocol

**Environment:** `EENGINE_SMTP_PROXY`
**Setting:** `smtpServerProxy`
**Default:** `false`

Accept the HAProxy PROXY protocol on the SMTP server, so the client address EmailEngine sees is the original caller rather than the load balancer. Only enable it when something in front actually speaks the protocol: a plain SMTP client connecting to a listener expecting a PROXY header is rejected.

```bash
EENGINE_SMTP_PROXY=true
```

### SMTP Max Message Size

**Environment:** `EENGINE_MAX_SMTP_MESSAGE_SIZE`
**Default:** `25MB`

Maximum message size the SMTP proxy server accepts for delivery.

```bash
EENGINE_MAX_SMTP_MESSAGE_SIZE=50MB
```

## IMAP Proxy Server

EmailEngine can run an IMAP proxy server that accepts IMAP connections and routes them through configured accounts.

### Enable IMAP Proxy

**Environment:** `EENGINE_IMAP_PROXY_ENABLED`
**Default:** `false`

```bash
EENGINE_IMAP_PROXY_ENABLED=true
```

### IMAP Proxy Port

**Environment:** `EENGINE_IMAP_PROXY_PORT`
**Default:** `2993`

```bash
EENGINE_IMAP_PROXY_PORT=993
```

### IMAP Proxy Host

**Environment:** `EENGINE_IMAP_PROXY_HOST`
**Default:** `127.0.0.1`

```bash
EENGINE_IMAP_PROXY_HOST=0.0.0.0
```

### IMAP Proxy Secret

**Environment:** `EENGINE_IMAP_PROXY_SECRET`
**Default:** Empty

Password for IMAP proxy authentication. If not set, API access tokens with the `imap-proxy` scope can be used for authentication instead.

```bash
EENGINE_IMAP_PROXY_SECRET=your-imap-password
```

### IMAP PROXY Protocol

**Environment:** `EENGINE_IMAP_PROXY_PROXY`
**Default:** `false`

Enable PROXY protocol for IMAP proxy.

```bash
EENGINE_IMAP_PROXY_PROXY=true
```

## OAuth Token Access

### Enable OAuth Tokens API

**Environment:** `EENGINE_ENABLE_OAUTH_TOKENS_API`
**Default:** `false`

Allow retrieving raw OAuth tokens via API.

```bash
EENGINE_ENABLE_OAUTH_TOKENS_API=true
```

:::warning Security
Only enable if you need to access raw OAuth tokens. This exposes sensitive credentials.
:::

## Access Token Audit Log

**Setting:** `tokenAuditLog`
**Default:** `false`

Records every request each access token makes, allowed and denied alike, readable with [`GET /v1/tokens/{token}/log`](/docs/api-reference/access-tokens#audit-log). Entries exist only from the moment it is switched on, so turning it on after an incident recovers nothing.

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"tokenAuditLog": true}'
```

Retention per token is bounded by `EENGINE_TOKEN_LOG_ENTRIES` (1000) and `EENGINE_TOKEN_LOG_AGE` (7 days). The log lives in Redis, so both cost memory.

## Behind a Reverse Proxy

**Setting:** `enableApiProxy`
**Default:** `false`

Trust `X-Forwarded-*` headers, so the client address EmailEngine records and checks against token IP restrictions is the real caller rather than the proxy.

Pair it with `EENGINE_API_PROXY_ADDRESSES`, a comma-separated list of the proxy addresses the headers may be trusted from. Without that list, `X-Forwarded-For` is honored from anywhere, which lets a caller state any address it likes.

```bash
EENGINE_API_PROXY_ADDRESSES=10.0.0.5,10.0.0.6
```

## MCP Endpoint

Serves the [Model Context Protocol](/docs/mcp) at `/mcp` so AI agents can work with the connected accounts. Two switches, both required.

### Register the MCP Routes

**Environment:** `EENGINE_MCP_ENABLED`
**Config file:** `[mcp] enabled`
**CLI flag:** `--mcp.enabled`
**Default:** `true`

Deployment gate. When false, the `/mcp` routes, the OAuth endpoints and the MCP configuration page are not registered at all, and the runtime setting below has no effect. Requires a restart.

```bash
EENGINE_MCP_ENABLED=false
```

### Enable the Endpoint

**Setting:** `mcpEnabled`
**Admin UI:** Configuration > MCP
**Default:** `false`

The runtime switch. While it is off, every request to `/mcp` answers `404`.

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer $EE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"mcpEnabled": true}'
```

### Enable OAuth Sign-in for MCP Clients

**Setting:** `mcpOAuthEnabled`
**Admin UI:** Configuration > MCP
**Default:** `false`

Runs the built-in OAuth 2.1 authorization server used by MCP clients that cannot be configured with a static access token, such as web connectors. Requires `mcpEnabled` and a configured Service URL; without both, the OAuth discovery endpoints answer `404`.

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer $EE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"mcpOAuthEnabled": true, "serviceUrl": "https://emailengine.example.com"}'
```

## Environment Variable Reference

### Quick Reference Table

| Environment Variable         | Default                    | Description                                 |
| ---------------------------- | -------------------------- | ------------------------------------------- |
| `EENGINE_PORT`               | `3000`                     | HTTP port                                   |
| `EENGINE_HOST`               | `127.0.0.1`                | Bind address                                |
| `EENGINE_REDIS`              | `redis://127.0.0.1:6379/8` | Redis URL                                   |
| `EENGINE_SECRET`             | None                       | Encryption secret (required for production) |
| `EENGINE_WORKERS`            | `4`                        | Account worker threads (IMAP, Gmail, Graph) |
| `EENGINE_WORKERS_API`        | `1`                        | API/HTTP worker threads (Linux + reuseport) |
| `EENGINE_LOG_LEVEL`          | `trace`                    | Log level                                   |
| `EENGINE_TIMEOUT`            | `10000`                    | Command timeout (ms)                        |
| `EENGINE_FETCH_BATCH_SIZE`   | `1000`                     | Messages per sync batch                     |
| `EENGINE_PREPARED_LICENSE`   | None                       | License key                                 |
| `EENGINE_SETTINGS`           | None                       | Pre-configured settings JSON                |
| `EENGINE_PREPARED_TOKEN`     | None                       | Exported token hash (not raw token)         |
| `EENGINE_PREPARED_PASSWORD`  | None                       | Password hash (not plain password)          |
| `EENGINE_REQUIRE_API_AUTH`   | `true`                     | Require API authentication                  |
| `EENGINE_CORS_ORIGIN`        | None                       | CORS allowed origin                         |
| `EENGINE_SMTP_ENABLED`       | `false`                    | Enable SMTP proxy                           |
| `EENGINE_IMAP_PROXY_ENABLED` | `false`                    | Enable IMAP proxy                           |
| `EENGINE_MCP_ENABLED`        | `true`                     | Register the MCP endpoint routes            |

## Configuration File Example

### Complete config.toml

```toml
# EmailEngine Configuration File
# Place in working directory or specify with --config

[service]
# Encryption secret - required for production
# Generate with: openssl rand -hex 32
secret = "your-64-character-hex-secret-here"

# Command timeout in milliseconds
commandTimeout = 10000

# Messages per sync batch
fetchBatchSize = 1000

[api]
port = 3000
host = "127.0.0.1"

[dbs]
redis = "redis://localhost:6379/8"

[workers]
imap = 4
webhooks = 1
submit = 1

[log]
level = "info"
# raw = true  # Enable for debugging only
```

:::tip Using TOML Config
TOML is the native configuration format for EmailEngine. Use `--config=/path/to/config.toml` to specify a custom config file location.
:::

## Priority Order

When the same setting is configured in multiple ways:

1. **Environment variables** (highest priority)
2. **Command-line arguments**
3. **Configuration file** (lowest priority)

Example:

```bash
# Config file: api.port = 3000
# Command line: --api.port=4000
# Environment: EENGINE_PORT=5000

# Result: Port 5000 (environment wins)
```

## See Also

- [Environment Variables Guide](/docs/configuration/environment-variables)
- [Redis Configuration](/docs/configuration/redis)
- [Security Best Practices](/docs/deployment/security)
- [Docker Deployment](/docs/deployment)
