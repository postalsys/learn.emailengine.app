---
title: Quick Reference
sidebar_position: 11
description: Quick reference cards for EmailEngine API endpoints, webhook events, environment variables, and error codes
---

# Quick Reference

Quick lookup tables for common EmailEngine configuration and API usage.

## Webhook Events Summary

| Event | Description | When Triggered |
|-------|-------------|----------------|
| `messageNew` | New email received | Email arrives in any folder |
| `messageDeleted` | Email deleted | A message previously seen in a folder is no longer there (deleted or moved) |
| `messageUpdated` | Email flags changed | Read/unread, flagged, labels modified |
| `messageSent` | Email accepted | A queued email is accepted by the outgoing mail server |
| `messageFailed` | Email send failed | Delivery error after retries |
| `messageBounce` | Bounce notification | Bounce report received |
| `messageDeliveryError` | Delivery issue | SMTP error during send |
| `messageComplaint` | Spam complaint | Abuse report (ARF) received |
| `messageMissing` | Message not found | Message disappeared from server |
| `accountAdded` | Account registered | New account created via API |
| `accountDeleted` | Account removed | Account deleted from EmailEngine |
| `accountInitialized` | Account ready | Account fully initialized and synced |
| `authenticationError` | Auth failed | Invalid credentials or expired token |
| `authenticationSuccess` | Auth succeeded | Successfully authenticated |
| `connectError` | Connection error | Network or protocol error |
| `mailboxNew` | Mailbox created | New folder detected |
| `mailboxDeleted` | Mailbox removed | Folder deleted |
| `mailboxReset` | Mailbox reset | Folder contents changed significantly |
| `trackOpen` | Email opened | Tracking pixel loaded |
| `trackClick` | Link clicked | Tracked link accessed |
| `listUnsubscribe` | Unsubscribe request | User unsubscribed via List-Unsubscribe |
| `listSubscribe` | Subscribe request | User re-subscribed to a list |
| `exportCompleted` | Export finished | A mailbox export job completed successfully |
| `exportFailed` | Export failed | A mailbox export job stopped with an error |

See [Webhook Events Reference](/docs/reference/webhook-events) for complete payload documentation.

## API Endpoints Summary

### Account Management

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/v1/account` | Register new account |
| `GET` | `/v1/account/{account}` | Get account details |
| `PUT` | `/v1/account/{account}` | Update account |
| `DELETE` | `/v1/account/{account}` | Delete account |
| `GET` | `/v1/accounts` | List all accounts |
| `PUT` | `/v1/account/{account}/reconnect` | Force reconnect |

`PUT /v1/account/{account}/reconnect` answers `{"reconnect": false}` for an account whose syncing was switched off after repeated authentication failures (`authFailureDisabledAt` is set). Supplying working credentials lifts that; a reconnect request does not.

### Message Operations

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/v1/account/{account}/messages` | List messages |
| `GET` | `/v1/account/{account}/message/{message}` | Get message details |
| `GET` | `/v1/account/{account}/message/{message}/source` | Get raw email |
| `DELETE` | `/v1/account/{account}/message/{message}` | Delete message |
| `PUT` | `/v1/account/{account}/message/{message}` | Update flags/move |
| `POST` | `/v1/account/{account}/search` | Search messages |

### Sending Emails

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/v1/account/{account}/submit` | Send email |
| `GET` | `/v1/outbox` | List queued emails |
| `GET` | `/v1/outbox/{queueId}` | Get queued email |
| `DELETE` | `/v1/outbox/{queueId}` | Cancel queued email |

### Mailbox Operations

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/v1/account/{account}/mailboxes` | List mailboxes |
| `POST` | `/v1/account/{account}/mailbox` | Create mailbox |
| `PUT` | `/v1/account/{account}/mailbox` | Rename mailbox |
| `DELETE` | `/v1/account/{account}/mailbox` | Delete mailbox |

### Attachments

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/v1/account/{account}/attachment/{attachment}` | Download attachment |

### MCP (AI Agents)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/mcp` | [MCP endpoint](/docs/mcp): JSON-RPC tool calls. Off by default (`mcpEnabled` setting) |
| `POST` | `/mcp/oauth/register` | Dynamic client registration, when OAuth sign-in is enabled |
| `POST` | `/mcp/oauth/token` | OAuth token endpoint |
| `GET` | `/.well-known/oauth-protected-resource/mcp` | Protected resource metadata |
| `GET` | `/.well-known/oauth-authorization-server` | Authorization server metadata |

