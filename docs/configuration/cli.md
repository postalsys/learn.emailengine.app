---
title: Command Line Interface (CLI)
description: Complete guide to using the EmailEngine command line interface for administration and automation
sidebar_position: 10
---

# EmailEngine CLI

The EmailEngine command line interface manages tokens, licenses, passwords, accounts, and encryption. It talks to Redis directly, so it can run anywhere that can reach the database, whether or not the server is running.

## Overview

### Remote Administration

The EmailEngine CLI does not need to run on the same server as EmailEngine. You can run CLI commands from any location as long as you can:

1. **Access the Redis instance** used by EmailEngine
2. **Know the encryption secret** (for some operations)

This enables:

- Remote administration
- Automated deployment scripts
- CI/CD pipeline integration
- Backup and migration tools

### Installation

The EmailEngine CLI can be installed in several ways:

#### Option 1: npm (Recommended)

Install globally from npm:

```bash
npm install -g emailengine-app
```

After installation, the `emailengine` command is available system-wide:

```bash
emailengine --version
emailengine help tokens
```

#### Option 2: Download Binary

Download a pre-built binary from the [installation guide](/docs/installation):

**Available formats:**

- Compiled binaries (Linux, macOS, Windows)
- Docker images
- Source distribution

The binary is the `emailengine` command and can be run directly:

```bash
./emailengine [command] [options]
```

#### Option 3: From Source

If running from source code, install globally from the project directory:

```bash
# From the EmailEngine directory
npm install -g .
```

### Getting Help

EmailEngine provides a dynamic help system that adapts to your terminal width:

```bash
# Show all available commands
emailengine --help
emailengine -h
emailengine help
```

View command-specific help:

```bash
emailengine help tokens
emailengine help license
emailengine help check-bounce
```

The `--help` flag is only recognized on its own. `emailengine tokens --help` runs the `tokens` command without a subcommand, so use the `help <command>` form. Help is printed on stderr.

### Version Information

```bash
emailengine --version
emailengine -v
emailengine version
```

**Output:**

```text
EmailEngine v2.79.4 (LICENSE_EMAILENGINE)
```

## Configuration Arguments

### Essential Arguments

#### Redis Connection

**Required for most commands:**

```bash
--dbs.redis="redis://host:port/db"
```

**Examples:**

```bash
# Local Redis, database 0
--dbs.redis="redis://127.0.0.1:6379/0"

# Remote Redis with password
--dbs.redis="redis://:password@remote.example.com:6379/8"

# Redis with username and password
--dbs.redis="redis://user:password@remote.example.com:6379/8"

# TLS connection (Redis over TLS)
--dbs.redis="rediss://user:password@remote.example.com:6380/8"
```

**Important:** Use the same Redis database number as your EmailEngine instance. EmailEngine accepts only `redis://` and `rediss://` (TLS) connection strings - Redis Sentinel and Redis Cluster connection URLs are not supported.

#### Encryption Secret

**Required for encryption-related operations:**

```bash
--service.secret="your-encryption-secret"
```

**When needed:**

- Account export with encrypted fields
- Encryption migration (`encrypt` command)

**Not needed for:**

- Token operations (tokens are stored as hashes, not encrypted)
- License operations
- Password operations

## Commands

### Run EmailEngine Server

Start the EmailEngine application:

```bash
emailengine
```

**With custom configuration:**

```bash
emailengine \
  --dbs.redis="redis://127.0.0.1:6379/8" \
  --api.port=3000 \
  --api.host="0.0.0.0" \
  --workers.imap=8 \
  --log.level="info"
```

**Common options:**

| Option               | Environment Variable       | Description          | Default     |
| -------------------- | -------------------------- | -------------------- | ----------- |
| `--dbs.redis`        | `EENGINE_REDIS`            | Redis connection URL | `redis://127.0.0.1:6379/8` |
| `--api.host`         | `EENGINE_HOST`             | API server host      | `127.0.0.1` |
| `--api.port`         | `EENGINE_PORT`             | API server port      | `3000`      |
| `--workers.imap`     | `EENGINE_WORKERS`          | Account worker count | `4`         |
| `--workers.webhooks` | `EENGINE_WORKERS_WEBHOOKS` | Webhook worker count | `1`         |
| `--log.level`        | `EENGINE_LOG_LEVEL`        | Log level            | `trace`     |
| `--service.secret`   | `EENGINE_SECRET`           | Encryption secret    | None        |

