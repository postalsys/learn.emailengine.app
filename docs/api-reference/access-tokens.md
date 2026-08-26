---
title: Access Tokens
description: Complete guide to managing API access tokens in EmailEngine
sidebar_position: 2
---

# Access Tokens

Access tokens are required to authenticate all API requests to EmailEngine. This guide covers token types, creation methods, security best practices, and management strategies.

## Overview

### What are Access Tokens

Access tokens are 64-character hexadecimal strings that authenticate API requests. EmailEngine supports two types of tokens:

1. **System-wide tokens**: Full access to all EmailEngine endpoints and all accounts
2. **Account-specific tokens**: Restricted to operations on a single account

Either kind can additionally carry a [`permissions` record](#permissions) that narrows what it may do. A system-wide token minted over the API must carry one.

### Token Format

All tokens are 64-character hexadecimal strings (32 bytes):

```
f05d76644ea39c4a2ee33e7bffe55808b716a34b51d67b388c7d60498b0f89bc
```

### Token ID

EmailEngine stores only the SHA-256 hash of each token, never the token value itself. This hash is the token's stable identifier:

- Returned as the `id` field by `GET /v1/tokens` (add `?account={account}` to list one account's tokens)
- Shown in the admin UI access tokens list as the "Token ID" column (first 8 hex characters, full hash on hover)
- Included in log entries, so API requests can be correlated to the token that made them

The token value cannot be recovered from the `id`. `DELETE /v1/tokens/{token}` accepts either the original token value or the token `id`.

## Token Types

### System-Wide Tokens

**Created via:**

- Web interface (Integrations > Access Tokens)
- CLI: `emailengine tokens issue`
- API: `POST /v1/tokens` without an `account`, which then requires a `permissions` record

**Characteristics:**

- Access all accounts
- Access all API endpoints, unless narrowed by `permissions`
- Can create other tokens, unless narrowed by `permissions`
- Can optionally be scoped to specific account using `-a` flag in CLI
- Recommended for administrative tasks

**Example use cases:**

- Account management (create, update, delete accounts)
- System configuration
- Multi-account operations
- Administrative automation

### Account-Specific Tokens

**Created via:**

- API: `POST /v1/tokens` (with an `account` field)

**Characteristics:**

- Bound to single account
- Can only access that account's data
- Cannot create other tokens
- Cannot access system-wide endpoints: any route that has no `{account}` in its path is refused with `403 Unauthorized account`
- Recommended for user-facing applications

**Example use cases:**

- Per-user API access in multi-tenant applications
- Limited scope for third-party integrations
- Security-sensitive deployments

**Important:** A token minted over the API must be narrowed: either bind it to an `account`, or send a `permissions` record. The API declines to mint an instance-wide token that can reach every account and every endpoint - create those in the web interface or with the CLI.

## Creating Tokens

### Method 1: Web Interface

**Best for:** Manual token creation, administrative tokens

1. Log in to EmailEngine web interface
2. Navigate to **Integrations** > **Access Tokens**
3. Click **Create access token**
4. Enter a description, optionally bind the token to an account, select scopes, and narrow it with **Restrict what this token can do** if needed
5. Click **Generate a token**
6. Copy the token (shown only once)

![Create token form](/img/screenshots/token-new-form.png)
_The token form takes a description, an optional account binding and the allowed scopes, and can narrow what the token may do. The generated token value is shown only once_

**Pros:**

- Simple and intuitive
- Visual scope selection
- Immediate feedback

**Cons:**

- Requires manual interaction
- Not suitable for automation

### Method 2: CLI

**Best for:** Automation, CI/CD, Docker deployments, infrastructure-as-code

The EmailEngine CLI provides commands to generate, export, import, and manage tokens programmatically. This is particularly useful for automated deployments and prepared token configuration.

:::tip CLI Documentation
For complete CLI usage, installation, and configuration options, see the [Command Line Interface (CLI)](/docs/configuration/cli) documentation.
:::

**Generate system-wide token:**

```bash
emailengine tokens issue \
  -d "My admin token" \
  -s "*" \
  --dbs.redis="redis://127.0.0.1:6379/8"
```

**Generate account-specific token:**

```bash
emailengine tokens issue \
  -d "User token" \
  -s "api" \
  -a "user123" \
  --dbs.redis="redis://127.0.0.1:6379/8"
```

**Output:**

```
f05d76644ea39c4a2ee33e7bffe55808b716a34b51d67b388c7d60498b0f89bc
```

Without `-s` the CLI issues a `*` token, and without `-d` the description is `Generated at <timestamp>`.

For detailed CLI usage, export/import workflows, and prepared token configuration for automated deployments, see [Prepared Tokens](/docs/configuration/prepared-settings/tokens).

### Method 3: API

**Best for:** Programmatic token creation, multi-tenant applications

**Endpoint:** `POST /v1/tokens`

[Detailed API reference →](/docs/api/post-v-1-tokens)

**Authentication:** Requires an instance-wide token with the `*` or `api` scope and no `permissions` record. Token provisioning is in the never-grantable `admin` group, so an account-bound token is refused (the route has no `{account}` parameter) and a narrowed token is refused whatever it lists. A request made while API authentication is switched off is refused too, since nothing could ever revoke a token handed to an anonymous caller.

```bash
curl -X POST https://emailengine.example.com/v1/tokens \
  -H "Authorization: Bearer EXISTING_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user123",
    "description": "User API token",
    "scopes": ["api"]
  }'
```

**Fields:**

- `description` (string, required): Token description
- `scopes` (array): Token scopes, default `["api"]`
- `account` (string): Account ID this token is bound to
- `permissions` (object): `actions` and `groups` allowlists, see [Permissions](#permissions)
- `restrictions` (object): IP, referrer and rate limits, see [Token Restrictions](#token-restrictions)
- `metadata` (string): Arbitrary JSON, stored with the token and returned by `GET /v1/tokens/{token}`
- `expires` (date-time): When the token stops working. Omit for a token that never expires. An expired token is refused and its record is removed the next time it is presented or listed

Either `account` or `permissions` has to be present.

**Response:**

```json
{
  "token": "f05d76644ea39c4a2ee33e7bffe55808b716a34b51d67b388c7d60498b0f89bc",
  "id": "1bc12baf7f0d5e51fe0a4e0eda06e1be5b8d6cc2c66b95dc0fbe4d2e9f5d5e1a"
}
```

**Important notes:**

- The token value is returned once and stored hashed. The `id` is what the listings report afterwards
- An unbound token must carry a `permissions` record; without one the request is refused
- You need an existing full-access token to mint tokens

:::note Deprecated paths
EmailEngine 2.79.0 renamed the token endpoints. `POST /v1/token`, `DELETE /v1/token/{token}` and `GET /v1/tokens/account/{account}` still answer, with the same handlers as `POST /v1/tokens`, `DELETE /v1/tokens/{token}` and `GET /v1/tokens?account=user123`, so existing integrations keep working, but they are left out of the OpenAPI document and the API reference. Use the new paths.
:::

## Token Scopes

Scopes define what a token can access:

| Scope        | Description     | Access                            | Available via           |
| ------------ | --------------- | --------------------------------- | ----------------------- |
| `*`          | Full access     | All API endpoints, all operations | Web UI, CLI only        |
| `api`        | API access only | Standard API calls, no metrics    | Web UI, CLI, API        |
| `metrics`    | Metrics only    | Prometheus metrics endpoint only  | Web UI, CLI only        |
| `smtp`       | SMTP proxy      | SMTP gateway access               | Web UI, CLI, API        |
| `imap-proxy` | IMAP proxy      | IMAP proxy access                 | Web UI, CLI, API        |
| `mcp`        | MCP endpoint    | AI agent tool calls at `/mcp`, and nothing on the REST API | Web UI, CLI, API |

:::info API Scope Limitations
When creating tokens via the `POST /v1/tokens` API endpoint, only `api`, `smtp`, `imap-proxy`, and `mcp` scopes are available. The `*` (full access) and `metrics` scopes can only be assigned through the Web UI or CLI.
:::

:::note The `mcp` scope is surface-bound
A token carrying `mcp` opens the [MCP endpoint](/docs/mcp) and is refused by `/v1` with an "Unauthorized scope" error. Inside MCP it admits only the operations the tool set wraps: reading accounts, folders, messages, the sending queue and templates, modifying and deleting messages, and sending mail. See [MCP Access Control](/docs/mcp/access-control).
:::

**Multiple scopes:**

```json
{
  "scopes": ["api", "smtp"]
}
```

**Default scope:** `["api"]` over the API, `["*"]` from the CLI.

The `smtp`, `imap-proxy` and `metrics` scopes are checked once, at login, and then hand over a session. A token that also carries a `permissions` record is admitted to them only if the record covers everything that session could do: `send` on `submit` for SMTP, `read` on `diagnostics` for metrics, and for the IMAP proxy `read`, `write` and `destructive` on `message` plus `write` and `destructive` on `mailbox`, because a proxied IMAP session can delete and expunge. The token form warns when a chosen scope would be unusable with the record.

## Permissions

A `permissions` record narrows a token below its scope. It has two axes, each an allowlist, and both apply together: an operation is allowed only when its action is in `actions` and its group is in `groups`. Omit an axis to leave it unrestricted. An empty array is refused, because a record that lists nothing allows nothing.

```json
{
  "description": "Read-only mail access",
  "scopes": ["api"],
  "permissions": {
    "actions": ["read"],
    "groups": ["account", "mailbox", "message"]
  }
}
```

**Actions**, one per operation:

| Action | Allows |
|--------|--------|
| `read` | Read data without changing anything |
| `write` | Create or modify data |
| `send` | Hand a message to a mail server, so it reaches real recipients |
| `destructive` | Call the endpoints that remove data. Deleting a message is a move to Trash, which a `write` grant can perform directly, so withholding this narrows the endpoints rather than the outcome for mail |

**Groups**, one per operation:

| Group | Covers |
|-------|--------|
| `account` | Reading accounts and operating on their connection state: list, get, delete, reconnect, sync, flush, server signatures |
| `mailbox` | Create, rename, delete and list mailbox folders |
| `message` | Read, modify, move and delete messages, including bulk actions and search |
| `submit` | Send email, including scheduled sends and the delivery test. A send may reference a stored message to forward it, so this also reads the message it names |
| `outbox` | Inspect and cancel queued outbound messages |
| `export` | Bulk export an account. One call archives every folder, so this is separate from message access |
| `template` | Manage stored email templates |
| `blocklist` | Manage suppression lists |
| `webhook` | Read webhook route definitions |
| `gateway` | Read and delete SMTP gateways. Creating or editing one is an admin operation, because it can redirect where stored relay credentials are sent |
| `events` | Subscribe to the instance-wide change stream, which covers every account |
| `diagnostics` | Read statistics and service status, including `/metrics` |
| `logs` | Read the per-account connection log. Entries are the raw protocol trace, so they include folder names and message subjects |

Every operation in the [API reference](/docs/api/emailengine-api) publishes the action and group it requires as `x-ee-action` and `x-ee-group`, so the reference and the enforcement cannot disagree.

Some operations belong to an `admin` group that no record may name. A token with any `permissions` record, whatever it lists, can never read or change settings, manage OAuth2 applications or the license, read a stored credential or an account's live OAuth2 access token, create or edit an account or a gateway, mint a hosted authentication form, verify credentials, or create, list, inspect or revoke tokens. That is what keeps a narrowed token from widening itself.

A request refused by the record answers `403 Forbidden` with the message `Unauthorized permission`. The [audit log](#audit-log) records which axis refused it.

The admin token form offers presets (read only, mail agent, send only, everything allowed) that fill in these records, and the [MCP access levels](/docs/mcp/access-control#access-levels) are the same records with the groups the MCP surface exposes.

## Token Management

### Export and Import Tokens

You can export tokens for backup or to transfer them between EmailEngine instances. Exported tokens can also be used as prepared tokens for automated deployments.

**Important:** Exported tokens are data structures containing the token hash, NOT the actual token value. The exported data CANNOT be used directly as an API token. Only the original token value generated during creation can be used for API authentication.

```bash
# Export a token (exports token metadata and hash)
emailengine tokens export -t TOKEN_VALUE

# Import a previously exported token
emailengine tokens import -t EXPORTED_DATA
```

For complete export/import workflows and prepared token configuration, see [Prepared Tokens](/docs/configuration/prepared-settings/tokens).

### Revoking Tokens

**Via web interface:**

1. Navigate to **Integrations** > **Access Tokens**
2. Find the token to revoke
3. Click **Delete**
4. Confirm deletion

![Access token list](/img/screenshots/tokens-list.png)
_The Access Tokens page lists active tokens with their descriptions and scopes_

**Via API:**

```bash
curl -X DELETE https://emailengine.example.com/v1/tokens/TOKEN_VALUE_OR_ID \
  -H "Authorization: Bearer ADMIN_TOKEN"
```

The path parameter accepts either the original token value or the token `id` (the SHA-256 hash shown by the token listing endpoints), so tokens can be revoked even if the original value is no longer available.

[Detailed API reference →](/docs/api/delete-v-1-tokens-token)

:::note CLI Token Deletion
The CLI does not have a `tokens delete` command. To delete tokens programmatically, use the API endpoint above or the web interface.
:::

### Audit Log

EmailEngine can record what each token actually did. It is off by default: turn it on under **Configuration** > **Security** > **Access Token Audit Log**, after which every request a token makes is recorded, refusals included.

```bash
curl "https://emailengine.example.com/v1/tokens/TOKEN_VALUE_OR_ID/log" \
  -H "Authorization: Bearer ADMIN_TOKEN"
```

Each entry names the moment, the client address, the request, and what happened to it:

| Field | Meaning |
|-------|---------|
| `time` | When the request arrived |
| `ip` | The client address, resolved through the proxy configuration if there is one |
| `method`, `path` | The API operation, as the route pattern: `get` and `/v1/account/{account}/messages`. For SMTP and the IMAP proxy, `method` is the surface name instead |
| `action`, `group` | The permission the operation resolved to |
| `account` | The account the request named, when it named one |
| `status` | `allowed` or `denied` |
| `reason` | Why a denied request was refused: `scope`, `account`, `address`, `referrer` or `rateLimit` for a token-level refusal, `action`, `group` or `restricted` when the `permissions` record refused it, `malformed` or `unclassified` when the record or the route could not be read, and `username` or `ip` when the SMTP server or the IMAP proxy refused the login. `null` when allowed |

Retention is bounded per token by [`EENGINE_TOKEN_LOG_ENTRIES`](/docs/configuration/environment-variables#security--access-control), 1000 by default, and [`EENGINE_TOKEN_LOG_AGE`](/docs/configuration/environment-variables#security--access-control), seven days by default, whichever is reached first. The log lives in Redis, so both limits cost memory across every token that has one.

A denied entry is the useful half: it shows an integration reaching for something its token was never granted, which is what a narrowed token is supposed to surface.

## Disabling Authentication (Development Only)

:::danger Development only
Disabling authentication removes all API access control. Use it on a local instance with test accounts, and re-enable it before the instance is reachable by anyone else.
:::

You can disable the access token requirement for development purposes:

1. Log in to EmailEngine web interface
2. Navigate to **Configuration** > **Security**
3. Uncheck **Require API Authentication**
4. Click **Save Changes**

**When disabled:**
- API calls work without `Authorization` header
- No token validation is performed
- All endpoints are accessible without authentication, except `POST /v1/tokens`, which refuses to mint a token for an unauthenticated caller
- No access control or user tracking

**Example:**
```bash
# With authentication disabled
curl https://emailengine.example.com/v1/accounts
# Works without Bearer token
```

**Use cases:**
- Local development without tokens
- Quick testing and debugging
- Development environment setup
- Learning the API

**Before going to production:**
1. Re-enable "Require API Authentication"
2. Create proper access tokens
3. Remove any unauthenticated API calls from your code
4. Test with authentication enabled

## Token Restrictions

Access tokens can be configured with security restrictions to limit their usage by IP address, HTTP referrer, and rate limits. These restrictions provide additional security layers for tokens used in different environments.

### Configuration Options

Token restrictions are configured when creating a token via the API:

```bash
curl -X POST https://emailengine.example.com/v1/tokens \
  -H "Authorization: Bearer EXISTING_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user123",
    "description": "Restricted API token",
    "scopes": ["api"],
    "restrictions": {
      "addresses": ["192.168.1.0/24", "10.0.0.5"],
      "referrers": ["https://myapp.com/*", "*.example.org/*"],
      "rateLimit": {
        "maxRequests": 100,
        "timeWindow": 60
      }
    }
  }'
```

### IP Address Allowlist

Restrict token usage to specific IP addresses or CIDR ranges:

```json
{
  "restrictions": {
    "addresses": ["1.2.3.4", "5.6.7.8", "192.168.0.0/16", "10.0.0.0/8"]
  }
}
```

**Supported formats:**

- Single IPv4 addresses: `"192.168.1.100"`
- Single IPv6 addresses: `"2001:db8::1"`
- CIDR ranges: `"192.168.0.0/24"`, `"10.0.0.0/8"`

Requests from IP addresses not in the allowlist are rejected with `403 Forbidden` and the message `Unauthorized address`. The address is the one the [proxy configuration](/docs/deployment/nginx-proxy) resolves, so behind a reverse proxy the trusted `X-Forwarded-For` value is what gets matched.

### HTTP Referrer Patterns

Restrict token usage based on the HTTP `Referer` header. This is useful for tokens used in browser-based applications:

```json
{
  "restrictions": {
    "referrers": ["*web.domain.org/*", "*.domain.org/*", "https://domain.org/*"]
  }
}
```

**Pattern syntax:**

- `*` matches any sequence of characters
- Patterns are matched against the full referrer URL
- Multiple patterns can be specified (any match allows the request)

**Use cases:**

- Restrict tokens to specific web applications
- Prevent token misuse if leaked
- Enforce origin-based access control

A request whose `Referer` header matches none of the patterns is rejected with `403 Forbidden` and the message `Unauthorized referrer`.

:::warning Referrer Limitations
HTTP referrer restrictions can be bypassed by clients that do not send the `Referer` header or forge it. Use this as an additional layer of security, not as the sole protection mechanism.
:::

### Rate Limiting

Limit the number of API requests a token can make within a time window:

```json
{
  "restrictions": {
    "rateLimit": {
      "maxRequests": 100,
      "timeWindow": 60
    }
  }
}
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `maxRequests` | integer | Maximum number of requests allowed in the time window |
| `timeWindow` | integer | Time window duration in seconds |

**Example configurations:**

```javascript
// 20 requests per 2 seconds (burst protection)
{ "maxRequests": 20, "timeWindow": 2 }

// 1000 requests per hour (daily limit)
{ "maxRequests": 1000, "timeWindow": 3600 }

// 100 requests per minute (standard rate limit)
{ "maxRequests": 100, "timeWindow": 60 }
```

When the rate limit is exceeded, requests are rejected with `429 Too Many Requests`. The body carries `ttl`, the number of seconds until the window resets, and the response sets `X-RateLimit-Limit` and `X-RateLimit-Reset`. Requests that are within the limit carry the same two headers plus `X-RateLimit-Remaining`.

```json
{
  "statusCode": 429,
  "error": "Too Many Requests",
  "message": "Rate limit exceeded",
  "ttl": 42
}
```

### Combining Restrictions

All restriction types can be combined. A request must satisfy ALL configured restrictions:

```bash
curl -X POST https://emailengine.example.com/v1/tokens \
  -H "Authorization: Bearer EXISTING_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user123",
    "description": "Fully restricted frontend token",
    "scopes": ["api"],
    "restrictions": {
      "addresses": ["203.0.113.0/24"],
      "referrers": ["https://app.example.com/*"],
      "rateLimit": {
        "maxRequests": 50,
        "timeWindow": 60
      }
    }
  }'
```

### Disabling Restrictions

Set any restriction to `false` to disable it:

```json
{
  "restrictions": {
    "addresses": false,
    "referrers": false,
    "rateLimit": false
  }
}
```

Or omit the `restrictions` object entirely to create an unrestricted token.

## See Also

- [Command Line Interface (CLI)](/docs/configuration/cli) - Complete CLI reference for token management and administration
- [Prepared Tokens](/docs/configuration/prepared-settings/tokens) - CLI commands, export/import, and automated deployment configuration
- [API Authentication](/docs/api-reference/#authentication) - Using tokens in API requests
- [Account Management API](/docs/api-reference/accounts-api) - Managing email accounts with tokens
- [Security Best Practices](/docs/deployment/security) - General security guidelines for EmailEngine deployment