## Environment Variables

### Strongly Recommended

| Variable | Description | Example |
|----------|-------------|---------|
| `EENGINE_REDIS` | Redis connection URL (defaults to `redis://127.0.0.1:6379/8` if not set) | `redis://localhost:6379/8` |
| `EENGINE_SECRET` | Encryption secret (32+ chars) - optional, but stored credentials are not encrypted without it | `openssl rand -hex 32` |

### Server Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `EENGINE_PORT` | `3000` | HTTP API port |
| `EENGINE_HOST` | `127.0.0.1` | Bind address |
| `EENGINE_WORKERS` | `4` | Account worker threads (IMAP, Gmail API, Outlook/Graph) |
| `EENGINE_LOG_LEVEL` | `trace` | Log level (trace/debug/info/warn/error) |

### Behavior Toggles

| Variable | Default | Description |
|----------|---------|-------------|
| `EENGINE_DISABLE_SETUP_WARNINGS` | `false` | Disable admin password warnings |
| `EENGINE_REQUIRE_API_AUTH` | `true` | Require API authentication |
| `EENGINE_LOG_RAW` | `false` | Log raw IMAP/SMTP traffic (includes unmasked credentials - debug only) |
| `EENGINE_MCP_ENABLED` | `true` | Register the [MCP endpoint](/docs/mcp) routes (the `mcpEnabled` setting is the on switch) |

### Pre-configured Settings

| Variable | Description |
|----------|-------------|
| `EENGINE_SETTINGS` | JSON string of runtime settings (webhooks, serviceUrl, etc.) |
| `EENGINE_PREPARED_LICENSE` | License key |
| `EENGINE_PREPARED_TOKEN` | Pre-configured API token (exported hash) |
| `EENGINE_PREPARED_PASSWORD` | Pre-configured admin password (hash) |

:::info OAuth2 and Webhooks
OAuth2 applications and webhooks are configured via the [Settings API](/docs/api/post-v-1-settings) or web interface, not environment variables. Use `EENGINE_SETTINGS` to pre-configure them at startup.
:::

See [Environment Variables](/docs/configuration/environment-variables) for complete list.

## Common Error Codes

### HTTP Status Codes

| Code | Meaning | Common Cause |
|------|---------|--------------|
| `200` | Success | Request completed |
| `400` | Bad Request | Invalid parameters (`fields` names them), or an OAuth2 user already bound to another account (`AccountAlreadyExists`) |
| `401` | Unauthorized | Missing/invalid token |
| `403` | Forbidden | Insufficient scope |
| `404` | Not Found | Account, message or folder does not exist, or the account has no SMTP configuration (`SMTPUnavailable`) |
| `422` | Unprocessable Entity | The request is valid but the account cannot satisfy it, such as a label search on a non-Gmail mailbox |
| `429` | Too Many Requests | Rate limited. The body carries `ttl` seconds |
| `500` | Server Error | Internal error |
| `503` | Service Unavailable | The account is not connected: still initializing, failing authentication, unreachable, or not syncing (the body carries `state` and a `code`) |
| `504` | Gateway Timeout | A worker thread did not answer within `EENGINE_TIMEOUT` (`Timeout`) |

### Account Connection States

| State | Description | Action |
|-------|-------------|--------|
| `init` | Registered, not connected yet | Wait |
| `unset` | Not syncing: no IMAP or OAuth2 configuration is set, or syncing was switched off, by the operator (`imap.disabled`) or automatically after repeated authentication failures (`authFailureDisabledAt` is set) | Add connection settings, or supply working credentials to lift an automatic switch-off |
| `connecting` | Establishing connection | Wait for completion |
| `syncing` | Sync in progress | Wait for completion |
| `connected` | Connected and watching for changes | Normal operation |
| `disconnected` | Connection dropped | Retried automatically with backoff |
| `connectError` | Network or TLS failure | Check server availability |
| `authenticationError` | Credentials rejected | Update credentials or re-authenticate |
| `paused` | Syncing paused through the API; no connection is maintained | Resume when needed |

### Webhook Delivery Status

Webhook deliveries are BullMQ jobs and use BullMQ state names:

| Status | Description |
|--------|-------------|
| `waiting` | Waiting to send |
| `delayed` | Scheduled for a later attempt (e.g. retry backoff) |
| `active` | Currently sending |
| `completed` | Successfully delivered |
| `failed` | Delivery failed (will retry) |

## IMAP/SMTP Server Settings

