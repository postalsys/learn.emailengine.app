---
title: Configuration Options Reference
description: Complete reference of all configuration options and settings
sidebar_position: 3
---

# Configuration Reference

EmailEngine is configured in two places. Startup configuration is read from the environment, the command line or a TOML file when the process starts, and changing it needs a restart. Runtime settings live in Redis, are written through `POST /v1/settings` or the admin interface, and take effect without one.

This page is the reference for the runtime settings, and for how the two halves fit together. Every startup variable is documented in [Environment Variables](/docs/configuration/environment-variables), which also gives the command-line argument and configuration-file key each one corresponds to.

## Configuration Types

### Startup Configuration

Loaded at startup. Requires a restart to apply changes.

- HTTP host and port, worker counts, Redis connection
- The encryption secret and the prepared license, token and password
- Log level and raw protocol logging
- TLS material for the listeners EmailEngine runs itself
- Feature gates such as the MCP routes and the deprecated Document Store

Some of these variables only seed a runtime setting the first time an instance starts, after which the stored setting is what counts. Each one says so where it is documented.

### Runtime Configuration

Stored in Redis. Changed through the settings API or the web interface without a restart.

- Service URL and the webhook target, events and payload contents
- Sending limits, tracking, and the local addresses and proxies used for mail connections
- The built-in SMTP server and IMAP proxy
- Locale, timezone and hosted-page branding
- OAuth2 applications, which are a separate API resource rather than settings keys

## Startup Configuration Reference

Startup variables are documented on one page, [Environment Variables](/docs/configuration/environment-variables). It is organized by topic:

