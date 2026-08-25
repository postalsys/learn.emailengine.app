---
title: Environment Variables Reference
description: Complete reference of all environment variables for configuring EmailEngine
sidebar_position: 2
---

# Environment Variables Reference

Complete reference for all EmailEngine environment variables. These settings are loaded at application startup and require a restart to take effect.

:::info .env File Support
EmailEngine automatically loads environment variables from a `.env` file located in the current working directory. This is the recommended way to configure EmailEngine as it ensures variables persist across restarts.

```bash
# Create .env file
echo "EENGINE_REDIS=redis://localhost:6379" > .env
echo "EENGINE_SECRET=$(openssl rand -hex 32)" >> .env

# Start EmailEngine (will load .env automatically)
emailengine
```
:::

:::tip Command-Line Alternative
Every environment variable can also be set via command-line arguments using the format `--section.key=value`. For example, `EENGINE_HOST=0.0.0.0` can be set as `--api.host=0.0.0.0`. [See CLI reference →](/docs/configuration/cli)
:::

## Quick Start

Minimal production configuration:

**Using environment variables:**
```bash
EENGINE_REDIS=redis://localhost:6379
EENGINE_HOST=0.0.0.0
EENGINE_PORT=3000
```

**Using command-line arguments:**
```bash
emailengine \
  --dbs.redis="redis://localhost:6379" \
  --api.host="0.0.0.0" \
  --api.port=3000
```

[Complete CLI reference →](/docs/configuration/cli)

## Loading Values From Files

Most variables can be provided as a file path instead of a literal value by appending `_FILE` to the variable name. EmailEngine reads the file at startup and uses its contents as the value. This is the usual way to pass credentials in Docker Swarm and Kubernetes deployments, where secrets are mounted as files rather than set in the environment.

```bash
EENGINE_SECRET_FILE=/run/secrets/ee_encryption_key
EENGINE_REDIS_FILE=/run/secrets/redis_url
```

- A trailing newline is stripped, so a file written with `echo "value" > secret.txt` works as expected. Avoid other whitespace around the value.
- If both `KEY` and `KEY_FILE` are set, `KEY` is used and the file is ignored
- Boolean variables work the same way, with the file containing `true` or `false`
- A file that cannot be read is logged as an error and resolves to an empty value. EmailEngine still starts, so check the logs if a setting seems to be missing.

A few non-secret variables are read straight from the environment and have no `_FILE` counterpart: `EENGINE_REDIS_PREFIX`, `EENGINE_FETCH_TIMEOUT`, `EENGINE_EXPORT_PATH`, `EENGINE_EXPORT_MAX_AGE`, `EENGINE_HTTP_PROXY_ENABLED`, `EENGINE_HTTP_PROXY_URL`, `EENGINE_TLS_MIN_VERSION`, `EENGINE_TLS_MIN_DH_SIZE`, and `EENGINE_TLS_CIPHERS`.

## Server & Connection

