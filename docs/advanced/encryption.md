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
- OAuth2 application client secrets and service account keys
- The settings and TLS private keys listed below

## Why Enable Encryption?

### Security Benefits

1. **Data at rest protection**: A copy of the Redis database, or of its backups, does not expose the stored credentials without the secret
2. **Compliance**: Encryption of stored credentials is a requirement of many security standards
3. **Defense in depth**: An additional layer beyond network access control on Redis

### What Gets Encrypted

**Account credentials**:

- IMAP passwords
- SMTP passwords
- OAuth access tokens
- OAuth refresh tokens

**OAuth2 applications** (`clientSecret`, `serviceKey`, `externalAccount` and `accessToken` in the app record):

- Client secrets
- Service account keys and external-account (workload identity federation) configurations
- The app-level access token of an application-access (client credentials) app

**SMTP gateways**:

- Gateway passwords

**Settings** (`smtpServerPassword`, `imapProxyServerPassword`, `serviceSecret`, `cookiePassword`, `totpSeed`, `openAiAPIKey`, `documentStorePassword`, and the legacy `gmailClientSecret`, `outlookClientSecret`, `mailRuClientSecret`, `gmailServiceKey` and `gmailServiceExternalAccount` values):

- The SMTP server and IMAP proxy global passwords
- The `serviceSecret` used for signing, the admin session cookie password and the admin TOTP seed
- The OpenAI API key and the Document Store password

**TLS private keys**:

- The ACME account key and the private key of every certificate EmailEngine provisions for its own listeners

**Not encrypted**:

- Email content (not stored by default)
- Metadata (subject lines, senders, etc.)
- Account IDs, and the rest of the account record: the IMAP and SMTP host names, the account's webhook URL and its custom headers, including an authorization header set there
- Every other setting
- Access tokens, which are not stored at all: only their SHA-256 hashes are, so the stored value cannot be used as a token

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
- **IMAP/SMTP passwords stay unencrypted** - They are encrypted the next time the account's credentials are saved, or when you run the migration tool; until then they remain in cleartext

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

**Note:** EmailEngine loads environment variables from a `.env` file in the current working directory (through `dotenv`), so this file is read on the next start.

### 2. Start EmailEngine

```bash
emailengine
```

Every credential stored from now on is encrypted.

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
- Find all unencrypted secrets in every store listed under [What Gets Encrypted](#what-gets-encrypted)
- Encrypt them with the provided secret
- Store encrypted values back to Redis
- Exit

`EENGINE_SECRET_FILE` and `EENGINE_REDIS_FILE` work here the same way as for the server, so the secret can be read from a mounted file rather than passed on the command line.

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

The command reports what it rotated, and it covers every store that holds an encrypted value. Settings holding secrets come first, one line per setting that changed, then one line per account, gateway, app and certificate entry that was rewritten, each store closing with a count:

```
smtpServerPassword: Updated setting value
user123: updated
user456: updated
Updated 2/2 accounts
Gateway sendgrid: updated
Updated 1/1 SMTP gateways
OAuth2 App AAABhaBPHscAAAAI: updated
Updated 1/1 OAuth2 apps
Certificate entry domain:emailengine.example.com:privateKey: updated
Updated 1 TLS private keys
```

The first number in each count is how many records were rewritten, the second how many exist. A record that held nothing to change, because it stores no secret or was already encrypted with the new secret, is not counted, so a lower first number is not an error on its own.

:::warning Check for "Could not process" lines before starting EmailEngine again
A value that none of the supplied secrets could decrypt is reported on stderr as `Could not process "imap.auth.pass" for user123. Check decryption secrets.` (the field and record vary) and is left untouched, so it remains readable only with the **old** secret. Whatever owns it breaks on next use, with no self-healing path. Keep the old secret until a run completes without such lines.

Rotating everything requires EmailEngine v2.77.0 or newer. Earlier versions reported `Updated 0/0 SMTP gateways` because they read the wrong index and never visited a gateway, left the `externalAccount` field of OAuth2 apps using workload identity federation under the old secret, and never touched the TLS private keys, while exiting successfully.
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

The tool takes the encryption secret from `EENGINE_SECRET` first, including the value a `.env` file in the working directory sets, and falls back to `--service.secret`. Clear `EENGINE_SECRET` from the environment and from `.env` before this run, otherwise the values are re-encrypted with it instead of being written back in cleartext.

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

EmailEngine derives the AES-256 key from the secret with scrypt (Node.js defaults: N=16384, r=8, p=1) and a random 16-byte salt per stored value, and does not enforce a minimum length. Treat a 32-byte random value as the floor, and do not reuse the secret anywhere else.

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
   - The tool rewrites one Redis hash per account, gateway and app, so the run is short even for large instances, but EmailEngine is stopped for its duration
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

   `redis-cli --rdb` only takes a snapshot; it cannot load one. Restoring means stopping Redis, replacing its `dump.rdb` with the backup, and starting Redis again. See [Redis Configuration](/docs/configuration/redis) for where that file lives on your install.

3. **Start without encryption**

   ```bash
   unset EENGINE_SECRET
   sudo systemctl start emailengine
   ```

4. **Investigate** issue before retrying

## Key Points

- Set `EENGINE_SECRET` before an instance stores any credentials, so nothing is ever written in the clear
- EmailEngine must be stopped while enabling, rotating, or removing the secret
- Keep the old secret until a rotation completes without a `Could not process` line
- Back up Redis before any migration, and rehearse it against a copy first
- Store the secret where it survives the loss of the server: without it, every stored credential is unrecoverable

## See Also

- [Environment Variables](/docs/configuration/environment-variables) - `EENGINE_SECRET` and the `_FILE` form for mounted secrets
- [CLI Reference](/docs/configuration/cli) - Full options for the `encrypt` command
- [Security Hardening](/docs/deployment/security) - The other half of protecting an instance
- [Compliance](/docs/deployment/compliance) - What EmailEngine stores, encrypted and not
- [Redis Configuration](/docs/configuration/redis) - Persistence and access control for the store holding this data