| Topic | Section |
|-------|---------|
| Host, port and reverse proxy handling | [Server and Connection](/docs/configuration/environment-variables#server--connection), [Reverse Proxy Mode](/docs/configuration/environment-variables#reverse-proxy-mode), [Trusted Proxy Addresses](/docs/configuration/environment-variables#trusted-proxy-addresses) |
| Redis connection and key prefix | [Redis](/docs/configuration/environment-variables#redis) |
| Timeouts, batch sizes and the authentication-failure window | [Email Protocol Settings](/docs/configuration/environment-variables#email-protocol-settings), [Max IMAP Auth Failure Time](/docs/configuration/environment-variables#max-imap-auth-failure-time) |
| Worker counts and queue concurrency | [Worker Threads](/docs/configuration/environment-variables#worker-threads), [Queue Management](/docs/configuration/environment-variables#queue-management) |
| Webhook delivery timeout and egress policy | [Webhook Delivery](/docs/configuration/environment-variables#webhook-delivery) |
| The encryption secret, admin allowlist, API auth and token audit retention | [Security and Access Control](/docs/configuration/environment-variables#security--access-control) |
| TLS for the API, SMTP and IMAP proxy listeners | [TLS Configuration](/docs/configuration/environment-variables#tls-configuration) |
| Single sign-on through OIDC or Okta | [Single Sign-On](/docs/configuration/environment-variables#single-sign-on-sso) |
| CORS, raw logging, feature gates and the update check | [Advanced Settings](/docs/configuration/environment-variables#advanced-settings), [Cross-Origin Requests](/docs/configuration/environment-variables#cors-configuration) |
| Prepared settings, token, password and license | [Prepared Configuration](/docs/configuration/environment-variables#prepared-configuration) |
| Log level and Sentry error reporting | [Logging and Monitoring](/docs/configuration/environment-variables#logging--monitoring) |

Command-line and configuration-file equivalents are in the same place, under [Environment Variable to CLI Mapping](/docs/configuration/environment-variables#environment-variable-to-cli-mapping), and the full argument list is in the [CLI reference](/docs/configuration/cli#all-server-arguments).

## Reading and Writing Runtime Settings

`POST /v1/settings` takes an object of the keys to change and answers with the list of keys it wrote:

```bash
curl -X POST https://emailengine.example.com/v1/settings \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "webhooks": "https://your-app.com/webhooks",
    "webhooksEnabled": true,
    "webhookEvents": ["messageNew", "messageDeleted", "authenticationError"]
  }'
```

```json
{
  "updated": ["webhooks", "webhooksEnabled", "webhookEvents"]
}
```

Keys left out of the request keep their stored values. Reading them back is the reverse: `GET /v1/settings` returns only the keys you ask for, each named as a query parameter set to `true`.

```bash
curl "https://emailengine.example.com/v1/settings?webhooks=true&webhooksEnabled=true" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

A read does not return credentials. The user information in a proxy or webhook URL and the values of custom webhook headers come back as `******`, and the encrypted keys come back as booleans saying whether a value is stored. That matters for a read-modify-write round trip: `POST /v1/settings` skips a masked value when it matches the stored one and refuses it otherwise, so send the real credential or leave the key out.

The same values are edited under **Configuration** in the admin interface, and can be provisioned before the first start with the `EENGINE_SETTINGS` environment variable:

```bash
EENGINE_SETTINGS='{"serviceUrl":"https://emailengine.example.com","webhooks":"https://your-app.com/webhooks","webhooksEnabled":true,"webhookEvents":["*"]}'
```

`EENGINE_SETTINGS` is validated against the settings schema at startup. A value that fails validation stops EmailEngine from starting, and an unknown key is dropped and logged as an error, so a typo does not pass unnoticed. See [Prepared Settings](/docs/configuration/prepared-settings) for the full provisioning workflow.

## Runtime Settings Reference

Every key `POST /v1/settings` accepts, as listed in the [OpenAPI spec](/docs/api/post-v-1-settings). Values live in Redis; `GET /v1/settings?webhooks=true` reads them back by name. Type and default are what the source applies when the key is unset.

### Webhooks

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `webhooksEnabled` | boolean | off | Enable webhook delivery for all accounts |
| `webhooks` | string | unset | Target URL for webhook POST requests |
| `webhookEvents` | array of strings | unset (nothing is delivered) | Allowlist of event names; `["*"]` delivers every event |
| `webhooksCustomHeaders` | array of `{key, value}` | unset | Extra HTTP headers sent with every webhook |
| `notifyHeaders` | array of strings | unset | Email headers to include in webhook payloads |
| `notifyText` | boolean | `true` | Include message text in webhook payloads |
| `notifyTextSize` | integer | `2097152` | Maximum text size in webhook payloads, in bytes |
| `notifyWebSafeHtml` | boolean | off | Replace the HTML body with a [web-safe](/docs/receiving/web-safe-html) rendering |
| `notifyAttachments` | boolean | off | Include attachment content in webhook payloads |
| `notifyAttachmentSize` | integer | unset (no limit) | Skip attachments larger than this many bytes |
| `notifyCalendarEvents` | boolean | off | Include parsed calendar events in webhook payloads |
| `inboxNewOnly` | boolean | off | Send `messageNew` only for messages arriving in the Inbox |

`webhookEvents` is an allowlist with no default, so an instance with `webhooksEnabled` on and no event list delivers nothing. The `notify*` keys decide how large a payload gets: a `messageNew` body carries up to `notifyTextSize` bytes of message text, and attachment content is base64, so it arrives about a third larger than the file. Per-account webhook routes and the full event list are covered in [Webhooks](/docs/webhooks/overview); the delivery timeout and the egress policy are startup variables, under [Webhook Delivery](/docs/configuration/environment-variables#webhook-delivery).

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
    "notifyHeaders": ["List-Unsubscribe", "X-Mailer"]
  }'
```

### Service

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `serviceUrl` | string | unset | Public base URL of this instance. Only the origin is stored; a path is dropped |
| `notificationBaseUrl` | string | falls back to `serviceUrl` | Public callback URL for provider push notifications |
| `serviceSecret` | string | random, generated at first start | HMAC key for webhook signatures and signed links. A blank value is ignored rather than stored |
| `authServer` | string | unset | External authentication server that returns account credentials on demand |
| `tokenAuditLog` | boolean | off | Record every request each access token makes |
| `enableApiProxy` | boolean | on, unless `EENGINE_API_PROXY` says otherwise | Trust `X-Forwarded-*` headers from a reverse proxy |
| `locale` | `en`, `et`, `fr`, `de`, `pl`, `ja`, `nl` | `en` | Default UI language |
| `timezone` | string | unset | Default timezone for date display, as an IANA identifier |
| `pageBrandName` | string | unset | Brand name in page titles |
| `templateHeader` | string | unset | HTML injected at the top of hosted pages |
| `templateHtmlHead` | string | unset | HTML injected into the `<head>` of hosted pages |
| `scriptEnv` | string (JSON object) | `{}` | Values exposed to pre-processing scripts |
| `logs` | object `{all, maxLogLines}` | `{"all": false, "maxLogLines": 10000}` | Per-account protocol logging |
| `sentryEnabled` | boolean | off; on while a trial license is active and the operator has not set it | Report unhandled errors to Sentry. Ignored when `SENTRY_DSN` is set in the environment |
| `sentryDsn` | string | unset (the EmailEngine developers' instance) | Sentry DSN to report to |
| `mcpEnabled` | boolean | off | Serve the [MCP endpoint](/docs/mcp) at `/mcp` |
| `mcpOAuthEnabled` | boolean | off | Run the built-in OAuth 2.1 authorization server for MCP clients |

#### Service URL

`serviceUrl` is the public base URL every generated link is built from: OAuth2 redirect URIs, hosted authentication forms, tracking and unsubscribe links, and the MCP OAuth discovery documents. It has no environment variable of its own, so on a fresh instance it is set through the API, through `EENGINE_SETTINGS`, or under **Configuration** > **General**.

```bash
curl -X POST https://emailengine.example.com/v1/settings \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"serviceUrl": "https://emailengine.example.com"}'
```

#### Access Token Audit Log

`tokenAuditLog` records every request each access token makes, allowed and denied alike, readable with [`GET /v1/tokens/{token}/log`](/docs/api-reference/access-tokens#audit-log). Entries exist only from the moment it is switched on, so turning it on after an incident recovers nothing.

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"tokenAuditLog": true}'
```

Retention per token is bounded by `EENGINE_TOKEN_LOG_ENTRIES` (1000) and `EENGINE_TOKEN_LOG_AGE` (7 days), whichever is reached first. The log lives in Redis, so both cost memory.

#### MCP Endpoint

Serving the [Model Context Protocol](/docs/mcp) at `/mcp` takes two switches, and both have to be on.

`EENGINE_MCP_ENABLED` is the deployment gate, and it defaults to `true`. Setting it to `false` means the `/mcp` routes, the OAuth endpoints and the MCP configuration page are never registered, and `mcpEnabled` has no effect. It is a startup variable, so a change needs a restart.

`mcpEnabled` is the runtime switch, off by default and changed under **Configuration** > **MCP**. While it is off, every request to `/mcp` answers `404`.

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"mcpEnabled": true}'
```

`mcpOAuthEnabled` runs the built-in OAuth 2.1 authorization server used by MCP clients that cannot be configured with a static access token, such as web connectors. It requires `mcpEnabled` and a configured `serviceUrl`; without both, the OAuth discovery endpoints answer `404`.

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"mcpOAuthEnabled": true, "serviceUrl": "https://emailengine.example.com"}'
```

#### Branding of Hosted Pages

`templateHeader` and `templateHtmlHead` inject HTML into the hosted authentication forms and the other hosted pages, for a header of your own, a stylesheet or an analytics snippet. Each accepts up to 1 MB. `pageBrandName` replaces the product name in page titles. All three are edited under **Configuration** > **Branding**. `locale` and `timezone` set the language and the timezone dates are rendered in, and live under **Configuration** > **General**.

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

### Mail Handling

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `imapIndexer` | `full` or `fast` | `full` | How much of a mailbox to track per sync; `fast` misses deletions and flag changes |
| `resolveGmailCategories` | boolean | off | Detect Gmail category tabs for IMAP connections |
| `ignoreMailCertErrors` | boolean | off | Accept invalid TLS certificates from mail servers |
| `trackOpens` | boolean | off | Insert a tracking pixel into outgoing HTML |
| `trackClicks` | boolean | off | Rewrite links in outgoing HTML for click tracking |
| `smtpEhloName` | string | system hostname | Hostname for SMTP EHLO/HELO |
| `deliveryAttempts` | integer | `10` | Delivery attempts before a queued message is marked failed |
| `queueKeep` | integer | `0`, or `EENGINE_QUEUE_REMOVE_AFTER` | Completed queue entries to keep; failed entries have their own floor |
| `imapStrategy` | `default`, `dedicated`, `random` | `default` | How outbound IMAP connections pick a local address |
| `smtpStrategy` | `default`, `dedicated`, `random` | `default` | How outbound SMTP connections pick a local address |
| `localAddresses` | array of IP strings | unset | Local addresses available to the strategies above |
| `proxyEnabled` | boolean | off | Route every outbound connection through a proxy: IMAP and SMTP sessions, and, since v2.79.9, HTTP requests too |
| `proxyUrl` | string | unset | Proxy URL, for example `socks5://proxy.example.com:1080` |
| `httpProxyEnabled` | boolean | off, or `EENGINE_HTTP_PROXY_ENABLED` | Send outbound HTTP requests (webhooks, OAuth2, provider APIs) through a proxy of their own instead of `proxyUrl` |
| `httpProxyUrl` | string | unset, or `EENGINE_HTTP_PROXY_URL` | HTTP proxy URL, used in place of `proxyUrl` for HTTP requests |
| `gmailSubscriptionTtl` | integer (days) | unset (Google default, 31 days) | Gmail Pub/Sub subscription inactivity expiry; `0` never expires |
| `imapClientName` | string | unset | Client name advertised through the IMAP ID extension |
| `imapClientVersion` | string | unset | Client version advertised through IMAP ID |
| `imapClientVendor` | string | unset | Vendor advertised through IMAP ID |
| `imapClientSupportUrl` | string | unset | Support URL advertised through IMAP ID |

#### Gmail Pub/Sub Subscription TTL

`gmailSubscriptionTtl` controls how long a Gmail Pub/Sub subscription survives without activity before Google deletes it. It is also editable at **Integrations** > **OAuth2 Apps** > **Gmail Subscriptions**.

| Value | Behavior |
|-------|----------|
| Unset | Google default, 31 days |
| `0` | Never expires |
| `1` to `365` | Custom TTL in days |

```bash
curl -X POST https://emailengine.example.com/v1/settings \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"gmailSubscriptionTtl": 0}'
```

A change takes effect when a subscription is next created or updated, not immediately. See [Gmail Pub/Sub Integration](/docs/accounts/gmail/gmail-pubsub#subscription-expiration-ttl).

#### IMAP ID Extension

The four `imapClient*` keys fill in the IMAP ID response ([RFC 2971](https://tools.ietf.org/html/rfc2971)) EmailEngine sends to mail servers. Some servers adjust rate limits, feature availability or logging based on what the client reports.

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "imapClientName": "EmailEngine",
    "imapClientVersion": "2.79.4",
    "imapClientVendor": "Postal Systems"
  }'
```

### AI Processing

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `generateEmailSummary` | boolean | off | Generate a summary for each incoming message. Turning it on also turns `notifyText` on |
| `openAiAPIKey` | string | unset | OpenAI API key |
| `openAiModel` | string | unset | Model name |
| `openAiAPIUrl` | string | `https://api.openai.com` | API base URL |
| `openAiTemperature` | number | unset | Sampling temperature |
| `openAiTopP` | number | unset | Nucleus sampling parameter |
| `openAiMaxTokens` | number | unset (model-dependent) | Token limit per request |
| `openAiPrompt` | string | unset | Custom system prompt |
| `openAiPreProcessingFn` | string | unset | JavaScript filter deciding which messages are processed |
| `openAiGenerateEmbeddings` | boolean | off | Generate vector embeddings. Turning it on also turns `notifyText` on |

`openAiAPIUrl` points these calls at an OpenAI-compatible service other than OpenAI itself, and it has to include whatever path prefix that service mounts its API under. For Azure OpenAI that means `https://<your-resource>.openai.azure.com/openai/v1`, not the bare host. See [AI and ChatGPT integration](/docs/integrations/ai-chatgpt) for what the generated fields contain and where they appear.

### Built-in SMTP Server and IMAP Proxy

Seeded from the matching `EENGINE_SMTP_*` and `EENGINE_IMAP_PROXY_*` variables at first start; afterwards the settings are what count.

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `smtpServerEnabled` | boolean | `false` | Run the SMTP submission server |
| `smtpServerPort` | integer | `2525` | Listening port |
| `smtpServerHost` | string | `127.0.0.1` | Bind address; empty means all interfaces |
| `smtpServerProxy` | boolean | `false` | Accept the HAProxy PROXY protocol |
| `smtpServerAuthEnabled` | boolean | `true` | Require SMTP authentication |
| `smtpServerPassword` | string | `null` | Shared password; `null` accepts access tokens with the `smtp` scope instead |
| `smtpServerTLSEnabled` | boolean | off | Offer TLS and STARTTLS |
| `imapProxyServerEnabled` | boolean | `false` | Run the IMAP proxy |
| `imapProxyServerPort` | integer | `2993` | Listening port |
| `imapProxyServerHost` | string | `127.0.0.1` | Bind address |
| `imapProxyServerProxy` | boolean | `false` | Accept the HAProxy PROXY protocol |
| `imapProxyServerPassword` | string | `null` | Shared password; `null` accepts access tokens with the `imap-proxy` scope |
| `imapProxyServerTLSEnabled` | boolean | off | Offer TLS |

The maximum message size the SMTP server accepts is a startup variable rather than a setting, `EENGINE_MAX_SMTP_MESSAGE_SIZE`, 25 MB by default. Both listeners take their certificates from the `EENGINE_SMTP_TLS_` and `EENGINE_IMAPPROXY_TLS_` prefixes, described under [TLS Configuration](/docs/configuration/environment-variables#tls-configuration). See [SMTP Interface](/docs/sending/smtp-interface) and [Proxying Connections](/docs/accounts/proxying-connections).

### Exports

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `exportMaxConcurrent` | integer | `2` | Concurrent exports per account |
| `exportMaxGlobalConcurrent` | integer | `8` | Concurrent exports across the instance |
| `gmailExportBatchSize` | integer | `10` (max 50) | Parallel message fetches per Gmail export |
| `outlookExportBatchSize` | integer | `20` (max 20) | Messages per MS Graph batch request |
| `exportMaxMessages` | integer | `500000` | Messages per export |
| `exportMaxSize` | integer | `10737418240` | Export file size limit, in bytes |

Where export files are written, how long they are kept and how long a single job may run are startup variables: `EENGINE_EXPORT_PATH`, `EENGINE_EXPORT_MAX_AGE` and `EENGINE_EXPORT_TIMEOUT`. See [Exporting Messages](/docs/receiving/exporting).

## OAuth2 Applications

OAuth2 applications for Gmail, Outlook and Mail.ru are runtime configuration, but they are their own API resource rather than settings keys. Register one with [`POST /v1/oauth2`](/docs/api/post-v-1-oauth-2), or in the admin interface at **Integrations** > **OAuth2 Apps** > **Register new application**.

**Providers:** `gmail` (three-legged OAuth2 with user consent), `gmailService` (Gmail service account, two-legged), `outlook` (Microsoft 365 delegated access), `outlookService` (Microsoft 365 client credentials, application access), and `mailRu`.

```bash
curl -X POST https://emailengine.example.com/v1/oauth2 \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Gmail",
    "provider": "gmail",
    "clientId": "123456789.apps.googleusercontent.com",
    "clientSecret": "GOCSPX-xxxxxxxxxxxxx",
    "baseScopes": "imap",
    "enabled": true
  }'
```

```bash
curl -X POST https://emailengine.example.com/v1/oauth2 \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Outlook",
    "provider": "outlook",
    "clientId": "12345678-1234-1234-1234-123456789012",
    "clientSecret": "your-azure-secret",
    "authority": "common",
    "enabled": true
  }'
```

```bash
curl -X POST https://emailengine.example.com/v1/oauth2 \
  -H "Authorization: Bearer YOUR_TOKEN" \
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
- `provider` - One of `gmail`, `gmailService`, `outlook`, `outlookService`, `mailRu`
- `clientId` and `clientSecret` - OAuth2 credentials for three-legged applications
- `serviceClient`, `serviceClientEmail` and `serviceKey` - Service account credentials for `gmailService`
- `authority` - Microsoft tenant: `common`, `organizations`, `consumers`, or a tenant ID
- `baseScopes` - Connection type: `imap`, `api`, or `pubsub`
- `enabled` - Whether the application is active

The rest of the resource is in the generated reference: [list](/docs/api/get-v-1-oauth-2), [register](/docs/api/post-v-1-oauth-2), [get](/docs/api/get-v-1-oauth-2-app), [update](/docs/api/put-v-1-oauth-2-app) and [delete](/docs/api/delete-v-1-oauth-2-app). For the provider-side setup, see [Gmail OAuth2 Setup](/docs/accounts/gmail/gmail-imap) and [Outlook OAuth2 Setup](/docs/accounts/microsoft-365/outlook-365).

## Configuration File Example

TOML is the native configuration format. The file is not picked up by name or location: point EmailEngine at it with `--config=/path/to/config.toml` or the `NODE_CONFIG_PATH` environment variable, or it is not read. Every key is the command-line argument without the leading `--`, so `--api.port=3000` is `port = 3000` under `[api]`.

```toml
# EmailEngine Configuration File

[service]
# Encryption secret, required for production
# Generate with: openssl rand -hex 32
secret = "your-64-character-hex-secret-here"

# Maximum time for an IMAP command
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
```

The [CLI reference](/docs/configuration/cli#all-server-arguments) lists every key the file accepts, including those `emailengine --help` does not show.

## Priority Order

When the same startup key is set in more than one place, the environment variable wins, then the command line, then the configuration file. [Configuration Precedence](/docs/configuration#configuration-precedence) lists all five layers in order.

```bash
# Config file: api.port = 3000
# Command line: --api.port=4000
# Environment: EENGINE_PORT=5000

# Result: port 5000, because the environment wins
```

Runtime settings are not part of this order. They are stored in Redis and read from there, so a variable that seeds one has an effect only on the first start, before a stored value exists.

## See Also

- [Environment Variables](/docs/configuration/environment-variables) - Every startup variable, with its CLI argument and configuration-file key
- [Settings API](/docs/api/post-v-1-settings) - The generated reference for the settings request and response shapes
- [Prepared Settings](/docs/configuration/prepared-settings) - Provisioning these settings at first start with `EENGINE_SETTINGS`
- [Security Best Practices](/docs/deployment/security) - Secrets, proxies and access control
- [Monitoring](/docs/advanced/monitoring) - The health, stats and Prometheus endpoints, which are not configurable