Configure HTTP server and connection settings.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `EENGINE_HOST` | string | `127.0.0.1` | HTTP server bind address | `0.0.0.0` |
| `EENGINE_PORT` | number | `3000` | HTTP server port | `8080` |
| `PORT` | number | `3000` | Alternative to EENGINE_PORT (used by some platforms) | `8080` |
| `EENGINE_TIMEOUT` | number | `10000` | HTTP request timeout (ms) | `30000` |
| `EENGINE_API_PROXY` | boolean | `false` | Trust reverse proxy headers (X-Forwarded-For) for client IP | `true` |
| `EENGINE_API_PROXY_ADDRESSES` | string | none | Comma-separated IPs/CIDRs of the proxies allowed to set X-Forwarded-For. Required for IP allowlists to be trustworthy, see [Reverse Proxy Mode](/docs/reference/configuration-options#reverse-proxy-mode) | `10.0.0.0/8` |

[Access token management →](/docs/api-reference/access-tokens)

**Examples:**

**Public deployment:**
```bash
EENGINE_HOST=0.0.0.0
EENGINE_PORT=3000
```

**Behind reverse proxy (Nginx, Apache, etc):**
```bash
EENGINE_HOST=127.0.0.1
EENGINE_PORT=3000
EENGINE_API_PROXY=true  # Trust X-Forwarded-For headers for client IP
EENGINE_API_PROXY_ADDRESSES=10.0.0.0/8  # ...but only when the request comes from these peers
```

## Redis

Redis database connection and configuration.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `EENGINE_REDIS` | string | `redis://127.0.0.1:6379/8` | Redis connection URL (primary) | `redis://user:pass@redis.example.com:6379/0` |
| `REDIS_URL` | string | `redis://127.0.0.1:6379/8` | Redis connection URL (fallback if EENGINE_REDIS not set) | `redis://localhost:6379` |
| `EENGINE_REDIS_PREFIX` | string | none | Optional key prefix for Redis keys | `{ee-prod}` |

**Connection URL Format:**
```
redis://[username:password@]host[:port][/database]
rediss://...  (with TLS)
```

**Examples:**

**Basic connection:**
```bash
EENGINE_REDIS=redis://localhost:6379
```

**With authentication:**
```bash
EENGINE_REDIS=redis://username:password@redis.example.com:6379
```

**With TLS:**
```bash
EENGINE_REDIS=rediss://redis.example.com:6380
```

**With database selection:**
```bash
EENGINE_REDIS=redis://localhost:6379/8
```

**Custom Redis key prefix:**
```bash
EENGINE_REDIS_PREFIX="{emailengine-prod}"
```

[Detailed Redis configuration →](./redis.md)

## Email Protocol Settings

Email protocol timeouts and limits.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `EENGINE_MAX_SIZE` | bytes | `5242880` | Max attachment size (5 MB) | `10485760` |
| `EENGINE_MAX_BODY_SIZE` | bytes | `52428800` | Max POST body size for message uploads (50 MB) | `104857600` |
| `EENGINE_MAX_SMTP_MESSAGE_SIZE` | bytes | `26214400` | Max message size for SMTP submission (25 MB) | `52428800` |
| `EENGINE_MAX_PAYLOAD_TIMEOUT` | ms | `10000` | Payload reception timeout for message uploads | `30000` |
| `EENGINE_TIMEOUT` | ms | `10000` | General timeout for operations | `30000` |
| `EENGINE_FETCH_TIMEOUT` | ms | `90000` | Timeout for HTTP fetch operations (90 seconds) | `120000` |
| `EENGINE_FETCH_BATCH_SIZE` | number | `1000` | Messages per batch during synchronization | `500` |
| `EENGINE_IMAP_SOCKET_TIMEOUT` | ms | none | Custom socket timeout for IMAP connections | `60000` |
| `EENGINE_CONNECTION_SETUP_DELAY` | ms | `0` | Delay before setting up account connections | `5000` |
| `EENGINE_CHUNK_SIZE` | bytes | `1000000` | Download chunk size for streaming attachments (1 MB) | `5000000` |
| `EENGINE_MAX_IMAP_AUTH_FAILURE_TIME` | ms | `259200000` | Stop retrying an account after this long of continuous authentication failures (3 days) | `86400000` |

**Examples:**

**High attachment limit:**
```bash
EENGINE_MAX_SIZE=20971520  # 20 MB
```

**Extended timeouts for slow servers:**
```bash
EENGINE_TIMEOUT=30000      # 30 seconds
EENGINE_FETCH_TIMEOUT=60000  # 60 seconds
```

**Delay connection setup on startup (useful for high account count):**
```bash
EENGINE_CONNECTION_SETUP_DELAY=10000  # 10 seconds
```

## Worker Threads

Control worker thread configuration for processing workload.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `EENGINE_WORKERS` | number | `4` | Account worker thread count - syncs IMAP, Gmail API, and Outlook (Microsoft Graph) accounts | `8` |
| `EENGINE_WORKERS_API` | number | `1` | API/HTTP worker threads. Values above `1` require `SO_REUSEPORT` (Linux with Node.js 23.1+); other platforms fall back to a single worker | `4` |
| `EENGINE_WORKERS_SUBMIT` | number | `1` | Worker threads for email submission | `2` |
| `EENGINE_WORKERS_WEBHOOKS` | number | `1` | Worker threads for webhook delivery | `2` |
| `EENGINE_WORKERS_EXPORT` | number | `1` | Worker threads for bulk message exports | `2` |

**Examples:**

**High-performance setup:**
```bash
EENGINE_WORKERS=8
EENGINE_WORKERS_SUBMIT=4
EENGINE_WORKERS_WEBHOOKS=4
EENGINE_WORKERS_EXPORT=2
```

**Resource-constrained environment:**
```bash
EENGINE_WORKERS=2
EENGINE_WORKERS_SUBMIT=1
EENGINE_WORKERS_WEBHOOKS=1
EENGINE_WORKERS_EXPORT=1
```

## Queue Management

Configure job queue retention, cleanup, and concurrency.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `EENGINE_QUEUE_REMOVE_AFTER` | number | `0` | Number of completed jobs to keep in queue (0 = remove immediately) | `5000` |
| `EENGINE_QUEUE_KEEP_FAILED` | number | `500` | Failed entries retained per queue, regardless of the setting above | `2000` |
| `EENGINE_QUEUE_KEEP_FAILED_AGE` | seconds | `604800` | How long failed entries are retained (7 days) | `259200` |
| `EENGINE_SUBMIT_QC` | number | `1` | Concurrency for email submission queue | `4` |
| `EENGINE_NOTIFY_QC` | number | `1` | Concurrency for notification/webhook queue | `4` |
| `EENGINE_EXPORT_QC` | number | `1` | Concurrency for export queue | `2` |
| `EENGINE_SUBMIT_DELAY` | ms | none | Delay between email submissions | `1000` |

**Examples:**

**Keep job history (retain last 1000 completed jobs):**
```bash
EENGINE_QUEUE_REMOVE_AFTER=1000
```

**Higher queue concurrency:**
```bash
EENGINE_SUBMIT_QC=4
EENGINE_NOTIFY_QC=4
EENGINE_EXPORT_QC=2
```

**Rate limit email submissions:**
```bash
EENGINE_SUBMIT_DELAY=1000  # 1 second between submissions
```

## Webhook Delivery

Configure how webhook deliveries are made. Webhook URLs, events, and custom routes are configured in the admin interface, not here.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `EENGINE_WEBHOOK_TIMEOUT` | ms | `30000` | Wall-clock timeout for a single delivery attempt | `10000` |
| `EENGINE_WEBHOOK_EGRESS_POLICY` | string | `link-local` | Which destinations deliveries may reach: `link-local`, `private`, or `off` | `private` |

**Egress policy values:**

| Value | Blocks | Use when |
|-------|--------|----------|
| `link-local` (default) | `169.254.0.0/16` and `fe80::/10`, where cloud instance metadata services live, plus the AWS IPv6 and Alibaba metadata addresses | Default. Webhook receivers on your private network keep working |
| `private` | The above plus RFC1918, loopback, CGNAT, and unique-local addresses | Receivers are all on the public internet |
| `off` | Nothing | You need the previous behavior, including following redirects |

Any policy other than `off` also stops redirects being followed, since a permitted host could otherwise redirect to a blocked one. See [Blocked destinations and redirects](/docs/webhooks/overview#blocked-destinations-and-redirects).

## Export Configuration

Configure bulk message export behavior.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `EENGINE_EXPORT_PATH` | string | OS temp dir | Directory for export files | `/data/exports` |
| `EENGINE_EXPORT_MAX_AGE` | ms | `86400000` | Export file retention time (24 hours) | `172800000` |
| `EENGINE_EXPORT_TIMEOUT` | duration | `5m` | Timeout for individual export operations | `10m` |

**Examples:**

**Custom export storage:**
```bash
EENGINE_EXPORT_PATH=/data/emailengine/exports
EENGINE_EXPORT_MAX_AGE=172800000  # 48 hours
EENGINE_EXPORT_TIMEOUT=10m
```

[Exporting Messages guide -->](/docs/receiving/exporting)

## IMAP Proxy Server

Enable and configure the built-in IMAP proxy server feature.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `EENGINE_IMAP_PROXY_ENABLED` | boolean | `false` | Enable IMAP proxy server | `true` |
| `EENGINE_IMAP_PROXY_HOST` | string | `127.0.0.1` | IMAP proxy bind address | `0.0.0.0` |
| `EENGINE_IMAP_PROXY_PORT` | number | `2993` | IMAP proxy server port | `993` |
| `EENGINE_IMAP_PROXY_SECRET` | string | none | IMAP proxy authentication password. If not set, API tokens with `imap-proxy` scope can be used | `your-secret-key` |
| `EENGINE_IMAP_PROXY_PROXY` | boolean | `false` | Enable PROXY protocol for IMAP proxy server | `true` |

**Examples:**

**Enable IMAP proxy with shared secret:**
```bash
EENGINE_IMAP_PROXY_ENABLED=true
EENGINE_IMAP_PROXY_HOST=0.0.0.0
EENGINE_IMAP_PROXY_PORT=2993
EENGINE_IMAP_PROXY_SECRET=my-secure-secret-key
```

**Enable IMAP proxy with API token authentication:**
```bash
# No secret set - use API tokens with "imap-proxy" scope for authentication
EENGINE_IMAP_PROXY_ENABLED=true
EENGINE_IMAP_PROXY_HOST=0.0.0.0
EENGINE_IMAP_PROXY_PORT=2993
```

## SMTP Proxy Server

Enable and configure the built-in SMTP proxy server feature.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `EENGINE_SMTP_ENABLED` | boolean | `false` | Enable SMTP proxy server | `true` |
| `EENGINE_SMTP_HOST` | string | `127.0.0.1` | SMTP server bind address | `0.0.0.0` |
| `EENGINE_SMTP_PORT` | number | `2525` | SMTP server port | `587` |
| `EENGINE_SMTP_SECRET` | string | none | SMTP authentication password. If not set, API tokens with `smtp` scope can be used | `your-secret-key` |
| `EENGINE_SMTP_PROXY` | boolean | `false` | Enable PROXY protocol for SMTP proxy server | `true` |
| `EENGINE_MAX_SMTP_MESSAGE_SIZE` | bytes | `26214400` | Max message size the SMTP proxy accepts (25 MB) | `52428800` |

**Examples:**

**Enable SMTP proxy with shared secret:**
```bash
EENGINE_SMTP_ENABLED=true
EENGINE_SMTP_HOST=0.0.0.0
EENGINE_SMTP_PORT=2525
EENGINE_SMTP_SECRET=my-secure-secret-key
```

**Enable SMTP proxy with API token authentication:**
```bash
# No secret set - use API tokens with "smtp" scope for authentication
EENGINE_SMTP_ENABLED=true
EENGINE_SMTP_HOST=0.0.0.0
EENGINE_SMTP_PORT=2525
```

## TLS Configuration

Configure TLS/SSL settings for secure connections.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `EENGINE_TLS_MIN_VERSION` | string | `TLSv1` | Minimum TLS version | `TLSv1.2` |
| `EENGINE_TLS_MIN_DH_SIZE` | number | `1024` | Minimum Diffie-Hellman key size | `2048` |
| `EENGINE_TLS_CIPHERS` | string | `DEFAULT@SECLEVEL=0` | TLS cipher suite list | `TLS_AES_256_GCM_SHA384` |
| `EENGINE_API_TLS` | boolean | `false` | Enable TLS for the API server | `true` |

**Examples:**

**Enforce TLS 1.2 minimum:**
```bash
EENGINE_TLS_MIN_VERSION=TLSv1.2
```

**Stronger DH parameters:**
```bash
EENGINE_TLS_MIN_DH_SIZE=2048
```

**Custom cipher suite:**
```bash
EENGINE_TLS_CIPHERS="TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256"
```

The variables above apply to connections EmailEngine makes *to* mail servers. The listeners EmailEngine runs itself take their certificates from a separate, per-server prefix.

### Certificates for EmailEngine's Own Listeners

Each server that can terminate TLS reads its certificate material from its own prefix:

| Server | Prefix | Enabled by |
|--------|--------|------------|
| API and admin interface | `EENGINE_API_TLS_` | `EENGINE_API_TLS=true` |
| [SMTP server](/docs/sending/smtp-interface) | `EENGINE_SMTP_TLS_` | The `smtpServerTLSEnabled` setting |
| [IMAP proxy](/docs/accounts/proxying-connections) | `EENGINE_IMAPPROXY_TLS_` | The `imapProxyServerTLSEnabled` setting |

All three accept the same suffixes:

| Suffix | Purpose |
|--------|---------|
| `KEY` | Private key, PEM encoded |
| `CERT` | Certificate chain, PEM encoded |
| `CA` | Additional CA certificates |
| `PASSPHRASE` | Passphrase for an encrypted private key |
| `DHPARAM` | Diffie-Hellman parameters |
| `CIPHERS` | Cipher suite list |
| `ECDH_CURVE` | Named curve for ECDH |
| `MIN_VERSION` / `MAX_VERSION` | TLS version bounds |
| `REJECT_UNAUTHORIZED` | Reject clients presenting an untrusted certificate |
| `REQUEST_CERT` | Ask the client for a certificate |

```bash
EENGINE_API_TLS=true
EENGINE_API_TLS_KEY_FILE=/etc/emailengine/tls/api.key
EENGINE_API_TLS_CERT_FILE=/etc/emailengine/tls/api.crt
EENGINE_API_TLS_MIN_VERSION=TLSv1.2
```

The `_FILE` suffix described under [Loading Values From Files](#loading-values-from-files) works here too, which is the usual way to mount a key into a container without putting it in the environment.

:::note A managed certificate wins over these variables
When the SMTP server starts with TLS enabled, EmailEngine looks for a certificate matching the hostname in `serviceUrl` and uses it if one is valid. That happens after these variables are read, so a certificate EmailEngine manages takes precedence over `EENGINE_SMTP_TLS_CERT`.
:::

Most deployments do not need any of this, because TLS is terminated at a reverse proxy instead. See [Nginx Reverse Proxy](/docs/deployment/nginx-proxy).

## Security & Access Control

Security settings and access restrictions.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `EENGINE_SECRET` | string | none | **Required for production.** Master encryption key for all credentials stored in Redis (AES-256-GCM). | `$(openssl rand -hex 32)` |
| `EENGINE_ADMIN_ACCESS_ADDRESSES` | string | all | Comma-separated list of IP addresses allowed to access admin interface | `192.168.1.0/24,10.0.0.1` |
| `EENGINE_REQUIRE_API_AUTH` | boolean | `true` | Require API authentication tokens | `false` |
| `EENGINE_ENABLE_OAUTH_TOKENS_API` | boolean | `false` | Allow retrieving raw OAuth tokens via API | `true` |
| `EENGINE_DISABLE_SETUP_WARNINGS` | boolean | `false` | Disable admin password setup warnings | `true` |
| `EENGINE_TOKEN_LOG_ENTRIES` | number | `1000` | Requests retained in a token's audit log | `5000` |
| `EENGINE_TOKEN_LOG_AGE` | seconds | `604800` | How long a token's audit log entries are kept (7 days) | `2592000` |

### Credential Encryption (EENGINE_SECRET)

:::danger Critical for Production
Without `EENGINE_SECRET`, all account passwords, OAuth2 tokens, and application secrets are stored **unencrypted** in Redis. Always configure this for production deployments.
:::

**What gets encrypted:**
- IMAP/SMTP passwords
- OAuth2 access and refresh tokens
- OAuth2 application client secrets
- Service account private keys

**Set encryption secret:**
```bash
# Generate a secure 256-bit secret
openssl rand -hex 32

# Add to .env file:
EENGINE_SECRET=generated-value-here
```

:::warning Secret Recovery
If you lose `EENGINE_SECRET`, encrypted credentials cannot be recovered. Back up this secret securely and separately from your Redis data.
:::

[Credential Security FAQ](/docs/support/security-faq) | [Encryption Guide](/docs/advanced/encryption)

**Restrict admin access to specific IPs:**
```bash
EENGINE_ADMIN_ACCESS_ADDRESSES="192.168.1.0/24,10.0.0.1"
```

**Development mode (disable API auth):**
```bash
# WARNING: Never use in production
EENGINE_REQUIRE_API_AUTH=false
```

**Token audit log retention:**

The audit log is off until you enable it under **Configuration** > **Security**. These two variables bound what it keeps once it is on, per token, whichever limit is reached first. Both live in Redis, so raising them costs memory.

```bash
EENGINE_TOKEN_LOG_ENTRIES=5000
EENGINE_TOKEN_LOG_AGE=2592000
```

## Single Sign-On (SSO)

Enable single sign-on for the EmailEngine admin interface, either through a generic OpenID Connect provider or through the dedicated Okta integration. See [Single Sign-On](/docs/deployment/security#single-sign-on-sso) in the security guide for the setup procedure.

### OpenID Connect (OIDC)

All three of `OIDC_ISSUER`, `OIDC_CLIENT_ID`, and `OIDC_CLIENT_SECRET` must be set to activate OIDC SSO.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `OIDC_ISSUER` | string | none | Issuer base URL of the OpenID Connect provider. Must match the `issuer` value in the provider's discovery document. | `https://keycloak.example.com/realms/main` |
| `OIDC_CLIENT_ID` | string | none | Client ID of the registered application | `emailengine-admin` |
| `OIDC_CLIENT_SECRET` | string | none | Client secret of the registered application | `your-client-secret` |
| `OIDC_PROVIDER_NAME` | string | `SSO` | Label shown on the sign-in button | `Keycloak` |
| `OIDC_SCOPES` | string | `openid profile email` | Scopes to request, space or comma separated. `openid` is always included. | `openid profile email groups` |
| `OIDC_ALLOWED_USERS` | string | none | Comma-separated allow-list of emails and/or `@domain` entries. Case-insensitive. Empty means any authenticated user is allowed. | `admin@example.com,@example.com` |
| `OIDC_ALLOWED_GROUPS` | string | none | Comma-separated allow-list of group names, matched against the groups claim. A user is allowed if they match either `OIDC_ALLOWED_USERS` or `OIDC_ALLOWED_GROUPS`. | `emailengine-admins` |
| `OIDC_GROUPS_CLAIM` | string | `groups` | Userinfo claim that carries group membership. Dotted paths are supported. | `realm_access.roles` |
| `OIDC_FORCED` | boolean | `false` | SSO-only login. The login page redirects straight to the identity provider and password/passkey sign-in is disabled. | `true` |
| `OIDC_LOGOUT` | boolean | `false` | Also end the identity provider session on logout (RP-initiated logout) | `true` |
| `OIDC_POST_LOGOUT_REDIRECT_URI` | string | none | Where the identity provider returns after logout. Must be registered as a post-logout redirect URI at the provider. | `https://emailengine.example.com/admin/login?loggedout=1` |

**Setup:**
```bash
OIDC_ISSUER=https://keycloak.example.com/realms/main
OIDC_CLIENT_ID=your-client-id
OIDC_CLIENT_SECRET=your-client-secret
OIDC_PROVIDER_NAME=Keycloak
```

**Identity provider application configuration:**
- Application type: confidential web application using the authorization code flow
- Sign-in redirect URI: `{serviceUrl}/admin/login/oidc` (where `serviceUrl` is the public URL configured in EmailEngine settings)
- The discovery document must be available at `<issuer>/.well-known/openid-configuration`
- Requires restart after configuration changes

`OIDC_CLIENT_SECRET_FILE` can be used instead of `OIDC_CLIENT_SECRET` to read the client secret from a file. See [Loading Values From Files](#loading-values-from-files).

### Okta

All three variables must be set to activate Okta SSO.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `OKTA_OAUTH2_ISSUER` | string | none | Okta OAuth2 issuer URL | `https://your-org.okta.com/oauth2/default` |
| `OKTA_OAUTH2_CLIENT_ID` | string | none | Okta application client ID | `0oa1bcdef2ghijk3lmn4` |
| `OKTA_OAUTH2_CLIENT_SECRET` | string | none | Okta application client secret | `your-client-secret` |

When configured, a "Sign in with Okta" button appears on the admin login page (`/admin/login`).

**Setup:**
```bash
OKTA_OAUTH2_ISSUER=https://your-org.okta.com/oauth2/default
OKTA_OAUTH2_CLIENT_ID=your-client-id
OKTA_OAUTH2_CLIENT_SECRET=your-client-secret
```

**Okta application configuration:**
- Application type: Web
- Sign-in redirect URI: `{serviceUrl}/admin/login/okta` (where `serviceUrl` is the public URL configured in EmailEngine settings)
- Requires restart after configuration changes

## Advanced Settings

Advanced configuration options for debugging and performance tuning.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `EENGINE_LOG_RAW` | boolean | `false` | Log raw IMAP protocol traffic (debug only) | `true` |
| `EENGINE_DISABLE_COMPRESSION` | boolean | `false` | Disable IMAP COMPRESS extension | `true` |
| `EENGINE_DISABLE_MESSAGE_BROWSER` | boolean | `false` | Disable web-based message browser | `true` |
| `EENGINE_CORS_ORIGIN` | string | none | CORS allowed origins (whitespace separated) | `https://app.example.com` |
| `EENGINE_CORS_MAX_AGE` | number | `60` | CORS preflight cache duration in seconds | `3600` |
| `EENGINE_DOCUMENT_STORE_ENABLED` | boolean | `false` | Enable the deprecated Document Store (Elasticsearch) feature gate | `true` |
| `EENGINE_MCP_ENABLED` | boolean | `true` | Register the [MCP endpoint](/docs/mcp) routes. Registration alone serves nothing: the `mcpEnabled` setting is the runtime switch | `false` |
| `EENGINE_DISABLE_THREAD_COLLAPSE` | boolean | `false` | Stop web-safe HTML from folding quoted thread history into a collapsible block | `true` |
| `EENGINE_BEACON_DISABLED` | boolean | `false` | Disable the anonymized feature beacon that rides on the license validation request | `true` |
| `EENGINE_UPDATE_CHECK_DISABLED` | boolean | `false` | Disable the update check against the GitHub releases API | `true` |

**Examples:**

**Enable protocol debugging:**
```bash
EENGINE_LOG_RAW=true
EENGINE_LOG_LEVEL=trace
```

**Enable CORS for API:**
```bash
EENGINE_CORS_ORIGIN="https://app.example.com https://admin.example.com"
```

**Disable IMAP compression (for debugging):**
```bash
EENGINE_DISABLE_COMPRESSION=true
```

**Enable the deprecated Document Store:**
```bash
EENGINE_DOCUMENT_STORE_ENABLED=true
```

**Render quoted thread history inline:**
```bash
EENGINE_DISABLE_THREAD_COLLAPSE=true
```

**Remove the MCP endpoint from an instance entirely:**
```bash
EENGINE_MCP_ENABLED=false
```

**Run without background network calls (strict egress or air-gapped):**
```bash
EENGINE_UPDATE_CHECK_DISABLED=true
EENGINE_BEACON_DISABLED=true
```

Since EmailEngine v2.75.0, [web-safe HTML](/docs/receiving/web-safe-html) wraps the quoted tail of a reply in a collapsible block so a message renders as what the sender wrote. Set this to restore the previous output shape, in which the whole thread is rendered inline.

The Document Store (Elasticsearch) feature is deprecated and disabled by default since EmailEngine v2.71.0, and it is **removed from EmailEngine releases starting October 1, 2026**. This startup gate must be turned on before EmailEngine will run the document indexing worker or register the Document Store API and admin endpoints (`/v1/chat/{account}`, `/v1/unified/search`, and the `Configuration > Document Store` page). While the gate is off, those endpoints return `404`, even if the runtime "Document Store" setting is still enabled. The equivalent config-file setting is `[documentStore] enabled = true` (CLI flag `--documentStore.enabled=true`).

If you depend on the Document Store, plan the migration now. Staying on the last release that still ships it means running an EmailEngine that no longer receives security updates.

`EENGINE_MCP_ENABLED` is a deployment gate rather than an on switch: it defaults to `true`, and setting it to `false` removes the `/mcp` routes and the MCP configuration page from the instance so the surface does not exist at all. The endpoint itself stays off until an admin enables the `mcpEnabled` setting, which starts out disabled. The equivalent config-file setting is `[mcp] enabled = false` (CLI flag `--mcp.enabled=false`), and a change requires a restart. See [MCP for AI Agents](/docs/mcp).

Since EmailEngine v2.76.0, `EENGINE_UPDATE_CHECK_DISABLED=true` disables the check against `api.github.com` that powers the "update available" notice in the admin dashboard. The check runs once at startup, sends nothing beyond a standard User-Agent header, and fails silently without network access, but it is the only background network call that is not tied to a subscription license. Subscription licenses additionally validate daily against `postalsys.com`, carrying an [anonymized feature beacon](/docs/deployment/compliance#no-developer-access) that `EENGINE_BEACON_DISABLED=true` disables; perpetual licenses are verified offline and never contact the license server at all. With the update check disabled, a perpetual-license instance makes no background network calls whatsoever.

## HTTP Proxy

Route outbound HTTP/HTTPS requests (webhooks, OAuth2 token requests, API calls) through an HTTP or SOCKS proxy.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `EENGINE_HTTP_PROXY_ENABLED` | boolean | `false` | Enable HTTP proxy for outbound requests | `true` |
| `EENGINE_HTTP_PROXY_URL` | string | none | Proxy server URL (HTTP, HTTPS, or SOCKS) | `socks5://proxy.example.com:1080` |

:::info Settings Override
These environment variables override the equivalent API settings (`httpProxyEnabled` and `httpProxyUrl` via `POST /v1/settings`). When both are set, environment variables take precedence.
:::

**Examples:**

**Route through HTTP proxy:**
```bash
EENGINE_HTTP_PROXY_ENABLED=true
EENGINE_HTTP_PROXY_URL=http://proxy.example.com:8080
```

**Route through SOCKS5 proxy:**
```bash
EENGINE_HTTP_PROXY_ENABLED=true
EENGINE_HTTP_PROXY_URL=socks5://proxy.example.com:1080
```

## Logging & Monitoring

Logging configuration and error tracking.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `EENGINE_LOG_LEVEL` | string | `trace` | Log level (trace, debug, info, warn, error, fatal) | `info` |
| `SENTRY_DSN` | string | none | Sentry DSN for error reporting. When set, it pins the DSN and overrides the runtime Sentry toggle | `https://key@sentry.example.com/1` |
| `NODE_ENV` | string | `production` | Node.js environment | `development` |

**Log Levels:**
- `trace` - Very detailed, includes all protocol messages
- `debug` - Detailed operational information
- `info` - General operational messages
- `warn` - Warning messages
- `error` - Error messages only
- `fatal` - Fatal errors only

**Examples:**

**Development/debugging:**
```bash
NODE_ENV=development
EENGINE_LOG_LEVEL=trace
```

**Production:**
```bash
NODE_ENV=production
EENGINE_LOG_LEVEL=info
```

**Enable error tracking:**
```bash
# Pin a Sentry DSN (overrides the runtime toggle)
SENTRY_DSN=https://public-key@sentry.example.com/1
```

Error reporting uses [Sentry](https://sentry.io/) (Bugsnag was removed in EmailEngine v2.70.0). You can also enable it at runtime from Configuration > Logging (settings `sentryEnabled` and `sentryDsn`, applied within a minute and without a restart). If you enable reporting but leave the DSN empty, reports go to `sentry.emailengine.dev`, the Sentry instance run by the EmailEngine developers. Set your own DSN to keep reports in-house. Setting `SENTRY_DSN` here pins the DSN and takes precedence over the runtime toggle.

Reports carry stack traces, account IDs, and error details. They never include credentials or message content.

:::info Trial licenses report errors by default
Activating a trial license turns error reporting on, so that problems with evaluation instances reach the EmailEngine developers through their shared Sentry instance. This only happens while you have never set `sentryEnabled` yourself. Activating a full license or removing the license turns it back off, and any explicit write to `sentryEnabled` - through the Configuration > Logging form, `POST /v1/settings`, or `EENGINE_SETTINGS` - makes your choice permanent. The Configuration > Logging page tells you when reporting is in this trial-managed state.
:::

[Monitoring and logging →](../advanced/monitoring)

## Prepared Configuration

Pre-configured settings for automated deployments.

| Variable | Type | Description | Example |
|----------|------|-------------|---------|
| `EENGINE_SETTINGS` | JSON | Pre-configured runtime settings | See below |
| `EENGINE_PREPARED_TOKEN` | string | Exported token hash (from `emailengine tokens export`) | `hKJpZNlAMzAxZThjNTFh...` |
| `EENGINE_PREPARED_PASSWORD` | string | Password hash (from `emailengine password --hash`) | `JHBia2RmMi1zaGE1MTIk...` |
| `EENGINE_PREPARED_LICENSE` | string | Pre-configured license key | `license-key-string` |

**Examples:**

**Prepared settings:**
```bash
EENGINE_SETTINGS='{
  "webhooks": "https://your-app.com/webhook",
  "webhookEvents": ["messageNew", "messageSent"]
}'
```

**Docker Compose (multiline):**
```yaml
environment:
  EENGINE_SETTINGS: >
    {
      "webhooks": "https://your-app.com/webhook",
      "webhookEvents": [
        "messageNew",
        "messageDeleted",
        "messageSent"
      ]
    }
```

**Prepared token (requires exported hash, not the raw token):**
```bash
# 1. Generate token
TOKEN=$(emailengine tokens issue -d "API Token" -s "*")

# 2. Export token to get the hash
EXPORTED=$(emailengine tokens export -t $TOKEN)

# 3. Use the exported hash (not the raw token)
EENGINE_PREPARED_TOKEN=$EXPORTED
```

**Prepared password (requires password hash, not plain password):**
```bash
# Generate password hash
emailengine password -p "your-secure-password" --hash
# Output: JHBia2RmMi1zaGE1MTIkaTEwMDAwMCRhYmNkZWYx...

# Use the hash
EENGINE_PREPARED_PASSWORD=JHBia2RmMi1zaGE1MTIkaTEwMDAwMCRhYmNkZWYx...
```

**Prepared license:**
```bash
EENGINE_PREPARED_LICENSE=your-license-key-here
```

[Prepared configuration guide →](./prepared-settings/)

## Complete Examples

### Minimal Production

```bash
# Required
EENGINE_REDIS=redis://localhost:6379
EENGINE_HOST=0.0.0.0
EENGINE_PORT=3000

# Recommended
EENGINE_LOG_LEVEL=info
```

### High-Performance Production

```bash
# Server
EENGINE_HOST=0.0.0.0
EENGINE_PORT=3000

# Redis
EENGINE_REDIS=redis://redis-cluster:6379
EENGINE_REDIS_PREFIX={ee-prod}

# Performance
EENGINE_WORKERS=8
EENGINE_WORKERS_SUBMIT=4
EENGINE_WORKERS_WEBHOOKS=4

# Limits
EENGINE_MAX_SIZE=20971520  # 20 MB attachments
EENGINE_TIMEOUT=30000

# Queue
EENGINE_QUEUE_REMOVE_AFTER=5000

# TLS
EENGINE_TLS_MIN_VERSION=TLSv1.3
EENGINE_TLS_MIN_DH_SIZE=2048

# Logging
EENGINE_LOG_LEVEL=info
SENTRY_DSN=https://public-key@sentry.example.com/1
```

### Development Setup

```bash
# Server
EENGINE_HOST=127.0.0.1
EENGINE_PORT=3001

# Redis (separate DB for dev)
EENGINE_REDIS=redis://localhost:6379/8
EENGINE_REDIS_PREFIX={ee-dev}

# Debugging
NODE_ENV=development
EENGINE_LOG_LEVEL=trace

# Relaxed limits for testing
EENGINE_MAX_SIZE=104857600  # 100 MB
EENGINE_TIMEOUT=180000
```

### Docker Compose Example

```yaml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data

  emailengine:
    image: postalsys/emailengine:v2
    depends_on:
      - redis
    ports:
      - "3000:3000"
    environment:
      # Server
      - EENGINE_HOST=0.0.0.0
      - EENGINE_PORT=3000

      # Redis
      - EENGINE_REDIS=redis://redis:6379
      - EENGINE_REDIS_PREFIX={ee-prod}

      # Performance
      - EENGINE_WORKERS=4
      - EENGINE_WORKERS_SUBMIT=2
      - EENGINE_WORKERS_WEBHOOKS=2

      # Settings
      - EENGINE_SETTINGS=${EENGINE_SETTINGS}

      # Credentials (EENGINE_PREPARED_PASSWORD requires hash, not plain password)
      # Generate hash: emailengine password -p "your-password" --hash
      - EENGINE_PREPARED_PASSWORD=${ADMIN_PASSWORD_HASH}
      - EENGINE_PREPARED_LICENSE=${LICENSE_KEY}

      # Logging
      - EENGINE_LOG_LEVEL=info

volumes:
  redis-data:
```

### With Proxy Servers Enabled

```bash
# Server
EENGINE_HOST=0.0.0.0
EENGINE_PORT=3000

# Redis
EENGINE_REDIS=redis://localhost:6379

# IMAP Proxy
EENGINE_IMAP_PROXY_ENABLED=true
EENGINE_IMAP_PROXY_HOST=0.0.0.0
EENGINE_IMAP_PROXY_PORT=2993
EENGINE_IMAP_PROXY_SECRET=imap-proxy-secret

# SMTP Proxy
EENGINE_SMTP_ENABLED=true
EENGINE_SMTP_HOST=0.0.0.0
EENGINE_SMTP_PORT=2525
EENGINE_SMTP_SECRET=smtp-proxy-secret

# Logging
EENGINE_LOG_LEVEL=info
```

## Environment Variable to CLI Mapping

Common environment variables and their command-line equivalents:

| Environment Variable | CLI Argument | Description |
|---------------------|--------------|-------------|
| `EENGINE_REDIS` or `REDIS_URL` | `--dbs.redis` | Redis connection URL |
| `EENGINE_HOST` | `--api.host` | HTTP server bind address |
| `EENGINE_PORT` or `PORT` | `--api.port` | HTTP server port |
| `EENGINE_LOG_LEVEL` | `--log.level` | Log level |
| `EENGINE_SECRET` | `--service.secret` | Encryption secret |
| `EENGINE_WORKERS` | `--workers.imap` | Account worker count |
| `EENGINE_WORKERS_API` | `--workers.api` | API/HTTP worker count |
| `EENGINE_WORKERS_WEBHOOKS` | `--workers.webhooks` | Webhook worker count |
| `EENGINE_WORKERS_SUBMIT` | `--workers.submit` | Submission worker count |
| `EENGINE_WORKERS_EXPORT` | `--workers.export` | Export worker count |
| `EENGINE_MAX_SIZE` | `--api.maxSize` | Max attachment size |
| `EENGINE_TIMEOUT` | `--service.commandTimeout` | Command timeout |

**Pattern:** Most environment variables follow the pattern `EENGINE_*` → `--section.key`. To find the CLI equivalent, check the [wild-config](https://github.com/nodemailer/wild-config) documentation or use `--help`.

## See Also

- [CLI Reference](/docs/configuration/cli) - Command-line arguments as an alternative to environment variables
- [Redis Configuration](/docs/configuration/redis) - Detailed Redis setup and optimization
- [Prepared Settings](/docs/configuration/prepared-settings) - Automated deployment configuration
- [Access Tokens](/docs/api-reference/access-tokens) - API authentication setup
- [Monitoring](../advanced/monitoring) - Logging and monitoring setup
