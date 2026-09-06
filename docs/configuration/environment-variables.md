---
title: Environment Variables Reference
description: Complete reference of all environment variables for configuring EmailEngine
sidebar_position: 2
---

# Environment Variables Reference

Complete reference for all EmailEngine environment variables, with the command-line argument and configuration-file key each one corresponds to. These settings are loaded at application startup and require a restart to take effect.

Startup configuration is only half of the picture. Webhook URLs, sending limits, proxies, branding and the rest live in Redis and are changed at runtime without a restart; a few of the variables below seed one of them at first start and then stop mattering. Those keys are documented in the [Configuration Options Reference](/docs/reference/configuration-options#runtime-settings-reference).

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
Most environment variables can also be set via command-line arguments using the format `--section.key=value`. For example, `EENGINE_HOST=0.0.0.0` can be set as `--api.host=0.0.0.0`. The tables below name the ones that have no CLI or config-file form. [See CLI reference →](/docs/configuration/cli)
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

A few non-secret variables are read straight from the environment and have no `_FILE` counterpart: `EENGINE_LOG_LEVEL`, `EENGINE_REDIS_PREFIX`, `EENGINE_FETCH_TIMEOUT`, `EENGINE_GMAIL_FALLBACK_POLL_INTERVAL`, `EENGINE_EXPORT_PATH`, `EENGINE_EXPORT_MAX_AGE`, `EENGINE_HTTP_PROXY_ENABLED`, `EENGINE_HTTP_PROXY_URL`, `EENGINE_QUEUE_KEEP_FAILED`, `EENGINE_QUEUE_KEEP_FAILED_AGE`, `EENGINE_TOKEN_LOG_ENTRIES`, `EENGINE_TOKEN_LOG_AGE`, `EENGINE_TLS_MIN_VERSION`, `EENGINE_TLS_MIN_DH_SIZE`, and `EENGINE_TLS_CIPHERS`.

Two value formats recur in the tables below. A **duration** is a number of milliseconds or a string with a unit, such as `30s`, `12h`, or `7d`. A **byte size** is a number of bytes or a string with a unit, such as `20M` or `1G`.

## Server & Connection

Configure HTTP server and connection settings.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `EENGINE_HOST` | string | `127.0.0.1` | HTTP server bind address | `0.0.0.0` |
| `EENGINE_PORT` | number | `3000` | HTTP server port | `8080` |
| `PORT` | number | `3000` | Alternative to EENGINE_PORT (used by some platforms) | `8080` |
| `EENGINE_API_PROXY` | boolean | unset | Seeds the **Behind Reverse Proxy** setting (`enableApiProxy`) on first start, which makes EmailEngine read the client IP from `X-Forwarded-For`. When the variable is not set, the setting starts out enabled. Once the setting exists in Redis, change it under **Configuration** > **General** or with `POST /v1/settings` | `true` |
| `EENGINE_API_PROXY_ADDRESSES` | string | none | Comma-separated IPs/CIDRs of the proxies allowed to set X-Forwarded-For. Required for IP allowlists to be trustworthy, see [Trusted Proxy Addresses](#trusted-proxy-addresses). Since v2.75.0 | `10.0.0.0/8` |

`0.0.0.0` publishes the admin interface and the API on every interface the host has. Bind to `127.0.0.1` unless something in front of EmailEngine, such as a reverse proxy, is what the outside world reaches.

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

### Reverse Proxy Mode {#reverse-proxy-mode}

**Environment:** `EENGINE_API_PROXY`
**Config file:** `api.proxy`
**Setting:** `enableApiProxy`

When the setting is on, the client address comes from the `X-Forwarded-For` header instead of the socket. `EENGINE_API_PROXY` and `api.proxy` only seed `enableApiProxy` at first start; from then on the stored setting is what counts, and it is changed through `POST /v1/settings` or under **Configuration** > **General**. When neither the variable nor the config-file key is set, the setting starts out enabled, so either declare the proxies with `EENGINE_API_PROXY_ADDRESSES` or switch it off.

```bash
EENGINE_API_PROXY=false
```

### Trusted Proxy Addresses {#trusted-proxy-addresses}

**Environment:** `EENGINE_API_PROXY_ADDRESSES`
**Default:** none

Comma-separated IP addresses or CIDR ranges of your own proxies. Only a request arriving from one of these peers may set `X-Forwarded-For`, and the entries those proxies contributed are discarded when the client address is resolved.

```bash
EENGINE_API_PROXY=true
EENGINE_API_PROXY_ADDRESSES=10.0.0.0/8,192.168.1.10
```

:::warning Security
`X-Forwarded-For` is a list that each proxy appends to, so the left-most entry is whatever the original caller sent. With reverse proxy mode on and no proxy addresses declared, EmailEngine keeps that left-most entry, which means any client able to reach the port directly chooses the address EmailEngine records.

That is fine for logging, but not for the two controls that match on the client address: the [admin interface allowlist](/docs/deployment/security#admin-interface-access-control) (`EENGINE_ADMIN_ACCESS_ADDRESSES`) and per-token `restrictions.addresses`. If you rely on either, set `EENGINE_API_PROXY_ADDRESSES` as well, or leave reverse proxy mode off so the socket address is used.

EmailEngine logs a warning at startup when it detects an admin allowlist combined with undeclared proxies.
:::

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

**With a password and no username (the usual `requirepass` setup):**
```bash
EENGINE_REDIS=redis://:password@redis.example.com:6379
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

A colon is appended to the prefix, so do not end the value with one. Braces make the prefix a Redis Cluster hash tag, which keeps every EmailEngine key on one slot. The prefix is what lets two instances share a Redis database; the database number in the URL does the same for two instances that can have a database each.

[Detailed Redis configuration →](/docs/configuration/redis)

## Email Protocol Settings

Email protocol timeouts and limits.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `EENGINE_MAX_SIZE` | byte size | `5242880` | Max attachment size (5 MB) | `20M` |
| `EENGINE_MAX_BODY_SIZE` | byte size | `52428800` | Max POST body size for message uploads (50 MB) | `100M` |
| `EENGINE_MAX_SMTP_MESSAGE_SIZE` | byte size | `26214400` | Max message size for SMTP submission (25 MB) | `50M` |
| `EENGINE_MAX_PAYLOAD_TIMEOUT` | duration | `10000` | Payload reception timeout for message uploads | `30s` |
| `EENGINE_TIMEOUT` | duration | `10000` | Timeout for an IMAP command, and the default timeout for API calls handed to a worker. A request can override it with the `X-EE-Timeout` header | `30000` |
| `EENGINE_FETCH_TIMEOUT` | ms | `90000` | Timeout for HTTP fetch operations (90 seconds) | `120000` |
| `EENGINE_FETCH_BATCH_SIZE` | number | `1000` | Messages per batch during synchronization | `500` |
| `EENGINE_IMAP_SOCKET_TIMEOUT` | duration | none | Custom socket timeout for IMAP connections | `60000` |
| `EENGINE_CONNECTION_SETUP_DELAY` | duration | `0` | Delay between assigning account connections to workers at startup | `5000` |
| `EENGINE_CHUNK_SIZE` | byte size | `1000000` | Download chunk size for streaming attachments (1 MB) | `5000000` |
| `EENGINE_GMAIL_FALLBACK_POLL_INTERVAL` | ms | `600000` | How often a Gmail API account is polled for changes when no Pub/Sub notification has arrived (10 minutes) | `300000` |
| `EENGINE_MAX_IMAP_AUTH_FAILURE_TIME` | duration | `259200000` | How long an account may keep failing authentication (3 days) before EmailEngine switches its syncing off. See [Max IMAP Auth Failure Time](#max-imap-auth-failure-time) | `24h` |

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

### Max IMAP Auth Failure Time {#max-imap-auth-failure-time}

**Environment:** `EENGINE_MAX_IMAP_AUTH_FAILURE_TIME`
**Default:** `259200000` (3 days)

How long an account may keep failing authentication before EmailEngine stops retrying it. Once the first authentication error of the current run is older than this window, syncing is switched off for the account: EmailEngine sets `imap.disabled`, records the time in the account's read-only `authFailureDisabledAt` field, stores the reason as the account's last error, closes the connection, and sends an [`authenticationError`](/docs/webhooks/authenticationerror) webhook. The account then reports the state `unset`, and `PUT /v1/account/{account}/reconnect` answers `{"reconnect": false}` for it. This keeps a mailbox with a changed password, or an OAuth2 grant the user has revoked, from hammering the mail server or the provider's token endpoint indefinitely.

```bash
EENGINE_MAX_IMAP_AUTH_FAILURE_TIME=1d
```

The threshold covers every account type. Gmail API and Microsoft Graph accounts have no IMAP configuration of their own, so EmailEngine writes the flag for them, and both clients check it before attempting a token refresh. Before v2.79.3 the check was gated on stored IMAP settings, which OAuth2 accounts do not have, so their revoked grants were retried indefinitely. Upgrading an instance that has collected such accounts switches off every one already past the threshold on its next failed refresh, so expect a burst of `authenticationError` webhooks. Raise this value before upgrading to stage that.

`authFailureDisabledAt` is what tells an automatic switch-off from the operator's own send-only switch, since both set `imap.disabled`. [Accounts switched off after authentication failures](/docs/accounts/managing-accounts#accounts-switched-off-after-authentication-failures) covers what the account looks like in each state, what turns syncing back on, and how the behavior changed across versions.

## Worker Threads

Control worker thread configuration for processing workload.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `EENGINE_WORKERS` | number | `4` | Account worker thread count - syncs IMAP, Gmail API, and Outlook (Microsoft Graph) accounts | `8` |
| `EENGINE_WORKERS_API` | number | `1` | API/HTTP worker threads. Values above `1` require `SO_REUSEPORT`, which needs Linux and a Node.js with the `reusePort` listen option (22.12 or 23.1 and later); other platforms fall back to a single worker. Accepts `cpus` | `4` |
| `EENGINE_WORKERS_SUBMIT` | number | `1` | Worker threads for email submission | `2` |
| `EENGINE_WORKERS_WEBHOOKS` | number | `1` | Worker threads for webhook delivery | `2` |

`EENGINE_WORKERS` also accepts `cpus`, meaning one worker per CPU core, so `EENGINE_WORKERS=cpus` is the equivalent of `EENGINE_WORKERS=$(nproc)` on Linux. `EENGINE_WORKERS_API` accepts it too.

The export worker count has no environment variable: set `--workers.export` on the command line or `export` under `[workers]` in the config file (default `1`). The Workers page at `/admin/internals` labels that row `EENGINE_WORKERS_EXPORT`, but nothing reads a variable by that name.

When `EENGINE_WORKERS_API` is above `1`, EmailEngine probes at startup whether two sockets can share the listen port. If they cannot, it starts a single API worker instead and reports the fallback and its cause on the Workers page.

**Examples:**

**High-performance setup:**
```bash
EENGINE_WORKERS=8
EENGINE_WORKERS_SUBMIT=4
EENGINE_WORKERS_WEBHOOKS=4
```

**Resource-constrained environment:**
```bash
EENGINE_WORKERS=2
EENGINE_WORKERS_SUBMIT=1
EENGINE_WORKERS_WEBHOOKS=1
```

## Queue Management

Configure job queue retention, cleanup, and concurrency.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `EENGINE_QUEUE_REMOVE_AFTER` | number | `0` | Number of completed jobs to keep per queue (0 = remove immediately). This is a count of jobs, not a duration. Seeds the `queueKeep` setting on first start; afterwards the **Job History Limit** under **Configuration** > **General** wins | `5000` |
| `EENGINE_QUEUE_KEEP_FAILED` | number | `500` | Floor for failed entries retained per queue, independent of the setting above | `2000` |
| `EENGINE_QUEUE_KEEP_FAILED_AGE` | seconds | `604800` | How long failed entries are retained (7 days) | `259200` |
| `EENGINE_SUBMIT_QC` | number | `1` | Concurrency for email submission queue | `4` |
| `EENGINE_NOTIFY_QC` | number | `1` | Concurrency for notification/webhook queue | `4` |
| `EENGINE_EXPORT_QC` | number | `1` | Concurrency for export queue | `2` |
| `EENGINE_SUBMIT_DELAY` | duration | none | Pause a submit worker takes after each delivery | `1000` |

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

A failed job is the only record that a delivery was given up on, so failures are retained on their own terms: at least `EENGINE_QUEUE_KEEP_FAILED` entries for `EENGINE_QUEUE_KEEP_FAILED_AGE` seconds, however few completed entries are wanted. Asking for more completed entries than that floor raises the failed floor with it. Completed entries are also dropped after 24 hours, whatever the count allows. See [Queue Management](/docs/advanced/queue-management#enable-job-retention).

## Webhook Delivery

Configure how webhook deliveries are made. Webhook URLs, events, and custom routes are configured in the admin interface, not here.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `EENGINE_WEBHOOK_TIMEOUT` | duration | `30000` | Wall-clock timeout for a single delivery attempt | `10s` |
| `EENGINE_WEBHOOK_EGRESS_POLICY` | string | `link-local` | Which destinations deliveries may reach: `link-local`, `private`, or `off`. An unrecognized value falls back to `link-local`. Since v2.75.0 | `private` |

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

[Exporting Messages guide →](/docs/receiving/exporting)

## IMAP Proxy Server

Enable and configure the built-in IMAP proxy server feature.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `EENGINE_IMAP_PROXY_ENABLED` | boolean | `false` | Enable IMAP proxy server | `true` |
| `EENGINE_IMAP_PROXY_HOST` | string | `127.0.0.1` | IMAP proxy bind address | `0.0.0.0` |
| `EENGINE_IMAP_PROXY_PORT` | number | `2993` | IMAP proxy server port | `993` |
| `EENGINE_IMAP_PROXY_SECRET` | string | none | Shared password accepted for every account. A password that does not match is checked as an access token with the `imap-proxy` scope instead, so tokens work whether or not this is set | `your-secret-key` |
| `EENGINE_IMAP_PROXY_PROXY` | boolean | `false` | Enable PROXY protocol for IMAP proxy server | `true` |

These five variables seed the `imapProxyServerEnabled`, `imapProxyServerHost`, `imapProxyServerPort`, `imapProxyServerPassword`, and `imapProxyServerProxy` settings on first start only. Once the settings exist in Redis, the values under **Configuration** > **IMAP Proxy** are what count, and a changed variable has no effect.

Turn the PROXY protocol on only when something in front of the listener actually speaks it. A plain IMAP client connecting to a listener that expects a PROXY header is rejected.

**Examples:**

**Enable IMAP proxy with shared secret:**
```bash
EENGINE_IMAP_PROXY_ENABLED=true
EENGINE_IMAP_PROXY_HOST=0.0.0.0
EENGINE_IMAP_PROXY_PORT=2993
EENGINE_IMAP_PROXY_SECRET=my-secure-secret-key
```

**Enable IMAP proxy without a shared password, so that only tokens with the `imap-proxy` scope are accepted:**
```bash
EENGINE_IMAP_PROXY_ENABLED=true
EENGINE_IMAP_PROXY_HOST=0.0.0.0
EENGINE_IMAP_PROXY_PORT=2993
```

## SMTP Proxy Server

Enable and configure the built-in [SMTP submission server](/docs/sending/smtp-interface).

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `EENGINE_SMTP_ENABLED` | boolean | `false` | Enable the SMTP server | `true` |
| `EENGINE_SMTP_HOST` | string | `127.0.0.1` | SMTP server bind address | `0.0.0.0` |
| `EENGINE_SMTP_PORT` | number | `2525` | SMTP server port | `587` |
| `EENGINE_SMTP_SECRET` | string | none | Shared password accepted for every account. A password that does not match is checked as an access token with the `smtp` scope instead, so tokens work whether or not this is set | `your-secret-key` |
| `EENGINE_SMTP_PROXY` | boolean | `false` | Accept the HAProxy PROXY protocol, so the client address EmailEngine sees is the original caller rather than the load balancer | `true` |
| `EENGINE_MAX_SMTP_MESSAGE_SIZE` | byte size | `26214400` | Max message size the SMTP server accepts (25 MB) | `50M` |

The first five seed the `smtpServerEnabled`, `smtpServerHost`, `smtpServerPort`, `smtpServerPassword`, and `smtpServerProxy` settings on first start only. Once the settings exist in Redis, the values under **Configuration** > **SMTP Server** are what count, and a changed variable has no effect. `EENGINE_MAX_SMTP_MESSAGE_SIZE` is read on every start.

As with the IMAP proxy, turn the PROXY protocol on only when something in front of the listener speaks it: a plain SMTP client connecting to a listener that expects a PROXY header is rejected.

**Examples:**

**Enable the SMTP server with a shared password:**
```bash
EENGINE_SMTP_ENABLED=true
EENGINE_SMTP_HOST=0.0.0.0
EENGINE_SMTP_PORT=2525
EENGINE_SMTP_SECRET=my-secure-secret-key
```

**Enable the SMTP server without a shared password, so that only tokens with the `smtp` scope are accepted:**
```bash
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

Spelled out for the API listener, that is `EENGINE_API_TLS_KEY`, `EENGINE_API_TLS_CERT`, `EENGINE_API_TLS_CA`, `EENGINE_API_TLS_PASSPHRASE`, `EENGINE_API_TLS_DHPARAM`, `EENGINE_API_TLS_CIPHERS`, `EENGINE_API_TLS_ECDH_CURVE`, `EENGINE_API_TLS_MIN_VERSION`, `EENGINE_API_TLS_MAX_VERSION`, `EENGINE_API_TLS_REJECT_UNAUTHORIZED` and `EENGINE_API_TLS_REQUEST_CERT`. Each one maps to the Node.js TLS option of the same name.

The variable carries the PEM content itself, not a path:

```bash
EENGINE_API_TLS=true
EENGINE_API_TLS_KEY="$(cat /etc/emailengine/tls/api.key)"
EENGINE_API_TLS_CERT="$(cat /etc/emailengine/tls/api.crt)"
EENGINE_API_TLS_MIN_VERSION=TLSv1.2
```

To point at a file instead, use the `_FILE` suffix described under [Loading Values From Files](#loading-values-from-files), which is the usual way to mount a key into a container without putting it in the environment:

```bash
EENGINE_API_TLS=true
EENGINE_API_TLS_KEY_FILE=/etc/emailengine/tls/api.key
EENGINE_API_TLS_CERT_FILE=/etc/emailengine/tls/api.crt
```

The configuration file has a third form: `keyPath`, `certPath`, `caPath`, and `dhparamPath` under `[api.tls]` name files to read, and any other key from the table above is given as a plain value. A variable set in the environment overrides the same key from the file.

:::note A managed certificate wins over these variables
When the SMTP server or the IMAP proxy starts with TLS enabled, EmailEngine looks for a certificate matching the hostname in `serviceUrl` and uses it if one is valid. That happens after these variables are read, so a certificate EmailEngine manages takes precedence over `EENGINE_SMTP_TLS_CERT` and `EENGINE_IMAPPROXY_TLS_CERT`.
:::

Most deployments do not need any of this, because TLS is terminated at a reverse proxy instead. See [Nginx Reverse Proxy](/docs/deployment/nginx-proxy).

## Security & Access Control

Security settings and access restrictions.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `EENGINE_SECRET` | string | none | **Required for production.** Master encryption key for all credentials stored in Redis (AES-256-GCM). | `$(openssl rand -hex 32)` |
| `EENGINE_ADMIN_ACCESS_ADDRESSES` | string | all | Comma-separated list of IP addresses allowed to access admin interface | `192.168.1.0/24,10.0.0.1` |
| `EENGINE_REQUIRE_API_AUTH` | boolean | `true` | Require access tokens on API calls. Seeds the corresponding setting on first start only; afterwards change it under **Configuration** > **Security** | `false` |
| `EENGINE_ENABLE_OAUTH_TOKENS_API` | boolean | `false` | Allow retrieving raw OAuth2 tokens through `GET /v1/account/{account}/oauth-token`. Seeds the `enableOAuthTokensApi` setting on first start only | `true` |
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
- The settings that hold a secret of their own: `cookiePassword` (the admin session key), `serviceSecret` (the webhook signature and signed-link key), `smtpServerPassword`, `imapProxyServerPassword`, `openAiAPIKey` and `totpSeed`

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

Changing the secret does not re-encrypt what is already stored. The `emailengine encrypt` command does that, taking the new secret and one or more `--decrypt` values for the old ones. See [Changing Encryption Secret](/docs/advanced/encryption#changing-encryption-secret) for the procedure and what a partially rotated database looks like.

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
| `EENGINE_CORS_MAX_AGE` | duration | `60` seconds | How long a browser may cache a CORS preflight response. A bare number is milliseconds, so use a unit | `1h` |
| `EENGINE_DOCUMENT_STORE_ENABLED` | boolean | `false` | Enable the deprecated Document Store (Elasticsearch) feature gate | `true` |
| `EENGINE_MCP_ENABLED` | boolean | `true` | Register the [MCP endpoint](/docs/mcp) routes. Registration alone serves nothing: the `mcpEnabled` setting is the runtime switch | `false` |
| `EENGINE_CSP_MODE` | string | `enforce` | How the Content-Security-Policy is delivered: `enforce`, `report-only` (violations are only reported to the browser console) or `off`. The other security headers are unaffected | `report-only` |
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

Since EmailEngine v2.75.0, [web-safe HTML](/docs/receiving/web-safe-html) wraps the quoted tail of a reply in a collapsible block so a message renders as what the sender wrote. `EENGINE_DISABLE_THREAD_COLLAPSE=true` restores the previous output shape, in which the whole thread is rendered inline.

The Document Store (Elasticsearch) feature is deprecated and disabled by default since EmailEngine v2.71.0, and it is **removed from EmailEngine releases starting October 1, 2026**. This startup gate must be turned on before EmailEngine will run the document indexing worker or register the Document Store API and admin endpoints (`/v1/chat/{account}`, `/v1/unified/search`, and the `Configuration > Document Store` page). While the gate is off, those endpoints return `404`, even if the runtime "Document Store" setting is still enabled. The equivalent config-file setting is `[documentStore] enabled = true` (CLI flag `--documentStore.enabled=true`).

If you depend on the Document Store, plan the migration now. Staying on the last release that still ships it means running an EmailEngine that no longer receives security updates.

`EENGINE_CSP_MODE` selects how the [Content-Security-Policy](/docs/deployment/security#security-headers) reaches the browser. `enforce` (the default) blocks what the policy forbids. `report-only` keeps only the framing protection enforced and delivers the rest of the policy as `Content-Security-Policy-Report-Only`, so a violation shows up in the browser console without breaking the page - use it to check a customised deployment before enforcing. `off` sends the framing directive only. The equivalent config-file setting is `[api] cspMode`, and a change requires a restart.

`EENGINE_MCP_ENABLED` is a deployment gate rather than an on switch: it defaults to `true`, and setting it to `false` removes the `/mcp` routes and the MCP configuration page from the instance so the surface does not exist at all. The endpoint itself stays off until an admin enables the `mcpEnabled` setting, which starts out disabled. The equivalent config-file setting is `[mcp] enabled = false` (CLI flag `--mcp.enabled=false`), and a change requires a restart. See [MCP for AI Agents](/docs/mcp).

Since EmailEngine v2.76.0, `EENGINE_UPDATE_CHECK_DISABLED=true` disables the check against `api.github.com` that powers the "update available" notice in the admin dashboard. The check runs once at startup, sends nothing beyond a standard User-Agent header, and fails silently without network access, but it is the only background network call that is not tied to a subscription license. Subscription licenses additionally validate daily against `postalsys.com`, carrying an [anonymized feature beacon](/docs/deployment/compliance#no-developer-access) that `EENGINE_BEACON_DISABLED=true` disables; perpetual licenses are verified offline and never contact the license server at all. With the update check disabled, a perpetual-license instance makes no background network calls whatsoever.

Two further variables exist but are set by installers rather than by you: `EENGINE_INSTALL_SCRIPT=true` is written into the systemd unit by the [installation script](/docs/installation/linux), and `EENGINE_DOCEAN=true` by the DigitalOcean Marketplace image. They only record the installation channel, which the **Upgrade** page in the admin interface uses to show the matching upgrade instructions and the feature beacon reports.

### Cross-Origin Requests {#cors-configuration}

The API sends no CORS headers at all until `EENGINE_CORS_ORIGIN` names the origins that may call it from a browser. The value is split on whitespace, so several origins fit in one variable, and `*` allows any origin.

```bash
EENGINE_CORS_ORIGIN="https://app.example.com https://admin.example.com"
EENGINE_CORS_MAX_AGE=1h
```

`EENGINE_CORS_MAX_AGE` is how long a browser may cache a preflight response. A bare number is read as milliseconds, so `EENGINE_CORS_MAX_AGE=3600` means 4 seconds rather than an hour; give it a unit. See [Cross-Origin Requests](/docs/deployment/security#cross-origin-requests) in the security guide for when exposing the API to a browser is the wrong shape.

### Raw Protocol Logging

`EENGINE_LOG_RAW=true` records the IMAP conversation byte for byte, including message content, and stops the OAuth2 token exchanges being masked in the logs: `access_token`, `refresh_token`, `id_token`, `client_secret` and the authorization `code` are written in full instead of truncated. The client frames of the IMAP authentication exchange are still withheld, so account passwords do not appear, but treat any log captured this way as a credential store: keep it out of production, and delete it once the debugging session is over. It also produces very large files. [Logging](/docs/advanced/logging#redaction) has the complete redaction rules.

## HTTP Proxy

Route outbound HTTP and HTTPS requests (webhooks, OAuth2 token requests, Gmail API and Microsoft Graph calls, license validation) through an HTTP or SOCKS proxy.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `EENGINE_HTTP_PROXY_ENABLED` | boolean | `false` | Enable a dedicated proxy for outbound HTTP requests | `true` |
| `EENGINE_HTTP_PROXY_URL` | string | none | Proxy server URL (HTTP, HTTPS, or SOCKS) | `socks5://proxy.example.com:1080` |

:::info Settings Override
These environment variables override the equivalent API settings (`httpProxyEnabled` and `httpProxyUrl` via `POST /v1/settings`). When both are set, environment variables take precedence.
:::

Since v2.79.9 these are not the only way to proxy HTTP traffic. The global proxy (`proxyEnabled` and `proxyUrl`) covers HTTP requests as well as IMAP and SMTP, and the variables here are the override for sending HTTP somewhere else. [Proxy Configuration](/docs/accounts/imap-smtp#proxy-configuration) has the full precedence order.

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

**Keep HTTP requests off the global proxy:**
```bash
EENGINE_HTTP_PROXY_ENABLED=false
```

An explicit `false` disables HTTP proxying entirely, so IMAP and SMTP keep using `proxyUrl` while HTTP requests go out directly. This restores the behavior of releases up to v2.79.8.

:::note Standard proxy variables are ignored
EmailEngine does not read `HTTP_PROXY`, `HTTPS_PROXY` or `NO_PROXY`. Use the variables above, or the proxy settings in the admin interface.
:::

## Logging & Monitoring

Logging configuration and error tracking.

| Variable | Type | Default | Description | Example |
|----------|------|---------|-------------|---------|
| `EENGINE_LOG_LEVEL` | string | `trace` | Log level (silent, fatal, error, warn, info, debug, trace) | `info` |
| `SENTRY_DSN` | string | none | Sentry DSN for error reporting. When set, it pins the DSN and overrides the runtime Sentry toggle | `https://key@sentry.example.com/1` |

**Log Levels:**
- `trace` - Very detailed, includes all protocol messages
- `debug` - Detailed operational information
- `info` - General operational messages
- `warn` - Warning messages
- `error` - Error messages only
- `fatal` - Fatal errors only
- `silent` - No output at all

The default is `trace`, which suits a first run and fills a disk in production. Set `info` once the deployment works.

**Examples:**

**Development/debugging:**
```bash
EENGINE_LOG_LEVEL=trace
```

**Production:**
```bash
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

[Monitoring and logging →](/docs/advanced/monitoring)

## Prepared Configuration

Pre-configured settings for automated deployments.

| Variable | CLI argument | Config file key | Description |
|----------|--------------|-----------------|-------------|
| `EENGINE_SETTINGS` | `--settings` | `settings` | JSON object of runtime settings to apply at startup |
| `EENGINE_PREPARED_TOKEN` | `--preparedToken` | `preparedToken` | Exported token hash (from `emailengine tokens export`) |
| `EENGINE_PREPARED_PASSWORD` | `--preparedPassword` | `preparedPassword` | Admin password hash (from `emailengine password --hash`) |
| `EENGINE_PREPARED_LICENSE` | `--preparedLicense` | `preparedLicense` | License key |

**Examples:**

**Prepared settings:**
```bash
EENGINE_SETTINGS='{
  "webhooks": "https://your-app.com/webhook",
  "webhooksEnabled": true,
  "webhookEvents": ["messageNew", "messageSent"]
}'
```

The value is validated against the settings schema at startup. A value that fails validation stops EmailEngine from starting; an unknown key is dropped and logged as an error, so a typo does not go unnoticed.

**Docker Compose (multiline):**
```yaml
environment:
  EENGINE_SETTINGS: >
    {
      "webhooks": "https://your-app.com/webhook",
      "webhooksEnabled": true,
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
EXPORTED=$(emailengine tokens export -t "$TOKEN")

# 3. Use the exported hash (not the raw token)
EENGINE_PREPARED_TOKEN=$EXPORTED
```

### Prepared Admin Password {#prepared-admin-password}

`EENGINE_PREPARED_PASSWORD` takes a base64url-encoded PBKDF2 hash, not a plain password. Generate the hash with the `--hash` flag, which prints it instead of the password:

```bash
# Generate password hash
emailengine password -p "your-secure-password" --hash
# Output: JHBia2RmMi1zaGEyNTYkaT02MDAwMDAkMEwwRUVtSkZPNC8vSmtDWlZDTTRTUSRiNCt3QURwWVhOL0ZDUndtMW1QcDdqRkhqR2tjdndxaDBKWFFpS1ZxQTRr

# Use the hash
EENGINE_PREPARED_PASSWORD=JHBia2RmMi1zaGEyNTYkaT02MDAwMDAkMEwwRUVtSkZPNC8vSmtDWlZDTTRTUSRiNCt3QURwWVhOL0ZDUndtMW1QcDdqRkhqR2tjdndxaDBKWFFpS1ZxQTRr
```

Setting a password matters beyond the login page: until one exists, the admin interface opens without a login and refuses to issue access tokens. See [Reset the admin password](/docs/configuration/reset-password) for the command in full.

### Prepared License

The license key is a multi-line PEM block, so read it from a file rather than pasting it inline:

```bash
EENGINE_PREPARED_LICENSE="$(cat /path/to/license.txt)"
```

`--licensePath=/path/to/license.txt` does the same thing from the command line, and stops EmailEngine with exit status 13 if the file fails verification.

Without a license key an instance runs unlicensed until a trial is activated from the dashboard. [Licensing](/docs/licensing) covers the trial, activation and what a suspended license stops.

[Prepared configuration guide →](/docs/configuration/prepared-settings)

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

Environment variables and their command-line equivalents:

| Environment Variable | CLI Argument | Description |
|---------------------|--------------|-------------|
| `EENGINE_REDIS` or `REDIS_URL` | `--dbs.redis` | Redis connection URL |
| `EENGINE_HOST` | `--api.host` | HTTP server bind address |
| `EENGINE_PORT` or `PORT` | `--api.port` | HTTP server port |
| `EENGINE_API_PROXY` | `--api.proxy` | Seed for the Behind Reverse Proxy setting |
| `EENGINE_LOG_LEVEL` | `--log.level` | Log level |
| `EENGINE_LOG_RAW` | `--log.raw` | Raw IMAP protocol logging |
| `EENGINE_SECRET` | `--service.secret` | Encryption secret |
| `EENGINE_WORKERS` | `--workers.imap` | Account worker count |
| `EENGINE_WORKERS_API` | `--workers.api` | API/HTTP worker count |
| `EENGINE_WORKERS_WEBHOOKS` | `--workers.webhooks` | Webhook worker count |
| `EENGINE_WORKERS_SUBMIT` | `--workers.submit` | Submission worker count |
| `EENGINE_MAX_SIZE` | `--api.maxSize` | Max attachment size |
| `EENGINE_MAX_BODY_SIZE` | `--api.maxBodySize` | Max request body for message uploads |
| `EENGINE_MAX_PAYLOAD_TIMEOUT` | `--api.maxPayloadTimeout` | Time allowed to receive a request body |
| `EENGINE_TIMEOUT` | `--service.commandTimeout` | IMAP command timeout |
| `EENGINE_CONNECTION_SETUP_DELAY` | `--service.setupDelay` | Delay between assigning connections to workers |
| `EENGINE_FETCH_BATCH_SIZE` | `--service.fetchBatchSize` | Messages per fetch batch |
| `EENGINE_SUBMIT_DELAY` | `--submitDelay` | Pause between submissions |
| `EENGINE_SUBMIT_QC` | `--queues.submit` | Concurrent email submissions |
| `EENGINE_NOTIFY_QC` | `--queues.notify` | Concurrent webhook deliveries |
| `EENGINE_EXPORT_QC` | `--queues.export` | Concurrent exports per export worker |
| `EENGINE_CORS_ORIGIN` | `--cors.origin` | Allowed CORS origins |
| `EENGINE_CORS_MAX_AGE` | `--cors.maxAge` | CORS preflight cache time |
| `EENGINE_SMTP_ENABLED`, `_HOST`, `_PORT`, `_SECRET`, `_PROXY` | `--smtp.enabled`, `--smtp.host`, `--smtp.port`, `--smtp.secret`, `--smtp.proxy` | Built-in SMTP submission server |
| `EENGINE_MAX_SMTP_MESSAGE_SIZE` | `--smtp.maxMessageSize` | Max message size the SMTP server accepts |
| `EENGINE_IMAP_PROXY_ENABLED`, `_HOST`, `_PORT`, `_SECRET`, `_PROXY` | `--imap-proxy.enabled`, `--imap-proxy.host`, `--imap-proxy.port`, `--imap-proxy.secret`, `--imap-proxy.proxy` | Built-in IMAP proxy |
| `EENGINE_MCP_ENABLED` | `--mcp.enabled` | Register the MCP endpoint routes |
| `EENGINE_DOCUMENT_STORE_ENABLED` | `--documentStore.enabled` | Deprecated Document Store feature gate |
| `EENGINE_SETTINGS` | `--settings` | Prepared runtime settings |
| `EENGINE_PREPARED_TOKEN` | `--preparedToken` | Exported token hash |
| `EENGINE_PREPARED_PASSWORD` | `--preparedPassword` | Admin password hash |
| `EENGINE_PREPARED_LICENSE` | `--preparedLicense` | License key |

**Config file form:** the configuration file key is the CLI argument without the leading `--`, so `--api.port=3000` is `port = 3000` under `[api]`, and `--imap-proxy.port=2993` is `port = 2993` under `[imap-proxy]`. The TLS prefixes are the exception: they are read from the environment only, apart from the `keyPath`, `certPath`, `caPath` and `dhparamPath` keys described under [TLS Configuration](#tls-configuration).

A few keys have no environment variable at all, among them `--workers.export`, `--workers.imapProxy` and `--licensePath`. The [CLI reference](/docs/configuration/cli#all-server-arguments) lists every argument, including the ones `--help` does not show.

## See Also

- [Configuration Options Reference](/docs/reference/configuration-options) - The runtime settings `POST /v1/settings` accepts, which these variables do not cover
- [CLI Reference](/docs/configuration/cli) - Command-line arguments as an alternative to environment variables
- [Redis Configuration](/docs/configuration/redis) - Connection URLs, the `family` parameter, persistence and memory
- [Prepared Settings](/docs/configuration/prepared-settings) - Provisioning settings, tokens, a password and a license at first start
- [Security Best Practices](/docs/deployment/security) - What the secret, the proxy variables and the admin allowlist protect
