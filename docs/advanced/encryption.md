---
title: Secret Encryption
sidebar_position: 3
description: Enable field-level encryption for sensitive data like passwords and OAuth tokens
---

# Secret Encryption

Learn how to enable field-level encryption for sensitive data stored by EmailEngine, including passwords, OAuth tokens, and API secrets.

## Overview

By default, EmailEngine stores all data in cleartext in Redis. This is fine for testing but not recommended for production environments.

EmailEngine offers **field-level encryption** that encrypts all sensitive fields using the **AES-256-GCM** cipher:

- Account passwords
- OAuth access and refresh tokens
- Google OAuth client secrets
- Other sensitive configuration values

## Why Enable Encryption?

### Security Benefits

1. **Data at Rest Protection**: Even if Redis is compromised, encrypted data remains secure
2. **Compliance**: Meets security requirements for many regulations
3. **Defense in Depth**: Additional security layer beyond network security
4. **Peace of Mind**: Production-ready security posture

### What Gets Encrypted

**Account credentials**:

- IMAP passwords
- SMTP passwords
- OAuth access tokens
- OAuth refresh tokens

**Configuration**:

- OAuth client secrets (Gmail, Outlook)
- SMTP server password
- Gmail service account key
- Document store password
- OpenAI API key

**Not encrypted**:

- Email content (not stored by default)
- Metadata (subject lines, senders, etc.)
- Account IDs
- Configuration settings

## Important Considerations

:::warning
To encrypt credentials that are already stored, run the encryption migration tool. Setting `EENGINE_SECRET` on its own only affects values written after that point, so existing credentials stay in cleartext.
:::

### How Encryption Works with Existing Data

When you enable `EENGINE_SECRET` on an instance with existing accounts:

- **Existing accounts continue working** - EmailEngine can read both encrypted and unencrypted credentials
- **Existing credentials remain unencrypted** - They are not automatically migrated
- **New accounts get encrypted credentials** - Any account added after enabling encryption stores credentials encrypted
- **OAuth2 tokens encrypt on renewal** - When EmailEngine refreshes an OAuth2 access token, the new tokens are stored encrypted
- **IMAP/SMTP passwords stay unencrypted** - Until you run the migration tool, these remain in cleartext

This means you can enable encryption without downtime, but for full protection you should run the `emailengine encrypt` migration tool to encrypt all existing credentials.

## Enabling Encryption on New Instance

If you don't have any email accounts set up yet, this is the easiest approach.

### 1. Set Encryption Secret

Create a `.env` file in your working directory:

```bash
echo "EENGINE_SECRET=your-secret-password-here" > .env
```

Or generate a random secret:

```bash
echo "EENGINE_SECRET=$(openssl rand -hex 32)" > .env
```

**Note:** EmailEngine automatically loads environment variables from `.env` file in the current working directory.

### 2. Start EmailEngine

```bash
emailengine
```

That's it! All new accounts will have encrypted secrets automatically.

### Environment Variable Best Practices

:::tip
Don't provide environment variables using the `export` command in production. Instead:

**SystemD Service**:

```ini
[Service]
Environment="EENGINE_SECRET=secret-password"
```

**Docker Compose**:

```yaml
services:
  emailengine:
    environment:
      - EENGINE_SECRET=secret-password
```

**Docker Run**:

```bash
docker run -e EENGINE_SECRET=secret-password postalsys/emailengine
```

**.env File**:

```bash
# .env file in working directory
EENGINE_SECRET=secret-password
```

:::

## Enabling Encryption on Existing Instance

If you already have email accounts configured, you need to encrypt existing data before enabling encryption.

### Process Overview

1. Stop EmailEngine
2. Run encryption migration tool
3. Start EmailEngine with encryption enabled

### Step-by-Step Instructions

#### 1. Stop EmailEngine

```bash
# SystemD
sudo systemctl stop emailengine

# Docker
docker stop emailengine

# PM2
pm2 stop emailengine

# Direct process
pkill emailengine
```

#### 2. Run Encryption Migration

The encryption migration tool is the same `emailengine` command with the `encrypt` argument. You can run this command from any machine that has network access to the Redis database.

