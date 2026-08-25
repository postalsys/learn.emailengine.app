---
title: Command Line Interface (CLI)
description: Complete guide to using the EmailEngine command line interface for administration and automation
sidebar_position: 10
---

# EmailEngine CLI

The EmailEngine command line interface manages tokens, licenses, passwords, accounts, and encryption. It talks to Redis directly, so it can run anywhere that can reach the database, whether or not the server is running.

## Overview

### Remote Administration

**Important:** The EmailEngine CLI does not need to run on the same server as EmailEngine. You can run CLI commands from any location as long as you can:

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
emailengine tokens --help
```

#### Option 2: Download Binary

Download pre-built binaries from the official website:

**Download from:** https://emailengine.app

**Available formats:**

- Compiled binaries (Linux, macOS, Windows)
- Docker images
- Source distribution

The `emailengine` command is available in the downloaded package and can be run directly:

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
# Show help for specific commands
emailengine help tokens
emailengine help license
emailengine help check-bounce

# Alternative syntax
emailengine tokens --help
emailengine license --help
```

The help output automatically wraps text based on your terminal width for better readability.

### Version Information

```bash
emailengine --version
emailengine -v
emailengine version
```

**Output:**

```
EmailEngine v2.78.0 (LICENSE_EMAILENGINE)
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

- Account export/import with encrypted fields
- Encryption migration (`encrypt` command)
- Field-level encryption operations

**Not needed for:**

- Token operations (tokens are hashed, not encrypted)
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
| `--dbs.redis`        | `EENGINE_REDIS`            | Redis connection URL | Required    |
| `--api.host`         | `EENGINE_HOST`             | API server host      | `127.0.0.1` |
| `--api.port`         | `EENGINE_PORT`             | API server port      | `3000`      |
| `--workers.imap`     | `EENGINE_WORKERS`          | Account worker count | `4`         |
| `--workers.webhooks` | `EENGINE_WORKERS_WEBHOOKS` | Webhook worker count | `1`         |
| `--log.level`        | `EENGINE_LOG_LEVEL`        | Log level            | `trace`     |
| `--service.secret`   | `EENGINE_SECRET`           | Encryption secret    | Optional    |

### All Server Arguments

Run `emailengine --help` for the authoritative list. The full set:

**General**

| Option | Description | Default |
|--------|-------------|---------|
| `--dbs.redis` | Redis connection URL | Required |
| `--workers.imap` | Account worker threads | `4` |
| `--workers.webhooks` | Webhook worker threads | `1` |
| `--settings` | Pre-configured settings as a JSON string | None |
| `--service.secret` | Key for encrypting stored credentials | None |
| `--service.commandTimeout` | Maximum time for an IMAP command | `10s` |
| `--service.setupDelay` | Delay between assigning connections to workers | `0ms` |
| `--log.level` | Logging level | `trace` |
| `--log.raw` | Log raw IMAP traffic. Includes unmasked credentials | `false` |

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
| `--smtp.host` | Bind address | `127.0.0.1` |
| `--smtp.port` | Port | `2525` |
| `--smtp.secret` | Shared SMTP password accepted for all accounts | None |
| `--smtp.proxy` | Accept the HAProxy PROXY protocol | `false` |
| `--smtp.maxMessageSize` | Maximum accepted email size | `25M` |

**Document Store** (deprecated, removed from releases starting October 1, 2026)

| Option | Description | Default |
|--------|-------------|---------|
| `--documentStore.enabled` | Enable the Document Store feature gate | `false` |

:::tip Environment Variables
All CLI arguments can also be set as environment variables. See [Environment Variables reference](/docs/configuration/environment-variables) for complete list.
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
redis = "redis://redis-cluster.example.com:6379"

# API server configuration
[api]
host = "0.0.0.0"
port = 3000
proxy = true
maxSize = 20971520  # 20 MB

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
commandTimeout = 30000

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

When multiple configuration sources are used:

1. **Environment variables** (highest priority)
2. **CLI arguments**
3. **Configuration file**
4. **Default values** (lowest priority)

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
| `--description` | `-d`  | Token description     | Required |
| `--scope`       | `-s`  | Token scope           | `"*"`    |
| `--account`     | `-a`  | Account ID (optional) | None     |
| `--dbs.redis`   |       | Redis connection      | Required |

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

```
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

```
hKJpZNlAMzAxZThjNTFhZjgxM2Q3MzUxNTYzYTFlM2I1NjVkYmEzZWJjMzk4ZjI4OWZjNjgzN...
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
| `--token`     | `-t`  | Exported token data (base64) |
| `--dbs.redis` |       | Redis connection             |

**Example:**

```bash
emailengine tokens import \
  -t "hKJpZNlAMzAxZThjNTFhZjgxM2Q3MzUxNTYzYTFlM2I1NjVkYmEzZWJjMzk4ZjI4OWZjNjgzN..." \
  --dbs.redis="redis://127.0.0.1:6379/8"