### Gmail

| Setting | Value |
|---------|-------|
| IMAP Server | `imap.gmail.com` |
| IMAP Port | `993` (SSL) |
| SMTP Server | `smtp.gmail.com` |
| SMTP Port | `587` (STARTTLS) or `465` (SSL) |

### Outlook / Microsoft 365

| Setting | Value |
|---------|-------|
| IMAP Server | `outlook.office365.com` |
| IMAP Port | `993` (SSL) |
| SMTP Server | `smtp.office365.com` |
| SMTP Port | `587` (STARTTLS) |

### Yahoo Mail

| Setting | Value |
|---------|-------|
| IMAP Server | `imap.mail.yahoo.com` |
| IMAP Port | `993` (SSL) |
| SMTP Server | `smtp.mail.yahoo.com` |
| SMTP Port | `587` (STARTTLS) |

## OAuth2 Scopes

### Gmail

| Scope | Access Level |
|-------|--------------|
| `https://mail.google.com/` | Full access (IMAP/SMTP) |
| `https://www.googleapis.com/auth/gmail.readonly` | Read-only (API only) |
| `https://www.googleapis.com/auth/gmail.modify` | Read/write (API only) |
| `https://www.googleapis.com/auth/gmail.send` | Send only (API only) |

### Microsoft / Outlook

| Scope | Access Level |
|-------|--------------|
| `https://outlook.office.com/IMAP.AccessAsUser.All` | Full IMAP access |
| `https://outlook.office.com/SMTP.Send` | SMTP send access |
| `offline_access` | Refresh token support |

## Special Folder Paths

| Logical Path | Gmail | Outlook | Standard IMAP |
|--------------|-------|---------|---------------|
| `\Inbox` | `INBOX` | `Inbox` | `INBOX` |
| `\Sent` | `[Gmail]/Sent Mail` | `Sent Items` | `Sent` |
| `\Drafts` | `[Gmail]/Drafts` | `Drafts` | `Drafts` |
| `\Trash` | `[Gmail]/Trash` | `Deleted Items` | `Trash` |
| `\Junk` | `[Gmail]/Spam` | `Junk Email` | `Junk` |
| `\Archive` | `[Gmail]/All Mail` | `Archive` | `Archive` |

## Docker Quick Commands

```bash
# Start with Docker Compose
docker-compose up -d

# View logs
docker-compose logs -f emailengine

# Stop services
docker-compose down

# Update to latest
docker-compose pull && docker-compose up -d

# Check health
curl https://emailengine.example.com/health
```

## API Authentication

```bash
# Using Bearer token
curl -H "Authorization: Bearer YOUR_TOKEN" \
  https://emailengine.example.com/v1/accounts

# Using query parameter
curl "https://emailengine.example.com/v1/accounts?access_token=YOUR_TOKEN"
```

## Common API Examples

### Register Account (OAuth2)

```bash
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user123",
    "email": "user@gmail.com",
    "oauth2": {
      "provider": "OAUTH_APP_ID",
      "refreshToken": "REFRESH_TOKEN",
      "auth": {"user": "user@gmail.com"}
    }
  }'
```

### Send Email

```bash
curl -X POST https://emailengine.example.com/v1/account/user123/submit \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "to": [{"address": "recipient@example.com"}],
    "subject": "Hello",
    "text": "Hello World"
  }'
```

### Search Messages

Search is a `POST` request - the search criteria go in the JSON body, and the mailbox path is passed as the `path` query parameter:

```bash
curl -X POST "https://emailengine.example.com/v1/account/user123/search?path=INBOX" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "search": {
      "subject": "invoice"
    }
  }'
```

### Configure Webhooks

```bash
curl -X POST https://emailengine.example.com/v1/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "webhooks": "https://your-app.com/webhooks",
    "webhooksEnabled": true,
    "webhookEvents": ["messageNew", "messageSent"]
  }'
```

Delivery only starts once `webhooksEnabled` is `true`, and `webhookEvents` is an allowlist. Use `["*"]` to receive every event.

## See Also

- [API Reference Overview](/docs/api-reference) - Authentication, conventions, and error handling
- [Environment Variables](/docs/configuration/environment-variables) - The complete configuration reference
- [Webhook Events Reference](/docs/reference/webhook-events) - Full payloads for every event
- [Error Codes](/docs/reference/error-codes) - Every error code and how to respond to it
- [Glossary](/docs/reference/glossary) - Terms used throughout the documentation
