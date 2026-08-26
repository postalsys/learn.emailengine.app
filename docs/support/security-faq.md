---
title: Credential Security FAQ
sidebar_position: 2
description: How EmailEngine stores and protects email account credentials, access tokens, and admin sessions, and what it logs and sends out
---

# Credential Security FAQ

Common questions about how EmailEngine stores, secures, and encrypts email account credentials, and about the other secrets an instance holds.

## Where are email credentials stored?

EmailEngine stores every credential it holds in **Redis**:

- IMAP and SMTP passwords
- OAuth2 access tokens and refresh tokens
- OAuth2 application client secrets, service account keys, and external account configurations
- SMTP gateway passwords
- The SMTP server and IMAP proxy passwords, the OpenAI API key, the TOTP seed, and the admin session cookie key

## Is email content stored?

Not as a copy of the mailbox. EmailEngine reads messages from the mail server on demand and keeps only sync state (message IDs, flags, folder listings) in Redis.

Content does pass through Redis in two cases:

- A submitted message is stored in Redis, in full, from submission until it is delivered, so that the queue can retry it. It is removed once delivery succeeds or the attempts run out
- Webhook payloads waiting in the notify queue carry whatever the payload includes; with `notifyText` or `notifyAttachments` enabled, that is message content

Neither is encrypted by `EENGINE_SECRET`, which covers credentials only. The deprecated Document Store, which indexed message content in Elasticsearch, is removed from releases starting 2026-10-01.

## Are credentials encrypted?

**It depends on your configuration:**

| Configuration | Storage Method | Security Level |
|--------------|----------------|----------------|
| Without `EENGINE_SECRET` | Cleartext | Development only |
| With `EENGINE_SECRET` | AES-256-GCM encrypted | Production ready |

For production deployments, always configure `EENGINE_SECRET`.

## How do I enable encryption?

Set the `EENGINE_SECRET` environment variable before starting EmailEngine. Generate the secret once and store it permanently - if a different or missing secret is used after a restart, the stored credentials cannot be decrypted:

```bash
# Generate the secret once and persist it in an .env file
echo "EENGINE_SECRET=$(openssl rand -hex 32)" >> .env
```

Alternatively, generate the value with `openssl rand -hex 32` and store it in a secrets manager, then provide it to EmailEngine on every start.

For existing installations with unencrypted data, run the encryption migration:

```bash
emailengine encrypt --service.secret="your-secret" --dbs.redis="redis://localhost:6379"
```

[Complete encryption guide](/docs/advanced/encryption)

## What encryption algorithm is used?

EmailEngine uses **AES-256-GCM** (Advanced Encryption Standard with 256-bit keys in Galois/Counter Mode). The key is derived from `EENGINE_SECRET` with scrypt, using a 16-byte salt generated per value; each value also carries its own 12-byte IV and a 16-byte authentication tag. Tampering with a stored value is therefore detected on decryption rather than silently accepted.

## What happens if Redis is compromised?

| Scenario | Impact |
|----------|--------|
| Without encryption | Attacker gains all passwords and OAuth tokens in cleartext |
| With encryption | Attacker sees encrypted data; credentials remain secure unless `EENGINE_SECRET` is also compromised |

Three things follow from that:

1. Enable encryption in production
2. Store `EENGINE_SECRET` separately from Redis, so one backup cannot yield both
3. Use Redis authentication and network isolation

## What happens if I lose EENGINE_SECRET?

Every encrypted value is unrecoverable. Password accounts have to be given their credentials again and OAuth2 accounts have to be re-authorized, one by one; there is no bulk recovery, because EmailEngine holds no second copy of the key.

Guard against it by backing the secret up somewhere other than the Redis backups, and by keeping it in a secrets manager rather than only in a deployment file.

## How do I rotate the encryption secret?

EmailEngine supports secret rotation:

```bash
# Re-encrypt with a new secret
emailengine encrypt \
  --service.secret="new-secret" \
  --decrypt="old-secret" \
  --dbs.redis="redis://localhost:6379"
```

The `--decrypt` argument can be repeated if data was encrypted with multiple old secrets.

The migration will:
1. Decrypt data with the old secret(s) provided via `--decrypt`
2. Re-encrypt with the new secret
3. Update all stored credentials