```bash
emailengine encrypt \
  --dbs.redis="redis://localhost:6379/8" \
  --service.secret="your-secret-password-here"
```

Or using environment variables:

```bash
export EENGINE_SECRET="your-secret-password-here"
export EENGINE_REDIS="redis://localhost:6379/8"
emailengine encrypt
```

:::tip Run From Anywhere
The `encrypt` command only needs Redis connectivity. You can run it from your local machine, a CI/CD pipeline, or any server with access to the Redis database.
:::

The tool will:

- Connect to Redis
- Find all unencrypted secrets
- Encrypt them with the provided secret
- Store encrypted values back to Redis
- Exit

#### 3. Start EmailEngine

```bash
export EENGINE_SECRET="your-secret-password-here"
emailengine
```

**SystemD**:

```bash
sudo systemctl start emailengine
```

**Docker**:

```bash
docker start emailengine
```

## Changing Encryption Secret

### When to Change

- Suspected secret compromise
- Regular security rotation policy
- Security audit requirements
- Compliance regulations

### Process

#### 1. Stop EmailEngine

```bash
sudo systemctl stop emailengine
```

#### 2. Run Migration with Old and New Secret

```bash
emailengine encrypt \
  --dbs.redis="redis://localhost:6379/8" \
  --service.secret="new-secret-password" \
  --decrypt="old-secret-password"
```

This will:

- Decrypt using old secret
- Re-encrypt using new secret
- Store updated values

The command reports what it rotated, and it covers every store that holds an encrypted value:

```
Updated 42/42 accounts
Updated 3/3 SMTP gateways
Updated 2/2 OAuth2 apps
Updated 1 TLS private keys
```

Settings holding secrets are rotated as well, each reported individually.

:::warning Check the counts before starting EmailEngine again
A count lower than the total means something did not re-encrypt, and those values remain readable only with the **old** secret. Whatever owns them breaks on next use, with no self-healing path. Keep the old secret until every count reads `n/n`.

Rotating everything requires EmailEngine v2.77.0 or newer. Earlier versions silently skipped SMTP gateways entirely and dropped the `externalAccount` field of OAuth2 apps using workload identity federation, while still reporting success.
:::

#### 3. Start EmailEngine with New Secret

Update your EmailEngine configuration to use the new secret, then start:

```bash
sudo systemctl start emailengine
```

### Multiple Old Secrets

If you have accounts encrypted with different secrets (after a botched migration), you can provide multiple old secrets:

```bash
emailengine encrypt \
  --dbs.redis="redis://localhost:6379/8" \
  --service.secret="new-secret" \
  --decrypt="old-secret-1" \
  --decrypt="old-secret-2" \
  --decrypt="old-secret-3"
```

The tool will try each old secret until one works for each account.

## Disabling Encryption

### When to Disable

Generally not recommended for production, but valid for:

- Moving to development environment
- Testing unencrypted performance
- Troubleshooting encryption issues

### Process

#### 1. Stop EmailEngine

```bash
sudo systemctl stop emailengine
```

#### 2. Run Decryption Migration

Provide old secret with `--decrypt` but no new secret:

```bash
emailengine encrypt \
  --dbs.redis="redis://localhost:6379/8" \
  --decrypt="old-secret-password"
```

This decrypts all secrets and stores them in cleartext.

#### 3. Start EmailEngine Without Secret

Remove `EENGINE_SECRET` from your EmailEngine configuration, then start:

```bash
sudo systemctl start emailengine
```

## Secret Management Best Practices

### 1. Use Strong Secrets

```bash
# Generate strong random secret
openssl rand -base64 32

# Or use password generator
pwgen -s 64 1
```

**Requirements**:

- Minimum 32 characters
- Mix of uppercase, lowercase, numbers, symbols
- Not reused elsewhere
- Not based on dictionary words

### 2. Secret Rotation

Implement regular rotation schedule:

**Recommended schedule**:

- **High security**: Every 30-90 days
- **Normal security**: Every 6-12 months
- **After incidents**: Immediately

**Process**:

1. Generate new secret
2. Schedule maintenance window
3. Run migration (see "Changing Encryption Secret")
4. Update secret storage systems
5. Verify all services working
6. Document change

