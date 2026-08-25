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

**Characteristics:**

- Access all accounts
- Access all API endpoints
- Can create other tokens
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
- Cannot access system-wide endpoints
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
3. Click **Create new**
4. Enter description and select scopes
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

For detailed CLI usage, export/import workflows, and prepared token configuration for automated deployments, see [Prepared Tokens](/docs/configuration/prepared-settings/tokens).

### Method 3: API

**Best for:** Programmatic token creation, multi-tenant applications

**Endpoint:** `POST /v1/tokens`

[Detailed API reference →](/docs/api/post-v-1-tokens)

**Authentication:** Requires existing valid token

```bash
curl -X POST http://localhost:3000/v1/tokens \
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
- `scopes` (array, required): Token scopes
- `account` (string): Account ID this token is bound to
- `permissions` (object): Narrowing for an instance-wide token, as `actions` and `groups` allowlists
- `restrictions` (object): IP, referrer and rate limits, see [Token Restrictions](#token-restrictions)

Either `account` or `permissions` has to be present.

**Response:**

```json
{
  "token": "a1b2c3d4e5f6...64-char-hex-string",
  "id": "4c2805ce5ea8...64-char-hex-hash"
}
```

**Important notes:**

- The token value is returned once and stored hashed. The `id` is what the listings report afterwards
- An unbound token must carry a `permissions` record; without one the request is refused
- You need an existing full-access token to mint tokens

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

**Default scope:** `["api"]`

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
curl -X DELETE http://localhost:3000/v1/tokens/TOKEN_VALUE_OR_ID \
  -H "Authorization: Bearer ADMIN_TOKEN"
```

The path parameter accepts either the original token value or the token `id` (the SHA-256 hash shown by the token listing endpoints), so tokens can be revoked even if the original value is no longer available.

[Detailed API reference →](/docs/api/delete-v-1-tokens-token)

:::note CLI Token Deletion
The CLI does not have a `tokens delete` command. To delete tokens programmatically, use the API endpoint above or the web interface.
:::

## Disabling Authentication (Development Only)

:::danger Never Use in Production
Disabling authentication removes all API security. Only use for local development or testing.
:::

You can disable the access token requirement for development purposes:

1. Log in to EmailEngine web interface
2. Navigate to **Configuration** → **Security**
3. Uncheck **"Require API Authentication"**
4. Save settings

**When disabled:**
- API calls work without `Authorization` header
- No token validation is performed
- All endpoints are accessible without authentication
- No access control or user tracking

**Example:**
```bash
# With authentication disabled
curl http://localhost:3000/v1/accounts
# Works without Bearer token
```

**Use cases:**
- Local development without tokens
- Quick testing and debugging
- Development environment setup
- Learning the API

**Security warnings:**

:::warning Production Deployment
- **NEVER** disable authentication in production
- **NEVER** disable authentication on publicly accessible instances
- **NEVER** disable authentication on instances with real email accounts
- **ALWAYS** re-enable before deploying to production
:::

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
curl -X POST http://localhost:3000/v1/token \
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

Requests from IP addresses not in the allowlist will be rejected with a 403 Forbidden response.

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

When the rate limit is exceeded, requests are rejected with a 429 Too Many Requests response.

### Combining Restrictions

All restriction types can be combined. A request must satisfy ALL configured restrictions:

```bash
curl -X POST http://localhost:3000/v1/token \
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

