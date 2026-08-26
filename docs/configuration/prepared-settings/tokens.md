---
title: Prepared Tokens
sidebar_position: 1
description: Pre-configure API access tokens via environment variables for automated deployments
---

# Prepared Access Tokens

Pre-configure API access tokens for fully automated deployments without manual token generation through the web interface.

## Overview

This guide covers the CLI-based token workflow for:
- Automated deployments (Docker, Kubernetes, cloud platforms)
- Infrastructure-as-code setups
- CI/CD pipelines
- Automated testing environments

For the rest of the token surface, the admin interface, the API, scopes, and how requests authenticate, see [Access Tokens](/docs/api-reference/access-tokens). The CLI commands used here are documented option by option in the [CLI reference](/docs/configuration/cli#token-management).

## Why Prepared Tokens

Manual token generation through the web interface requires:
1. Starting EmailEngine
2. Logging into the web interface
3. Opening **Access Tokens** in the menu
4. Creating a token
5. Copying the token

Prepared tokens eliminate these manual steps, enabling fully automated infrastructure deployment.

## Issuing and Exporting a Token

The CLI talks to Redis directly, so the commands work whether or not EmailEngine is running. They need the Redis URL of the instance (`--dbs.redis`, or `EENGINE_REDIS` in the environment) and nothing else: tokens are stored as SHA-256 hashes, so `EENGINE_SECRET` is not involved. The CLI also works on an instance that has no admin password yet, which the admin interface refuses, and it needs no existing token, which `POST /v1/tokens` does (the credential-less `preauth` caller that the `disableTokens` setting lets through cannot mint one).

### 1. Issue a token

```bash
emailengine tokens issue \
  -d "Production API" \
  -s "*" \
  --dbs.redis="redis://127.0.0.1:6379/8"
```

**Output:** the token itself, 64 hexadecimal characters on stdout with no trailing newline:

```
f05d76644ea39c4a2ee33e7bffe55808b716a34b51d67b388c7d60498b0f89bc
```

This is the value API clients send in the `Authorization: Bearer` header. It is shown once; EmailEngine keeps only its hash.

`-s` picks the scope (`*`, `api`, `metrics`, `smtp`, `imap-proxy` or `mcp`, repeatable), `-d` sets the description and `-a` binds the token to one account. See [Token Scopes](/docs/api-reference/access-tokens#token-scopes) and [Account-Specific Tokens](/docs/api-reference/access-tokens#account-specific-tokens) for what each choice allows.

### 2. Export the token

The value `EENGINE_PREPARED_TOKEN` expects is not the token but its exported record:

```bash
emailengine tokens export \
  -t f05d76644ea39c4a2ee33e7bffe55808b716a34b51d67b388c7d60498b0f89bc \
  --dbs.redis="redis://127.0.0.1:6379/8"
```

**Output:**
```
hKJpZNlAMzAxZThjNTFhZjgxM2Q3MzUxNTYzYTFlM2I1NjVkYmEzZWJjMzk4ZjI4OWZjNjgzN
```

The exported string is the stored token record, serialized with MessagePack and base64url-encoded. It carries:
- The token hash (SHA-256 of the token)
- The scopes, the description, and the account binding if there is one
- The creation timestamp

It does not carry the token itself, which is why the record can sit in a deployment manifest: importing it makes the original token valid on the target instance, but the record alone cannot be used to authenticate.

### 3. Import the token

The same record can be loaded by hand, for example from an initialization script:

```bash
emailengine tokens import \
  -t hKJpZNlAMzAxZThjNTFhZjgxM2Q3MzUxNTYzYTFlM2I1NjVkYmEzZWJjMzk4ZjI4OWZjNjgzN \
  --dbs.redis="redis://127.0.0.1:6379/8"
```

**Output:**
```
Token was imported
```

If a token with the same hash already exists, the command prints `Token was not imported` and leaves the existing record alone. The creation timestamp of an imported token is the time of the import, not of the original issue.

## Account-Bound Tokens

The `-a` / `--account` flag binds a token to one account ID:

```bash
emailengine tokens issue \
  -d "user@example.com API Token" \
  -s "api" \
  -a "user@example.com"
```

A bound token can only call endpoints that name that account, and cannot list accounts or change settings. For the full rules see [Account-Specific Tokens](/docs/api-reference/access-tokens#account-specific-tokens).

## Prepared Token Configuration

Set the exported record as `EENGINE_PREPARED_TOKEN` (or `--preparedToken` on the command line, `preparedToken` in a TOML configuration file, or `EENGINE_PREPARED_TOKEN_FILE` to read it from a file). At startup EmailEngine decodes the record, checks that it carries a valid token hash, and imports it.

**How it works:**
1. Export a token using `emailengine tokens export -t TOKEN_VALUE`
2. Set the exported string as `EENGINE_PREPARED_TOKEN`
3. When EmailEngine starts, it imports the record if that hash is not stored yet
4. The original token authenticates API requests immediately

A malformed value is fatal: the log shows `Invalid API token provided` and the process exits with status 1, so a truncated secret stops the deployment rather than starting without the token.

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

<Tabs groupId="config-method">
<TabItem value="env" label="Environment Variable">

```bash
export EENGINE_PREPARED_TOKEN=hKJpZNlAMzAxZThjNTFhZjgxM2Q3MzUxNTYzYTFlM2I1NjVkYmEzZWJjMzk4ZjI4OWZjNjgzN
emailengine
```

**Startup log (debug level):**
```
{"level":20,"time":1638811265629,"msg":"Token imported","token":"301e8c51af813d7351563a1e3b565dba3ebc398f289fc683..."}
```

On later starts the same record logs `Token already exists` and nothing is written.

</TabItem>
<TabItem value="cli" label="Command-Line">

```bash
emailengine --preparedToken="hKJpZNlAMzAxZThjNTFhZjgxM2Q3MzUxNTYzYTFlM2I1NjVkYmEzZWJjMzk4ZjI4OWZjNjgzN"
```

</TabItem>
<TabItem value="docker" label="Docker">

```dockerfile
ENV EENGINE_PREPARED_TOKEN=hKJpZNlAMzAxZThjNTFhZjgxM2Q3MzUxNTYzYTFlM2I1NjVkYmEzZWJjMzk4ZjI4OWZjNjgzN
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
PREPARED_TOKEN=hKJpZNlAMzAxZThjNTFhZjgxM2Q3MzUxNTYzYTFlM2I1NjVkYmEzZWJjMzk4ZjI4OWZjNjgzN
```

</TabItem>
</Tabs>

## Complete Workflow

**1. Generate and export a token:**
```bash
# Issue the token (against a Redis database the CLI can reach)
TOKEN=$(emailengine tokens issue -d "Production API" -s "*" --dbs.redis="redis://localhost:6379/8")

echo "Generated token: $TOKEN"

# Export the record for prepared configuration
EXPORTED=$(emailengine tokens export -t "$TOKEN" --dbs.redis="redis://localhost:6379/8")

echo "Exported token: $EXPORTED"
```

The record can be issued on any Redis database, including a throwaway one: what ties it to an instance is the import, not the issue.

**2. Use it in the deployment:**
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
EXPORTED_TOKEN=hKJpZNlAMzAxZThjNTFhZjgxM2Q3MzUxNTYzYTFlM2I1NjVkYmEzZWJjMzk4ZjI4OWZjNjgzN
```

**3. Use the original token for API calls:**
```bash
curl https://emailengine.example.com/v1/accounts \
  -H "Authorization: Bearer $TOKEN"
```

## Multiple Prepared Tokens

`EENGINE_PREPARED_TOKEN` holds one record. To provision several tokens, import the others before starting the server; the import commands only need Redis:

```bash
# Initialization script
emailengine tokens import -t "$ADMIN_TOKEN" --dbs.redis="$EENGINE_REDIS"
emailengine tokens import -t "$METRICS_TOKEN" --dbs.redis="$EENGINE_REDIS"
emailengine tokens import -t "$USER1_TOKEN" --dbs.redis="$EENGINE_REDIS"

# Start EmailEngine
emailengine
```

Re-running the script is safe: a record whose hash is already stored is skipped.

## See Also

- [Command Line Interface (CLI)](/docs/configuration/cli#token-management) - Every option of `tokens issue`, `export` and `import`
- [Access Tokens](/docs/api-reference/access-tokens) - Token types, scopes, restrictions, and how requests authenticate
- [Prepared Settings](/docs/configuration/prepared-settings) - The other values that can be provisioned at startup
- [Prepared License](/docs/configuration/prepared-settings/license) - Pre-configure license keys for automated deployments
- [Security Best Practices](/docs/deployment/security) - Keeping tokens and other secrets out of images and logs
