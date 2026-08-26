---
title: Configuration Overview
description: Configure EmailEngine with environment variables, command-line arguments, and runtime settings
sidebar_position: 1
---

# EmailEngine Configuration

EmailEngine provides flexible configuration options to adapt to various deployment scenarios. This guide covers the configuration methods, precedence, and best practices.

## Configuration Types

EmailEngine uses two distinct types of configuration:

### 1. Application Configuration

**Loaded at startup** and cannot be changed without restarting the application.

**Examples:**

- HTTP server port
- Redis connection URL
- Encryption secrets
- Log levels

**Configure via:**

- [Environment variables](/docs/configuration/environment-variables) (recommended)
- [Command-line arguments](/docs/configuration/cli)
- [Configuration files](/docs/configuration/cli) (TOML)

### 2. Runtime Configuration

**Can be updated** at any time via the Settings API or web interface.

**Examples:**

- Webhook URLs
- Webhook event filters
- OAuth2 application credentials
- Email templates

**Configure via:**

- Web interface (the Configuration section of the admin dashboard)
- [Settings API endpoint](/docs/api/post-v-1-settings)
- [Prepared settings](/docs/configuration/prepared-settings) (environment variable)

## Configuration Methods

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

<Tabs groupId="config-method">
<TabItem value="env" label="Environment Variables">

**Recommended for production deployments.**

Create a `.env` file in the working directory:

```bash
# .env file
EENGINE_HOST=0.0.0.0
EENGINE_PORT=3000
EENGINE_REDIS=redis://localhost:6379
EENGINE_SECRET=your-secret-at-least-32-chars
```

EmailEngine automatically loads environment variables from `.env` file in the current working directory:

```bash
emailengine
```

**Docker Compose:**

```yaml
services:
  emailengine:
    image: postalsys/emailengine:v2
    environment:
      - EENGINE_HOST=0.0.0.0
      - EENGINE_PORT=3000
      - EENGINE_REDIS=redis://redis:6379
```