### All Server Arguments

These are the options `emailengine --help` lists:

**General**

| Option | Description | Default |
|--------|-------------|---------|
| `--dbs.redis` | Redis connection URL | `redis://127.0.0.1:6379/8` |
| `--workers.imap` | Account worker threads. `cpus` uses one per CPU core | `4` |
| `--settings` | Pre-configured settings as a JSON string | None |
| `--service.secret` | Key for encrypting stored credentials | None |
| `--service.commandTimeout` | Maximum time for an IMAP command | `10s` |
| `--service.setupDelay` | Delay between assigning connections to workers | `0ms` |
| `--log.level` | Logging level | `trace` |
| `--log.raw` | Log raw IMAP traffic. Includes unmasked credentials | `false` |
| `--workers.webhooks` | Webhook worker threads | `1` |

**API server**

| Option | Description | Default |
|--------|-------------|---------|
| `--api.host` | Bind address | `127.0.0.1` |
| `--api.port` | Port | `3000` |
| `--api.maxSize` | Maximum attachment size when submitting or uploading | `5M` |

**Background tasks**

| Option | Description | Default |
|--------|-------------|---------|
| `--queues.notify` | Concurrent webhook deliveries | `1` |
| `--queues.submit` | Concurrent email submissions | `1` |

**SMTP server**

| Option | Description | Default |
|--------|-------------|---------|
| `--smtp.enabled` | Enable the [SMTP submission server](/docs/sending/smtp-interface) | `false` |
| `--smtp.secret` | Shared SMTP password accepted for all accounts | None |
| `--smtp.host` | Bind address | `127.0.0.1` |
| `--smtp.port` | Port | `2525` |
| `--smtp.proxy` | Accept the HAProxy PROXY protocol | `false` |
| `--smtp.maxMessageSize` | Maximum accepted email size | `25M` |

**Document Store** (deprecated, removed from releases starting October 1, 2026)

| Option | Description | Default |
|--------|-------------|---------|
| `--documentStore.enabled` | Enable the Document Store feature gate | `false` |

**MCP**

| Option | Description | Default |
|--------|-------------|---------|
| `--mcp.enabled` | Register the [MCP endpoint](/docs/mcp) routes. A deployment gate, not the on switch: the `mcpEnabled` setting still has to be turned on | `true` |

Every configuration key follows the same `--section.key=value` form, including keys that `--help` does not list. The ones EmailEngine reads that are not shown above:

| Option | Environment Variable | Description | Default |
|--------|----------------------|-------------|---------|
| `--workers.api` | `EENGINE_WORKERS_API` | API worker threads. Anything above 1 needs `SO_REUSEPORT`, so it falls back to 1 off Linux | `1` |
| `--workers.submit` | `EENGINE_WORKERS_SUBMIT` | Submission worker threads | `1` |
| `--workers.export` | none | Export worker threads. This one has no environment variable | `1` |
| `--workers.imapProxy` | none | IMAP proxy worker threads | `1` |
| `--queues.export` | `EENGINE_EXPORT_QC` | Concurrent exports per export worker | `1` |
| `--submitDelay` | `EENGINE_SUBMIT_DELAY` | Pause between submissions | None |
| `--api.proxy` | `EENGINE_API_PROXY` | Seed for the Behind Reverse Proxy setting on first start | Unset |
| `--licensePath` | none | Path to a license key file to load and verify at startup. A file that fails verification stops EmailEngine with exit status 13 | None |
| `--preparedLicense`, `--preparedToken`, `--preparedPassword` | `EENGINE_PREPARED_LICENSE`, `EENGINE_PREPARED_TOKEN`, `EENGINE_PREPARED_PASSWORD` | The [prepared settings](/docs/configuration/prepared-settings) in argument form | None |
| `--api.maxBodySize` | `EENGINE_MAX_BODY_SIZE` | Maximum request body for message uploads | `50M` |
| `--api.maxPayloadTimeout` | `EENGINE_MAX_PAYLOAD_TIMEOUT` | Time allowed to receive a request body | `10s` |
| `--cors.origin` | `EENGINE_CORS_ORIGIN` | Allowed CORS origins | None |
| `--cors.maxAge` | `EENGINE_CORS_MAX_AGE` | CORS preflight cache time | `60s` |
| `--service.fetchBatchSize` | `EENGINE_FETCH_BATCH_SIZE` | Messages per fetch batch during sync | `1000` |
| `--imap-proxy.enabled`, `--imap-proxy.host`, `--imap-proxy.port`, `--imap-proxy.secret`, `--imap-proxy.proxy` | `EENGINE_IMAP_PROXY_ENABLED`, `EENGINE_IMAP_PROXY_HOST`, `EENGINE_IMAP_PROXY_PORT`, `EENGINE_IMAP_PROXY_SECRET`, `EENGINE_IMAP_PROXY_PROXY` | The [IMAP proxy](/docs/accounts/proxying-connections) listener, with the same meaning as the SMTP server options | `false`, `127.0.0.1`, `2993`, none, `false` |