[Secret rotation guide](/docs/advanced/encryption#2-secret-rotation)

## Can I use external secret managers?

`EENGINE_SECRET` is read from the environment like any other variable, so any secret manager that can populate the environment before the process starts works. Four common ones:

**HashiCorp Vault:**
```bash
export EENGINE_SECRET=$(vault kv get -field=secret secret/emailengine)
```

**AWS Secrets Manager:**
```bash
export EENGINE_SECRET=$(aws secretsmanager get-secret-value \
  --secret-id emailengine/secret --query SecretString --output text)
```

**Kubernetes Secrets:**
```yaml
env:
  - name: EENGINE_SECRET
    valueFrom:
      secretKeyRef:
        name: emailengine-secrets
        key: encryption-secret
```

**Docker Secrets:**
```bash
export EENGINE_SECRET=$(cat /run/secrets/emailengine_secret)
```

[Secret management examples](/docs/advanced/encryption#using-secret-management-systems)

## How are API tokens stored?

Only as a SHA-256 hash. The token value is shown once, when it is created, and cannot be recovered from the hash afterwards. The hash is the token's `id` in listings and log entries, and `DELETE /v1/tokens/{token}` accepts either the value or the hash. See [Access tokens](/docs/api-reference/access-tokens#token-id).

## How are admin sessions protected?

The admin interface sets a session cookie named `ee`. It is sealed with a key EmailEngine generates on first start and stores encrypted in Redis, so it cannot be read or forged without that key. The cookie is marked `Secure` when `serviceUrl` uses `https://`, and `SameSite=Lax`. Changing the admin password ends every existing session at once.

## What do the logs contain?

The `Authorization` and `Cookie` request headers and an `access_token` query parameter are redacted before a request is logged, as are the offending values inside a request-validation error, which would otherwise carry the credential a rejected payload contained. Account passwords and OAuth2 tokens are not written to the log in normal operation.

The exception is raw protocol logging: `EENGINE_LOG_RAW=true` writes the IMAP conversation as-is, which includes the login exchange. Use it for a short debugging session, and treat the resulting log as sensitive. See [Logging](/docs/advanced/logging).

## What does EmailEngine send out?

An instance with a subscription license validates the key against `postalsys.com` once a day. That request carries the license key, the EmailEngine version, an instance ID, and an anonymized feature beacon; `EENGINE_BEACON_DISABLED=true` removes the beacon. The beacon itself holds enable flags, provider type names, coarse magnitude tiers rather than counts, usage booleans, and runtime context such as the Node.js version and CPU architecture. No email content, addresses, URLs, or credentials leave the server. A trial key and any other time-limited key are verified offline and make no request at all. [Licensing](/docs/licensing#what-a-licensed-instance-sends-home) and [Compliance](/docs/deployment/compliance#no-developer-access) describe the request in full.

## Is there a software bill of materials?

Yes, in SPDX format, listing every package in the running build:

- `GET /sbom.json`, which needs an access token holding the full `api` scope. An account-bound token or one narrowed with a `permissions` record is refused, because the inventory belongs to the instance rather than to any account
- Since v2.79.4, as a download from the **Legal Information** page of the admin interface (`/admin/legal/sbom.json`). It is served on the admin surface because the API token strategy has no session-cookie fallback, so a link from a rendered page answered a signed-in admin with a 401. Reaching it therefore requires an admin session, and an address inside `EENGINE_ADMIN_ACCESS_ADDRESSES` wherever that allowlist is set

See [Compliance](/docs/deployment/compliance#audit-support) for the request.

## How do I secure Redis itself?

Beyond encrypting credentials, secure your Redis instance:

### Enable Redis Authentication

```bash
# redis.conf
requirepass your-redis-password

# Connection URL
EENGINE_REDIS="redis://:your-redis-password@localhost:6379"
```

### Use TLS Encryption

```bash
# Connect via TLS
EENGINE_REDIS="rediss://localhost:6379"
```

### Network Isolation

- Bind Redis to localhost or private network only
- Use firewall rules to restrict access
- Consider Redis ACLs for fine-grained permissions

[Redis security guide](/docs/configuration/redis)

## Security Checklist for Production

Before deploying EmailEngine to production:

- [ ] `EENGINE_SECRET` is configured with a strong random value
- [ ] Secret is stored securely (not in code repository)
- [ ] Secret is backed up separately from Redis data
- [ ] Redis authentication is enabled
- [ ] Redis is not exposed to public network
- [ ] TLS is enabled for Redis connections (if over network)
- [ ] Admin interface is reachable only from known addresses (`EENGINE_ADMIN_ACCESS_ADDRESSES`, or a VPN in front)
- [ ] API tokens are scoped, and bound to an account or a permissions record where they do not need instance-wide access

## See Also

- [Encryption Guide](/docs/advanced/encryption) - Detailed encryption configuration
- [Security Best Practices](/docs/deployment/security) - Production security hardening
- [Redis Configuration](/docs/configuration/redis) - Redis setup and security
- [Environment Variables](/docs/configuration/environment-variables) - All configuration options