`REDIS_URL` is accepted as a fallback when `EENGINE_REDIS` is unset, and `PORT` as a fallback for `EENGINE_PORT`, which is what makes EmailEngine work unchanged on platforms that inject those variables. Most variables can also be supplied as a file path by appending `_FILE` to the name, the usual way to mount secrets in containers; see [Loading values from files](/docs/configuration/environment-variables#loading-values-from-files) for the few that cannot.

:::tip Interchangeable Configuration
Environment variables and CLI arguments can be used together. Environment variables take precedence over CLI arguments. See the [mapping table](/docs/configuration/environment-variables#environment-variable-to-cli-mapping) for equivalents.
:::

[Complete environment variables reference →](./environment-variables.md)

</TabItem>
<TabItem value="cli" label="Command-Line Arguments">

**Useful for development and testing.**

```bash
emailengine \
  --dbs.redis="redis://localhost:6379" \
  --api.port=3000 \
  --api.host="0.0.0.0" \
  --log.level="trace"
```

:::tip Interchangeable Configuration
Environment variables and CLI arguments can be used together. Environment variables take precedence over CLI arguments. See the [mapping table](/docs/configuration/environment-variables#environment-variable-to-cli-mapping) for equivalents.
:::

[Complete CLI reference →](/docs/configuration/cli)

</TabItem>
<TabItem value="toml" label="Configuration Files">

**TOML configuration files for persistent settings.**

Create a TOML configuration file with your settings:

```toml
# config.toml
[dbs]
redis = "redis://localhost:6379"

[api]
host = "0.0.0.0"
port = 3000

[log]
level = "info"

[service]
secret = "your-encryption-secret"
```

**Load configuration file:**

```bash
emailengine --config=/path/to/config.toml
```

[TOML configuration guide →](/docs/configuration/cli#configuration-files)

</TabItem>
</Tabs>

### Settings API

**For runtime configuration.** (See: [Settings API](/docs/api/post-v-1-settings))

```bash
curl -X POST https://emailengine.example.com/v1/settings \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "webhooks": "https://your-app.com/webhook",
    "webhookEvents": ["messageNew", "messageSent"]
  }'
```

The response lists the keys that were written, as `{"updated": ["webhooks", "webhookEvents"]}`.

### Web Interface

Runtime settings are also editable in the admin interface:

1. Open the admin interface at your service URL and log in
2. Open **Configuration** in the menu and pick the page for the setting (General, Webhooks, and so on)
3. Change the value and click **Save Changes**

## Configuration Precedence

When multiple configuration methods are used, they follow this precedence (highest to lowest):

1. **`EENGINE_*` environment variables** (highest priority)
2. **Command-line arguments**
3. **`APPCONF_*` environment variables**, a generic form the configuration loader accepts for any key: `APPCONF_workers_imap=8` is `--workers.imap=8`. Underscores become dots, so a key that itself contains an underscore cannot be set this way
4. **Configuration file**, given with `--config` or the `NODE_CONFIG_PATH` environment variable
5. **Default values** (lowest priority, from the bundled `config/default.toml`)

The configuration file and the command line can set any key; environment variables cover the keys listed in the [mapping table](/docs/configuration/environment-variables#environment-variable-to-cli-mapping) and the other `EENGINE_*` variables on the environment variables page. A key with no environment variable, such as `smtp.maxMessageSize`, is set on the command line or in the file.

Runtime settings are not part of this order. They live in Redis and are read from there, so an environment variable that seeds one has an effect only on the first start, before a stored value exists. See [Configuration Options Reference](/docs/reference/configuration-options) for those.

**Example:**

```bash
# config.toml has: port = 3000
# CLI argument: --api.port=4000
# Environment variable: EENGINE_PORT=5000

emailengine --config=config.toml --api.port=4000

# Result: Port 5000 (environment variable wins)
```

**Another example:**

```bash
# config.toml has: port = 3000
# CLI argument: --api.port=4000

emailengine --config=config.toml --api.port=4000

# Result: Port 4000 (CLI argument wins over config file)
```

## Configuration Best Practices

### Production Deployments

**Use environment variables:**

```yaml
environment:
  - EENGINE_HOST=0.0.0.0
  - EENGINE_PORT=3000
  - EENGINE_REDIS=redis://redis:6379
  - EENGINE_PREPARED_PASSWORD=${ADMIN_PASSWORD_HASH}
  - EENGINE_PREPARED_LICENSE=${LICENSE_KEY}
```

:::warning Password Hash Required
`EENGINE_PREPARED_PASSWORD` requires a **password hash**, not a plain password. Generate it with:
```bash
emailengine password -p "your-password" --hash --dbs.redis="redis://127.0.0.1:6379/8"
```

The command writes the password to the Redis database it is pointed at as well as printing the hash, so run it against the instance's own database or a scratch one. See [Reset Password](/docs/configuration/reset-password).
:::

**Keep secrets secure:**

- Never commit secrets to version control
- Use secret management systems (AWS Secrets Manager, HashiCorp Vault)
- Use `.env` files only for development
- Rotate secrets regularly

**Document your configuration:**

```bash
# .env.example (commit this)
EENGINE_HOST=0.0.0.0
EENGINE_PORT=3000
EENGINE_REDIS=redis://localhost:6379
# Generate hash: emailengine password -p "your-password" --hash
EENGINE_PREPARED_PASSWORD=JHBia2RmMi1zaGEyNTYkaT02MDAwMDAk...
```

### Development Setup

**Use command-line arguments for flexibility:**

```bash
emailengine \
  --dbs.redis="redis://localhost:6379/8" \
  --api.port=3001 \
  --log.level="trace"
```

**Or local `.env` file:**

```bash
# .env (don't commit)
EENGINE_PORT=3001
EENGINE_REDIS=redis://localhost:6379/8
EENGINE_LOG_LEVEL=trace
```

### Docker Deployments

**Use Docker Compose environment variables:**

```yaml
services:
  emailengine:
    image: postalsys/emailengine:v2
    env_file:
      - .env.production
    environment:
      - EENGINE_REDIS=redis://redis:6379
```

**Multi-environment setup:**

```
.env.development
.env.staging
.env.production
```

### Kubernetes Deployments

**Use ConfigMaps and Secrets:**

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: emailengine-config
data:
  EENGINE_HOST: "0.0.0.0"
  EENGINE_PORT: "3000"
  EENGINE_REDIS: "redis://redis-service:6379"

---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: emailengine-secrets
type: Opaque
stringData:
  EENGINE_PREPARED_PASSWORD: "JHBia2RmMi1zaGEyNTYkaT02MDAwMDAk..."
  EENGINE_PREPARED_LICENSE: "your-license-key"

---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: emailengine
spec:
  template:
    spec:
      containers:
        - name: emailengine
          image: postalsys/emailengine:v2
          envFrom:
            - configMapRef:
                name: emailengine-config
            - secretRef:
                name: emailengine-secrets
```

## Quick Reference

### Essential Configuration

**Minimal production setup:**

| Setting     | Environment Variable | Description              |
| ----------- | -------------------- | ------------------------ |
| Redis URL   | `EENGINE_REDIS`      | Redis connection string  |
| Server Host | `EENGINE_HOST`       | Listen address (default: 127.0.0.1) |
| Server Port | `EENGINE_PORT`       | HTTP port (default 3000) |

**Example:**

```bash
EENGINE_REDIS=redis://localhost:6379
EENGINE_HOST=0.0.0.0
EENGINE_PORT=3000
```

### Common Configuration Scenarios

**Behind Reverse Proxy:**

```bash
EENGINE_HOST=127.0.0.1
EENGINE_PORT=3000
```

**With Redis Key Prefix:**

```bash
EENGINE_REDIS=redis://localhost:6379/8
EENGINE_REDIS_PREFIX=ee-prod
```

**Verbose logging:**

```bash
EENGINE_LOG_LEVEL=trace
```

## Configuration Categories

### Server & Connection

Configure HTTP server, base URL, and proxy settings.

[View details →](./environment-variables.md#server--connection)

### Redis

Redis connection, clustering, and persistence.

[View details →](./redis.md)

### Email Protocol Settings

Email handling, attachment size limits, timeouts.

[View details →](./environment-variables.md#email-protocol-settings)

### Worker Threads

Worker thread configuration for processing workload.

[View details →](./environment-variables.md#worker-threads)

### Queue Management

Job queue retention and cleanup configuration.

[View details →](./environment-variables.md#queue-management)

### OAuth2

OAuth2 provider credentials and configuration.

[View details →](/docs/accounts/oauth2-setup)

### TLS Configuration

TLS/SSL settings for secure connections.

[View details →](./environment-variables.md#tls-configuration)

### Logging & Monitoring

Log levels, metrics endpoints, monitoring.

[View details →](/docs/advanced/monitoring)

### Prepared Configuration

Pre-configured settings, tokens, and licenses.

[View details →](/docs/configuration/prepared-settings)

## Validation

### Check Configuration

**View current settings via API:** (See: [Get Settings](/docs/api/get-v-1-settings))

```bash
curl https://emailengine.example.com/v1/settings \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

The startup configuration (ports, worker counts, Redis URL) is not part of that response. The **Workers** page in the admin menu shows the configured worker counts, and the first log line at startup, `EmailEngine starting up`, records the version and Node.js runtime.

**Check application config:**

```bash
# View logs for configuration issues
docker logs emailengine | grep -i config
```

### Common Issues

**Port already in use:**

```
Error: listen EADDRINUSE: address already in use :::3000
```

**Solution:** Change `EENGINE_PORT` to unused port.

**Redis connection failed:**

```
Error: connect ECONNREFUSED 127.0.0.1:6379
```

**Solution:** Verify `EENGINE_REDIS` is correct and Redis is running.

**Encrypted data cannot be decrypted:**

EmailEngine does not enforce a minimum length for `EENGINE_SECRET`, but the secret must stay the same across restarts. If the secret changes, previously encrypted values (OAuth2 tokens, passwords) can no longer be decrypted.

**Solution:** Use a single, stable secret for the lifetime of the deployment (a 32-byte random value such as `openssl rand -hex 32` is recommended) and store it securely.

### Generate Secrets

**Random secret key:**

```bash
# OpenSSL
openssl rand -hex 32

# /dev/urandom
head -c 32 /dev/urandom | base64

# Python
python3 -c "import secrets; print(secrets.token_hex(32))"
```

## Migration & Updates

When upgrading EmailEngine:

1. **Review the [changelog](https://github.com/postalsys/emailengine/blob/master/CHANGELOG.md)** for upgrade notes
2. **Back up Redis** (see [Redis](/docs/configuration/redis#backups))
3. **Test in staging** with the same prepared configuration
4. **Deploy to production**

Runtime settings that a release retires are dropped from `EENGINE_SETTINGS` with an error log line naming the ignored keys (since v2.79.1); the instance still starts.

## See Also

- [Environment variables](/docs/configuration/environment-variables) - Every startup variable and its CLI equivalent
- [CLI reference](/docs/configuration/cli) - Commands, arguments, and TOML files
- [Prepared settings](/docs/configuration/prepared-settings) - Provisioning runtime settings at first start
- [Redis](/docs/configuration/redis) - Connection URLs, persistence, and memory policy
- [Settings API](/docs/api/post-v-1-settings) - Changing runtime settings on a live instance
