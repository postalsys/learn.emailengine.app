---
title: Prepared Tokens
sidebar_position: 1
description: Pre-configure API access tokens via environment variables for automated deployments
---

# Prepared Access Tokens

Pre-configure API access tokens for fully automated deployments without manual token generation through the web interface.

## Overview

This guide covers the CLI-based token management workflow for:
- Automated deployments (Docker, Kubernetes, cloud platforms)
- Infrastructure-as-code setups
- CI/CD pipelines
- Automated testing environments

For the rest of the token surface, the admin interface, the API, scopes, and how requests authenticate, see [Access Tokens](/docs/api-reference/access-tokens).

## Why Prepared Tokens

Manual token generation through the web interface requires:
1. Starting EmailEngine
2. Logging into web interface
3. Navigating to Access Tokens
4. Clicking "Create new"
5. Copying the token

Prepared tokens eliminate these manual steps, enabling fully automated infrastructure deployment.

## Token Management via CLI

EmailEngine CLI provides commands for token management.

:::tip CLI Documentation
For complete CLI usage, installation, configuration options, and advanced examples, see the [Command Line Interface (CLI)](/docs/configuration/cli) documentation.
:::

### 1. Issue New Token

Generate a new token directly:

```bash
emailengine tokens issue \
  -d "API Token" \
  -s "*" \
  --dbs.redis="redis://127.0.0.1:6379"
```

**Output:**
```
f05d76644ea39c4a2ee33e7bffe55808b716a34b51d67b388c7d60498b0f89bc
```

**Options:**

| Flag | Long Form | Description | Default |
|------|-----------|-------------|---------|
| `-d` | `--description` | Token description | - |
| `-s` | `--scope` | Token scope | `*` |
| `-a` | `--account` | Bind to specific account | none |

**Scopes:**
- `*` - Full access (all operations, all scopes)
- `api` - Standard API calls (account operations, messages, mailboxes, etc.)
- `smtp` - SMTP proxy access only
- `imap-proxy` - IMAP proxy access only
- `metrics` - Prometheus metrics endpoint (`/metrics`) access only