:::tip Environment Variables
The [Environment Variables reference](/docs/configuration/environment-variables) lists every variable with its default and the config-file equivalent, including the ones that have no CLI form.
:::

---

## Configuration Files

EmailEngine supports TOML configuration files for persistent settings.

### Using Configuration Files

**Create a TOML file:**

```toml
# /etc/emailengine/config.toml

[dbs]
redis = "redis://localhost:6379/8"

[api]
host = "0.0.0.0"
port = 3000

[log]
level = "info"

[service]
secret = "your-encryption-secret"

[workers]
imap = 8
webhooks = 2
submit = 2
```

**Load the configuration:**

```bash
emailengine --config=/etc/emailengine/config.toml
```

### Complete Configuration Example

```toml
# /etc/emailengine/production.toml

# Database configuration
[dbs]
redis = "redis://redis.example.com:6379"

# API server configuration
[api]
host = "0.0.0.0"
port = 3000
proxy = true
maxSize = "20M"

# Worker configuration
[workers]
imap = 8
webhooks = 4
submit = 2

# Logging
[log]
level = "info"

# Service settings
[service]
secret = "your-encryption-secret-32-chars-min"
commandTimeout = "30s"

# IMAP Proxy (optional)
[imap-proxy]
enabled = true
host = "0.0.0.0"
port = 2993
secret = "imap-proxy-secret"

# SMTP Server (optional)
[smtp]
enabled = true
host = "0.0.0.0"
port = 2525
secret = "smtp-server-secret"
```

**Run with config file:**

```bash
emailengine --config=/etc/emailengine/production.toml
```

### Configuration Precedence