```

**Output:**

```
Token was imported
```

**Important:** Use the exported base64 data, not the original token.

---

## License Management

Manage EmailEngine licenses.

### Show License

Display current license information:

```bash
emailengine license
```

**Output:**

```
EmailEngine License Information
Version: 1.0
Licensed to: Company Name
Valid until: 2025-12-31
```

### Export License

Export license for backup or transfer:

```bash
emailengine license export --dbs.redis="redis://127.0.0.1:6379/8"
```

**Output:**

```
eyJsaWNlbnNlIjoiZXlKaGJHY2lPaUpJVXpJMU5pSXNJblI1Y0NJNklrcFhWQ0o5...
```

### Import License

Import license key:

```bash
emailengine license import [options]
```

**Options:**

| Option        | Short | Description         |
| ------------- | ----- | ------------------- |
| `--license`   | `-l`  | Encoded license key |
| `--dbs.redis` |       | Redis connection    |

**Example:**

```bash
emailengine license import \
  -l "eyJsaWNlbnNlIjoiZXlKaGJHY2lPaUpJVXpJMU5pSXNJblI1Y0NJNklrcFhWQ0o5..." \
  --dbs.redis="redis://127.0.0.1:6379/8"
```

**Use cases:**

- Automated license deployment
- License updates
- Migration to new instance

See [Prepared License](/docs/configuration/prepared-settings/license) for automated setup.

---

## Password Management

Manage admin password for web interface.

### Set Password

Set or reset admin password:

```bash
emailengine password [options]
```

**Options:**

| Option        | Short | Description                                 |
| ------------- | ----- | ------------------------------------------- |
| `--password`  | `-p`  | Password to set (auto-generated if omitted) |
| `--hash`      | `-r`  | Return password hash for imports            |
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

```
a7f3c9e1d2b4f8a6
```

**Hash output (with `-r`):**

```
JDJhJDEyJGVYdEJ5Q3VrZXJlTGNXRkJ...
```

**Security notes:**

- Minimum 8 characters required
- Auto-generated passwords are 32 hex characters
- Password hash can be used in prepared settings
- Resets TOTP 2FA if enabled

See [Reset Password](/docs/configuration/reset-password) for details.

---

## Account Management

Export and manage account data.

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

**Output:** JSON with full account data including encrypted credentials.

**Use cases:**

- Backup specific accounts
- Migrate accounts between instances
- Disaster recovery
- Account auditing

**Security warning:** Exported data contains sensitive credentials. Store securely.

---

## Encryption Management

Manage field-level encryption for account credentials.

### Encrypt Command

Migrate encryption settings. This command can be run from any machine with network access to the Redis database - it does not need to run on the EmailEngine server itself.

```bash
emailengine encrypt [options]
```

**Options:**

| Option             | Description                                               |
| ------------------ | --------------------------------------------------------- |
| `--service.secret` | New encryption secret                                     |
| `--decrypt`        | Old secret(s) for decrypting (can be used multiple times) |
| `--dbs.redis`      | Redis connection (required)                               |

**Use cases:**

1. **Enable encryption** (no encryption → encrypted)
2. **Disable encryption** (encrypted → plain text)
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

**Important:**

- Backup your data before encryption changes
- All EmailEngine instances must use the same secret
- Changing keys requires access to old key(s)

See [Field Encryption](/docs/advanced/encryption) for details.

---

## Redis Keyspace Scan

Scan and analyze Redis keyspace usage.

### Scan Command

Scan Redis and display keyspace statistics:

```bash
emailengine scan --dbs.redis="redis://127.0.0.1:6379/8"
```

**Output:** CSV format with key patterns and statistics.

**Example output:**

```csv
Pattern,Count,TotalSize,AvgSize
ee:account:*,150,45000,300
ee:tokens,1,2048,2048
ee:settings,1,512,512
```

**Use cases:**

- Monitor Redis memory usage
- Identify large keys
- Audit keyspace structure
- Capacity planning
- Debugging

---

## Bounce Email Analysis

Analyze bounce emails to understand delivery failures.

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

**Output:** JSON object with bounce analysis including:

- `recipient` - The email address that bounced
- `action` - Type of bounce action (failed, delayed, etc.)
- `response` - Server response message
- `headers` - Relevant message headers
- `category` - ML-based classification of bounce type
- `recommendedAction` - Suggested action to take

**Example output:**

```json
{
  "recipient": "user@example.com",
  "action": "failed",
  "response": {
    "message": "550 5.1.1 User unknown",
    "status": "5.1.1"
  },
  "headers": {
    "from": "mailer-daemon@mail.example.com",
    "subject": "Delivery Status Notification (Failure)"
  },
  "category": "hard-bounce",
  "recommendedAction": "remove"
}
```

**Use cases:**

- Debug delivery issues
- Classify bounce types (hard bounce vs soft bounce)
- Automate bounce handling workflows
- Test bounce detection accuracy

:::tip
This command uses the same enhanced bounce detection engine as EmailEngine's automatic bounce handling, including support for Exim-style diagnostic messages, legacy bounce formats, and non-standard patterns.
:::

---

## Remote Administration Examples

### Scenario 1: Automated Deployment

Deploy EmailEngine with pre-configured token:

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
    run: |
      TOKEN=$(emailengine tokens issue \
        -d "CI deployment token" \
        -s "*" \
        --dbs.redis="${REDIS_URL}")
      echo "::set-output name=token::$TOKEN"

  - name: Run migrations
    env:
      EMAILENGINE_TOKEN: ${{ steps.create-token.outputs.token }}
    run: |
      # Run deployment scripts using token
      curl -H "Authorization: Bearer $EMAILENGINE_TOKEN" \
        http://emailengine.example.com/v1/accounts
```