For detailed scope descriptions and security implications, see [Token Scopes](/docs/api-reference/access-tokens#token-scopes).

**Examples:**

```bash
# Full access token
emailengine tokens issue -d "Production API" -s "*"

# API-only token
emailengine tokens issue -d "API Access" -s "api"

# Account-specific token (see Account-Bound Tokens section below)
emailengine tokens issue -d "User Token" -s "api" -a "user@example.com"
```

**Important:** When running token CLI commands, database settings must be provided, but encryption keys are NOT needed (tokens are hashed, not encrypted).

### 2. Export Token

Export an existing token for import elsewhere:

```bash
emailengine tokens export -t f05d76644ea39c4a2...
```

**Output:**
```
hKJpZNlAMzAxZThjNTFhZjgxM2Q3MzUxNTYzYTFlM2I1NjVkYmEzZWJjMzk4ZjI4OWZjNjgzN...
```

The exported string contains:
- Token hash
- Token metadata (description, scope, account)
- Creation timestamp

**Use cases:**
- Share token between EmailEngine instances
- Backup token for prepared configuration
- Migrate token to different environment

### 3. Import Token

Import a previously exported token:

```bash
emailengine tokens import -t hKJpZNlAMzAxZThjNTFhZjgxM2Q3MzUxN...
```

**Output:**
```
Token was imported
```

The original token value (the actual API key) is preserved during import.

## Account-Bound Tokens

The `-a` / `--account` flag creates tokens bound to a specific account, restricting their usage to operations on that single account only.

**Examples:**

```bash
# API access for specific account only
emailengine tokens issue \
  -d "user@example.com API Token" \
  -s "api" \
  -a "user@example.com"

# SMTP access for specific account
emailengine tokens issue \
  -d "user@example.com SMTP" \
  -s "smtp" \
  -a "user@example.com"

# Combined API and SMTP access for one account
emailengine tokens issue \
  -d "user@example.com Full Access" \
  -s "*" \
  -a "user@example.com"
```

**Key characteristics:**
- Token can ONLY access the bound account's data
- Cannot access system-wide endpoints or other accounts
- Ideal for multi-tenant applications and customer self-service portals
- Limits security blast radius if token is compromised

For complete details on account-specific tokens, restrictions, and use cases, see [Account-Specific Tokens](/docs/api-reference/access-tokens#account-specific-tokens).

## Prepared Token Configuration

Load a token automatically on startup using the exported token string. When EmailEngine starts, it checks for the `EENGINE_PREPARED_TOKEN` environment variable (or `--preparedToken` CLI flag) and automatically imports the token into the system without requiring manual web interface interaction.

**How it works:**
1. Export a token using `emailengine tokens export -t TOKEN_VALUE`
2. Set the exported string as `EENGINE_PREPARED_TOKEN` environment variable
3. When EmailEngine starts, it automatically imports the token
4. The token is immediately available for API authentication

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

<Tabs groupId="config-method">
<TabItem value="env" label="Environment Variable">

```bash
export EENGINE_PREPARED_TOKEN=hKJpZNlAMzAxZThjNTFhZjgxM2Q3MzUxN...
emailengine
```

**Output:**
```
{"level":20,"time":1638811265629,"msg":"Token imported"}
```

</TabItem>
<TabItem value="cli" label="Command-Line">

```bash
emailengine --preparedToken="hKJpZNlAMzAxZThjNTFhZjgxM2Q3MzUxN..."
```

</TabItem>
<TabItem value="docker" label="Docker">

```dockerfile
ENV EENGINE_PREPARED_TOKEN=hKJpZNlAMzAxZThjNTFhZjgxM2Q3MzUxN...
```

</TabItem>
<TabItem value="docker-compose" label="Docker Compose">

```yaml
services:
  emailengine:
    image: postalsys/emailengine:v2
    environment:
      - EENGINE_PREPARED_TOKEN=${PREPARED_TOKEN}
```

```bash
# .env file
PREPARED_TOKEN=hKJpZNlAMzAxZThjNTFhZjgxM2Q3MzUxN...
```

</TabItem>
</Tabs>

## Complete Workflow

**1. Generate and export token:**
```bash
# Generate token
TOKEN=$(emailengine tokens issue -d "Production API" -s "*" --dbs.redis="redis://localhost:6379")

echo "Generated token: $TOKEN"

# Export token for prepared config
EXPORTED=$(emailengine tokens export -t $TOKEN --dbs.redis="redis://localhost:6379")

echo "Exported token: $EXPORTED"
```

**2. Use in deployment:**
```yaml
# docker-compose.yml
services:
  emailengine:
    image: postalsys/emailengine:v2
    environment:
      - EENGINE_PREPARED_TOKEN=${EXPORTED_TOKEN}
```

```bash
# .env
EXPORTED_TOKEN=hKJpZNlAMzAxZThjNTFhZjgxM2Q3MzUxN...
```

**3. Use the original token for API calls:**
```bash
curl http://localhost:3000/v1/accounts \
  -H "Authorization: Bearer $TOKEN"
```

## Multiple Prepared Tokens

To prepare multiple tokens:

1. Generate and export each token
2. Import them sequentially on startup
3. Or combine in initialization script

**Example:**
```bash
# Initialization script
emailengine tokens import -t $ADMIN_TOKEN
emailengine tokens import -t $METRICS_TOKEN
emailengine tokens import -t $USER1_TOKEN

# Start EmailEngine
emailengine
```

## See Also

- [Command Line Interface (CLI)](/docs/configuration/cli) - Complete CLI reference for token management and administration
- [Access Tokens](/docs/api-reference/access-tokens) - Complete token guide (types, creation methods, security, authentication)
- [Prepared Settings](/docs/configuration/prepared-settings) - Pre-configure runtime settings via environment variables
- [Prepared License](/docs/configuration/prepared-settings/license) - Pre-configure license keys for automated deployments
- [Docker Deployment](/docs/deployment) - Containerized deployment with prepared configuration
- [Security Best Practices](/docs/deployment/security) - Security guidelines for production deployments