### 3. Backup Considerations

**Encrypted backups**: Redis backups contain encrypted data, but you MUST securely store:

- The encryption secret itself
- Recovery procedures
- Documentation of encryption status

**Without the secret**: Encrypted data is unrecoverable.

## Using Secret Management Systems

### HashiCorp Vault

```bash
#!/bin/bash
# Fetch secret from Vault
export EENGINE_SECRET=$(vault kv get -field=encryption_key secret/emailengine)
emailengine
```

### AWS Secrets Manager

```bash
#!/bin/bash
# Fetch from AWS Secrets Manager
export EENGINE_SECRET=$(aws secretsmanager get-secret-value \
  --secret-id emailengine/encryption-key \
  --query SecretString \
  --output text)
emailengine
```

### Kubernetes Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: emailengine-secrets
type: Opaque
stringData:
  encryption-key: your-secret-here
---
apiVersion: v1
kind: Pod
metadata:
  name: emailengine
spec:
  containers:
    - name: emailengine
      image: postalsys/emailengine
      env:
        - name: EENGINE_SECRET
          valueFrom:
            secretKeyRef:
              name: emailengine-secrets
              key: encryption-key
```

### Docker Secrets

```bash
# Create secret
echo "your-secret-password" | docker secret create ee_encryption_key -

# Use in service
docker service create \
  --name emailengine \
  --secret ee_encryption_key \
  --env EENGINE_SECRET_FILE=/run/secrets/ee_encryption_key \
  postalsys/emailengine
```

The `_FILE` suffix tells EmailEngine to read the secret from the specified file path. Most other environment variables accept the same suffix - see [Loading Values From Files](/docs/configuration/environment-variables#loading-values-from-files).

## Migration Planning

### Migration Steps

1. **Backup** Redis database

   ```bash
   redis-cli --rdb /backup/redis-backup-$(date +%Y%m%d).rdb
   ```

2. **Test in staging**

   ```bash
   # Restore backup to staging
   # Run migration
   # Verify functionality
   ```

3. **Schedule maintenance**

   - Choose low-traffic period
   - Allow 15-30 minutes for migration
   - Have team on standby

4. **Execute migration**

   ```bash
   sudo systemctl stop emailengine

   # If enabling encryption for the first time (no existing encryption):
   emailengine encrypt \
     --dbs.redis="redis://localhost:6379/8" \
     --service.secret="your-new-secret"

   # If changing an existing encryption secret:
   emailengine encrypt \
     --dbs.redis="redis://localhost:6379/8" \
     --service.secret="your-new-secret" \
     --decrypt="your-old-secret"

   sudo systemctl start emailengine
   ```

5. **Verify**

   - Check logs for errors
   - Test account connections
   - Verify emails sending/receiving
   - Monitor for issues

### Rollback Plan

If migration fails:

1. **Stop EmailEngine**

   ```bash
   sudo systemctl stop emailengine
   ```

2. **Restore Redis backup**

   ```bash
   redis-cli --rdb /backup/redis-backup-20231001.rdb
   ```

3. **Start without encryption**

   ```bash
   unset EENGINE_SECRET
   sudo systemctl start emailengine
   ```

4. **Investigate** issue before retrying

## Key Points

- Set `EENGINE_SECRET` before an instance stores any credentials, so nothing is ever written in the clear
- EmailEngine must be stopped while enabling, rotating, or removing the secret
- Keep the old secret until a rotation reports `n/n` for every store
- Back up Redis before any migration, and rehearse it against a copy first
- Store the secret where it survives the loss of the server: without it, every stored credential is unrecoverable

## See Also

- [Environment Variables](/docs/configuration/environment-variables) - `EENGINE_SECRET` and the `_FILE` form for mounted secrets
- [CLI Reference](/docs/configuration/cli) - Full options for the `encrypt` command
- [Security Hardening](/docs/deployment/security) - The other half of protecting an instance
- [Compliance](/docs/deployment/compliance) - What EmailEngine stores, encrypted and not
- [Redis Configuration](/docs/configuration/redis) - Persistence and access control for the store holding this data