A CLI argument loses to the matching `EENGINE_*` environment variable and wins over everything else, so a flag has no effect when the variable is set as well. [Configuration Precedence](/docs/configuration#configuration-precedence) lists all five layers in order.

**Example:**

```bash
# Configuration file has: port = 3000
# CLI argument: --api.port=4000
# Environment variable: EENGINE_PORT=5000

emailengine --config=config.toml --api.port=4000

# Result: Port 5000 (environment variable wins)
```

---

## Token Management

Manage API access tokens for authentication.

### Issue Token

Create a new access token:

```bash
emailengine tokens issue [options]
```

**Options:**

| Option          | Short | Description           | Default  |
| --------------- | ----- | --------------------- | -------- |
| `--description` | `-d`  | Token description | `Generated at <timestamp>` |
| `--scope`       | `-s`  | One of `*`, `api`, `metrics`, `smtp`, `imap-proxy`, `mcp`. Repeat the flag for several | `*` |
| `--account`     | `-a`  | Bind the token to one account | None |
| `--dbs.redis`   |       | Redis connection      | `redis://127.0.0.1:6379/8` |

An unknown scope is refused with the allowed list, rather than issuing a token that cannot be used.

**Examples:**

```bash
# System-wide token with full access
emailengine tokens issue \
  -d "Admin token" \
  -s "*" \
  --dbs.redis="redis://127.0.0.1:6379/8"

# API-only token
emailengine tokens issue \
  -d "API token" \
  -s "api" \
  --dbs.redis="redis://127.0.0.1:6379/8"

# Account-specific token
emailengine tokens issue \
  -d "User token for john@example.com" \
  -s "api" \
  -a "user123" \
  --dbs.redis="redis://127.0.0.1:6379/8"

# Metrics-only token
emailengine tokens issue \
  -d "Prometheus metrics" \
  -s "metrics" \
  --dbs.redis="redis://127.0.0.1:6379/8"
```

**Available scopes:**

- `"*"` - Full access (all operations)
- `"api"` - API calls only
- `"metrics"` - Prometheus metrics endpoint only
- `"smtp"` - SMTP gateway access
- `"imap-proxy"` - IMAP proxy access
- `"mcp"` - [MCP endpoint](/docs/mcp) access for AI agents, and nothing on the REST API

The CLI does not set a permissions record, so an `mcp` token issued here reaches every tool the scope allows. Pair it with `-a` to bind it to one account, or mint a narrowed one in the web interface or over the API. See [MCP Access Control](/docs/mcp/access-control).

**Output:**

The token value, on stdout, with no trailing newline:

```text
f05d76644ea39c4a2ee33e7bffe55808b716a34b51d67b388c7d60498b0f89bc
```

### Export Token

Export token data for backup or transfer:

```bash
emailengine tokens export [options]
```

**Options:**

| Option        | Short | Description            |
| ------------- | ----- | ---------------------- |
| `--token`     | `-t`  | Access token to export |
| `--dbs.redis` |       | Redis connection       |

**Example:**

```bash
emailengine tokens export \
  -t "f05d76644ea39c4a2ee33e7bffe55808b716a34b51d67b388c7d60498b0f89bc" \
  --dbs.redis="redis://127.0.0.1:6379/8"
```

**Output:**

The token record, msgpack-encoded and base64url-encoded, on stdout:

```text
hKJpZNlAMzAxZThjNTFhZjgxM2Q3MzUxNTYzYTFlM2I1NjVkYmEzZWJjMzk4ZjI4OWZjNjgzN
```

**Use cases:**

- Backup tokens before migration
- Transfer tokens between instances
- Pre-configure tokens via environment variables

### Import Token

Import previously exported token data:

```bash
emailengine tokens import [options]
```

**Options:**

| Option        | Short | Description                  |
| ------------- | ----- | ---------------------------- |
| `--token`     | `-t`  | Exported token data (base64url) |
| `--dbs.redis` |       | Redis connection             |

**Example:**

```bash
emailengine tokens import \
  -t "hKJpZNlAMzAxZThjNTFhZjgxM2Q3MzUxNTYzYTFlM2I1NjVkYmEzZWJjMzk4ZjI4OWZjNjgzN" \
  --dbs.redis="redis://127.0.0.1:6379/8"
```

**Output:**

```text
Token was imported
```

**Important:** Use the exported data, not the original token. The same exported value is what `EENGINE_PREPARED_TOKEN` accepts.

---

## License Management

Manage EmailEngine licenses.

### Show the License Text

```bash
emailengine license
```

This prints the version on stdout and the EmailEngine license agreement on stderr, so `emailengine license 2>/dev/null` leaves you with just the version string. It reports the terms EmailEngine is distributed under, not the license key installed on an instance.

For the installed key, read [`GET /v1/license`](/docs/api/get-v-1-license), which returns whether the license is `active`, its `type`, and the `details` of the key, or open the License page in the admin interface.

### Export License

Export the installed license key for backup or transfer:

```bash
emailengine license export --dbs.redis="redis://127.0.0.1:6379/8"
```

**Output:**

The key as one base64url string on stdout. When no license is installed the command reports `Failed to load license information` on stderr and exits with status 1.

```text
eyJsaWNlbnNlIjoiZXlKaGJHY2lPaUpJVXpJMU5pSXNJblI1Y0NJNklrcFhWQ0o5
```

### Import License

Import license key:

```bash
emailengine license import [options]
```

**Options:**

| Option        | Short | Description         |
| ------------- | ----- | ------------------- |
| `--license`   | `-l`  | The license key, either the `BEGIN LICENSE` block or the string from `license export` |
| `--dbs.redis` |       | Redis connection    |

**Example:**

```bash
emailengine license import \
  -l "eyJsaWNlbnNlIjoiZXlKaGJHY2lPaUpJVXpJMU5pSXNJblI1Y0NJNklrcFhWQ0o5" \
  --dbs.redis="redis://127.0.0.1:6379/8"
```

**Output:**

```text
License key was imported
```

A key that fails verification is reported as `Failed to import license information` and the command exits with status 1. Both success and failure messages go to stderr.

**Use cases:**

- Automated license deployment
- License updates
- Migration to new instance

See [Prepared License](/docs/configuration/prepared-settings/license) for automated setup.

---

## Password Management

Manage the admin password for the web interface.

### Set Password

Set or reset the admin password:

```bash
emailengine password [options]
```

**Options:**

| Option        | Short | Description                                 |
| ------------- | ----- | ------------------------------------------- |
| `--password`  | `-p`  | Password to set (auto-generated if omitted) |
| `--hash`      | `-r`  | Print the password hash instead of the password |
| `--dbs.redis` |       | Redis connection                            |

**Examples:**

```bash
# Set specific password
emailengine password \
  -p "MySecurePassword123" \
  --dbs.redis="redis://127.0.0.1:6379/8"

# Auto-generate password
emailengine password \
  --dbs.redis="redis://127.0.0.1:6379/8"

# Get password hash for prepared settings
emailengine password \
  -p "MySecurePassword123" \
  -r \
  --dbs.redis="redis://127.0.0.1:6379/8"
```

**Auto-generated output:**

```text
5b0c9b3e5f0d4a7c8e1f2a3b4c5d6e7f
```

**Hash output (with `-r`):**

```text
JHBia2RmMi1zaGEyNTYkaT02MDAwMDAkMEwwRUVtSkZPNC8vSmtDWlZDTTRTUSRiNCt3QURwWVhOL0ZDUndtMW1QcDdqRkhqR2tjdndxaDBKWFFpS1ZxQTRr
```

**Notes:**

- Minimum 8 characters
- Auto-generated passwords are 32 hex characters
- The hash is a PBKDF2-SHA256 string, base64url-encoded, and is what `EENGINE_PREPARED_PASSWORD` accepts
- Resetting the password also disables TOTP two-factor authentication and removes every registered passkey

See [Reset Password](/docs/configuration/reset-password) for details.

---

## Account Management

### Export Account

Export account data including credentials:

```bash
emailengine export [options]
```

**Options:**

| Option             | Short | Description                               |
| ------------------ | ----- | ----------------------------------------- |
| `--account`        | `-a`  | Account identifier                        |
| `--dbs.redis`      |       | Redis connection                          |
| `--service.secret` |       | Encryption secret (if encryption enabled) |

**Example:**

```bash
emailengine export \
  -a "user123" \
  --dbs.redis="redis://127.0.0.1:6379/8" \
  --service.secret="my-encryption-secret"
```

**Output:** the account record as JSON, on stdout.

**Use cases:**

- Backup specific accounts
- Migrate accounts between instances
- Disaster recovery
- Account auditing

:::danger The export is decrypted
Passing `--service.secret` is what lets the command read the stored values, and what it writes is the decrypted result: the IMAP password or the OAuth2 refresh token in cleartext. Redirect it to a file only where that file is protected the way a credential store would be, and delete it when the migration is done.
:::

---

## Encryption Management

Manage field-level encryption for stored secrets: account credentials, SMTP gateway passwords, OAuth2 application secrets, and the encrypted settings values.

### Encrypt Command

Migrate encryption settings. This command can be run from any machine with network access to the Redis database - it does not need to run on the EmailEngine server itself.

```bash
emailengine encrypt [options]
```

**Options:**

| Option             | Description                                               |
| ------------------ | --------------------------------------------------------- |
| `--service.secret` | New encryption secret. Leave out to remove encryption     |
| `--decrypt`        | Old secret for decrypting. Repeat the flag if values were encrypted with different secrets |
| `--dbs.redis`      | Redis connection (required)                               |

Running it with neither option prints the usage text and changes nothing.

**Use cases:**

1. **Enable encryption** (no encryption to encrypted)
2. **Disable encryption** (encrypted to plain text)
3. **Re-encrypt** (change encryption key)
4. **Migrate from multiple old keys**

**Examples:**

**Enable encryption:**

```bash
emailengine encrypt \
  --service.secret="new-encryption-key" \
  --dbs.redis="redis://127.0.0.1:6379/8"
```

**Change encryption key:**

```bash
emailengine encrypt \
  --service.secret="new-key" \
  --decrypt="old-key" \
  --dbs.redis="redis://127.0.0.1:6379/8"
```

**Migrate from multiple old keys:**

```bash
emailengine encrypt \
  --service.secret="new-key" \
  --decrypt="old-key-1" \
  --decrypt="old-key-2" \
  --decrypt="old-key-3" \
  --dbs.redis="redis://127.0.0.1:6379/8"
```

**Disable encryption:**

```bash
emailengine encrypt \
  --decrypt="current-key" \
  --dbs.redis="redis://127.0.0.1:6379/8"
```

The command reports each updated account, gateway, and OAuth2 app, and a value it cannot decrypt with any of the given secrets as `Check decryption secrets`.

**Important:**

- Back up your data before encryption changes
- All EmailEngine instances must use the same secret
- Changing keys requires access to the old key or keys

See [Field Encryption](/docs/advanced/encryption) for details.

---

## Redis Keyspace Scan

### Scan Command

Scan Redis and count keys by pattern:

```bash
emailengine scan --dbs.redis="redis://127.0.0.1:6379/8"
```

**Output:** CSV on stdout with a `KEY,COUNT` header. Account IDs, hashes, numbers, email addresses, and dates in key names are replaced with placeholders so that keys of the same kind are counted together; progress and the final `Checked N keys` line go to stderr.

**Example output:**

```csv
KEY,COUNT
"iad:hash(16)",150
"ia:accounts",1
"settings",1
"bull:submit:*",42
```

**Use cases:**

- Audit keyspace structure
- Capacity planning
- Debugging

---

## Bounce Email Analysis

### Check Bounce Command

Analyze an EML file to detect bounce information:

```bash
emailengine check-bounce [options]
```

**Options:**

| Option   | Short | Description                |
| -------- | ----- | -------------------------- |
| `--file` | `-f`  | Path to EML file to analyze |

**Examples:**

```bash
# Analyze a bounce email
emailengine check-bounce /path/to/bounce.eml

# Using the --file option
emailengine check-bounce --file /path/to/bounce.eml
emailengine check-bounce -f /path/to/bounce.eml
```

**Output:** a JSON object with the fields the bounce detector could extract:

- `recipient` - The email address that bounced
- `action` - The delivery action from the report, for example `failed` or `delayed`
- `response.message` - The error message from the remote server
- `response.status` - Enhanced status code, for example `5.1.1`
- `response.source` - Where the diagnostic code came from
- `response.category` - Classifier label, for example `user_unknown` or `mailbox_full`
- `response.recommendedAction` - One of `remove`, `retry`, `retry_different_ip`, `fix_configuration`, `review`, `remove_content`
- `response.blocklist` and `response.retryAfter` - Set when the classifier recognized a blocklist listing or a retry hint
- `mta` - The mail server that reported the failure
- `queueId` - Queue ID from the sending MTA, when present
- `messageId` - Message-ID of the original message

Fields that could not be determined are left out. A file that is not a bounce produces `{}`.

**Example output:**

```json
{
  "recipient": "user@example.com",
  "action": "failed",
  "response": {
    "message": "550 5.1.1 User unknown",
    "status": "5.1.1",
    "source": "smtp",
    "category": "user_unknown",
    "recommendedAction": "remove"
  },
  "mta": "mx.example.com",
  "messageId": "<original@example.com>"
}
```

**Use cases:**

- Debug delivery issues
- Classify bounce types
- Test bounce detection against a saved message

:::tip
This command uses the same bounce detection and classification as EmailEngine's automatic bounce handling, so its output matches what a [`messageBounce`](/docs/webhooks/messagebounce) webhook would carry for the same message.
:::

---

## Remote Administration Examples

### Scenario 1: Automated Deployment

Deploy EmailEngine with a pre-configured token:

```bash
#!/bin/bash

# Generate token on admin workstation
TOKEN=$(emailengine tokens issue \
  -d "Production API token" \
  -s "api" \
  --dbs.redis="redis://prod-redis.example.com:6379/0")

# Export for prepared settings
PREPARED=$(emailengine tokens export \
  -t "$TOKEN" \
  --dbs.redis="redis://prod-redis.example.com:6379/0")

# Deploy to production server
ssh prod-server "export EENGINE_PREPARED_TOKEN='$PREPARED' && emailengine"
```

### Scenario 2: CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
steps:
  - name: Create deployment token
    id: create-token
    run: |
      TOKEN=$(emailengine tokens issue \
        -d "CI deployment token" \
        -s "api" \
        --dbs.redis="${REDIS_URL}")
      echo "token=$TOKEN" >> "$GITHUB_OUTPUT"

  - name: Check the account list
    env:
      EMAILENGINE_TOKEN: ${{ steps.create-token.outputs.token }}
    run: |
      curl -fsS -H "Authorization: Bearer $EMAILENGINE_TOKEN" \
        https://emailengine.example.com/v1/accounts
```

### Scenario 3: Backup Automation

The CLI has no command that lists tokens or accounts, so the script needs the token values and account IDs as input. Account IDs can come from `GET /v1/accounts`.

```bash
#!/bin/bash
# backup-emailengine.sh

REDIS_URL="redis://backup-source.example.com:6379/0"
BACKUP_DIR="/backups/emailengine/$(date +%Y%m%d)"

mkdir -p "$BACKUP_DIR"

# Export the license
emailengine license export \
  --dbs.redis="$REDIS_URL" \
  > "$BACKUP_DIR/license.txt"

# Export tokens, one per line in tokens.txt
while read -r TOKEN; do
  emailengine tokens export \
    -t "$TOKEN" \
    --dbs.redis="$REDIS_URL" \
    >> "$BACKUP_DIR/tokens.txt"
  echo >> "$BACKUP_DIR/tokens.txt"
done < tokens.txt

# Export accounts, one ID per line in accounts.txt
while read -r ACCOUNT; do
  emailengine export \
    -a "$ACCOUNT" \
    --dbs.redis="$REDIS_URL" \
    --service.secret="$ENCRYPTION_SECRET" \
    > "$BACKUP_DIR/account-$ACCOUNT.json"
done < accounts.txt
```

### Scenario 4: Multi-Instance Management

Issue a token on several EmailEngine instances:

```bash
#!/bin/bash

INSTANCES=(
  "redis://instance1.example.com:6379/0"
  "redis://instance2.example.com:6379/0"
  "redis://instance3.example.com:6379/0"
)

# Each instance gets its own token value
for REDIS_URL in "${INSTANCES[@]}"; do
  echo "Creating token on $REDIS_URL"
  emailengine tokens issue \
    -d "Monitoring token" \
    -s "metrics" \
    --dbs.redis="$REDIS_URL"
  echo
done
```

## Common Patterns

### Pattern 1: Environment-Based Configuration

```bash
# dev.env
REDIS_URL=redis://localhost:6379/8
ENCRYPTION_SECRET=dev-secret

# prod.env
REDIS_URL=redis://prod-redis:6379/0
ENCRYPTION_SECRET=prod-secret-key

# Usage
source dev.env
emailengine tokens issue -d "Dev token" --dbs.redis="$REDIS_URL"

source prod.env
emailengine tokens issue -d "Prod token" --dbs.redis="$REDIS_URL"
```

### Pattern 2: Token Rotation Script

The CLI can issue tokens but not delete them, so revoke the old one with [`DELETE /v1/tokens/{token}`](/docs/api/delete-v-1-tokens-token) once the new one is in place.

```bash
#!/bin/bash
# rotate-tokens.sh

OLD_TOKEN="$1"
REDIS_URL="$2"

# Create the replacement
NEW_TOKEN=$(emailengine tokens issue \
  -d "Rotated token $(date +%Y%m%d)" \
  -s "api" \
  --dbs.redis="$REDIS_URL")

echo "New token: $NEW_TOKEN"

# Revoke the old one through the API, using the new token
curl -fsS -X DELETE \
  -H "Authorization: Bearer $NEW_TOKEN" \
  "https://emailengine.example.com/v1/tokens/$OLD_TOKEN"
```

### Pattern 3: Health Check

`/health` needs no token, and it checks the things a token would not: that every IMAP worker is up and that Redis answers a write and a read.

```bash
#!/bin/bash
# check-emailengine.sh

if curl -fsS https://emailengine.example.com/health > /dev/null; then
  echo "EmailEngine is healthy"
  exit 0
else
  echo "EmailEngine is unhealthy"
  exit 1
fi
```

Minting a token per check would work too, and would leave one behind on every run.

## See Also

- [Environment Variables](/docs/configuration/environment-variables) - The same settings as environment variables, plus those with no CLI equivalent
- [Configuration Overview](/docs/configuration) - How config files, environment variables, and CLI arguments combine
- [Prepared Settings](/docs/configuration/prepared-settings) - Provisioning license, token, and password without manual setup
- [Reset Password](/docs/configuration/reset-password) - The `password` command in detail
- [Access Tokens](/docs/api-reference/access-tokens) - What `tokens issue` produces and how scopes work