### Scenario 3: Backup Automation

```bash
#!/bin/bash
# backup-emailengine.sh

REDIS_URL="redis://backup-source.example.com:6379/0"
BACKUP_DIR="/backups/emailengine/$(date +%Y%m%d)"

mkdir -p "$BACKUP_DIR"

# Export all tokens
emailengine tokens export \
  --dbs.redis="$REDIS_URL" \
  > "$BACKUP_DIR/tokens.txt"

# Export license
emailengine license export \
  --dbs.redis="$REDIS_URL" \
  > "$BACKUP_DIR/license.txt"

# Export accounts (if you have account list)
for ACCOUNT in $(cat accounts.txt); do
  emailengine export \
    -a "$ACCOUNT" \
    --dbs.redis="$REDIS_URL" \
    --service.secret="$ENCRYPTION_SECRET" \
    > "$BACKUP_DIR/account-$ACCOUNT.json"
done
```

### Scenario 4: Multi-Instance Management

Manage tokens across multiple EmailEngine instances:

```bash
#!/bin/bash

INSTANCES=(
  "redis://instance1.example.com:6379/0"
  "redis://instance2.example.com:6379/0"
  "redis://instance3.example.com:6379/0"
)

# Create same token on all instances
for REDIS_URL in "${INSTANCES[@]}"; do
  echo "Creating token on $REDIS_URL"
  emailengine tokens issue \
    -d "Shared monitoring token" \
    -s "metrics" \
    --dbs.redis="$REDIS_URL"
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

```bash
#!/bin/bash
# rotate-tokens.sh

OLD_TOKEN="$1"
REDIS_URL="$2"

# Export old token
EXPORTED=$(emailengine tokens export \
  -t "$OLD_TOKEN" \
  --dbs.redis="$REDIS_URL")

# Create new token with same settings
NEW_TOKEN=$(emailengine tokens issue \
  -d "Rotated token $(date +%Y%m%d)" \
  -s "api" \
  --dbs.redis="$REDIS_URL")

echo "Old token: $OLD_TOKEN"
echo "New token: $NEW_TOKEN"
echo "Exported: $EXPORTED"
```

### Pattern 3: Health Check

```bash
#!/bin/bash
# check-emailengine.sh

REDIS_URL="redis://localhost:6379/0"

# Create temporary token
TOKEN=$(emailengine tokens issue \
  -d "Health check $(date +%s)" \
  -s "api" \
  --dbs.redis="$REDIS_URL")

# Test API
if curl -f -H "Authorization: Bearer $TOKEN" \
  http://localhost:3000/v1/stats > /dev/null 2>&1; then
  echo "EmailEngine is healthy"
  exit 0
else
  echo "EmailEngine is unhealthy"
  exit 1
fi
```

## See Also

- [Environment Variables](/docs/configuration/environment-variables) - The same settings as environment variables, plus those with no CLI equivalent
- [Configuration Overview](/docs/configuration) - How config files, environment variables, and CLI arguments combine
- [Prepared Settings](/docs/configuration/prepared-settings) - Provisioning license, token, and password without manual setup
- [Reset Password](/docs/configuration/reset-password) - The `password` command in detail
- [Access Tokens](/docs/api-reference/access-tokens) - What `tokens issue` produces and how scopes work
