---
title: Glossary
sidebar_position: 10
description: Technical terms and definitions used in EmailEngine documentation
---

# Glossary

Technical terms and definitions used throughout the EmailEngine documentation.

## Email Protocol Terms

### IMAP (Internet Message Access Protocol)

A protocol for accessing email messages stored on a mail server. IMAP allows multiple clients to access the same mailbox and keeps messages on the server. EmailEngine maintains persistent IMAP connections to sync email data.

### SMTP (Simple Mail Transfer Protocol)

The standard protocol for sending emails across the internet. EmailEngine uses SMTP connections to submit outgoing messages through email providers.

### IDLE

An IMAP extension that allows the server to notify the client immediately when new messages arrive or existing messages change, without the client having to poll repeatedly. EmailEngine uses IDLE for real-time email notifications.

### UID (Unique Identifier)

A unique number assigned to each message in an IMAP mailbox. UIDs persist across sessions and are used to identify specific messages. EmailEngine uses UIDs internally but exposes its own ID system to users.

### UIDVALIDITY

A value that indicates whether the UIDs in a mailbox are still valid. If UIDVALIDITY changes (e.g., after mailbox reconstruction), all UIDs become invalid and messages must be re-synced. EmailEngine handles UIDVALIDITY changes automatically.

### MODSEQ (Modification Sequence)

A counter that increases each time any message in a mailbox is modified. Used for efficient synchronization - EmailEngine only fetches changes since the last known MODSEQ value.

### Envelope

The metadata of an email message including From, To, Subject, Date, and Message-ID headers. Distinguished from the message body and attachments.

### MIME (Multipurpose Internet Mail Extensions)

A standard for formatting non-ASCII content in email messages, including attachments, HTML content, and international characters.

### Special-Use Folders

Mailboxes with specific purposes defined by the IMAP server, such as Sent, Drafts, Trash, Junk, and Archive. EmailEngine detects these automatically using IMAP SPECIAL-USE extension.

## OAuth2 Terms

### OAuth2 (Open Authorization 2.0)

An authorization framework that allows applications to access user accounts without handling passwords directly. EmailEngine uses OAuth2 for Gmail, Outlook, and other providers.

### Access Token

A short-lived credential (typically 1 hour) that grants access to a user's account. EmailEngine automatically refreshes access tokens before they expire.

### Refresh Token

A long-lived credential used to obtain new access tokens without requiring the user to re-authenticate. EmailEngine stores refresh tokens securely and uses them to maintain persistent access.

### OAuth2 Scope

Permissions that define what actions an application can perform. Examples:
- `https://mail.google.com/` - Full Gmail access (IMAP/SMTP)
- `https://www.googleapis.com/auth/gmail.modify` - Gmail API read/write access
- `https://outlook.office.com/IMAP.AccessAsUser.All` - Outlook IMAP access

### Consent Screen

The authorization page shown to users where they grant permission for an application to access their account. Configured in Google Cloud Console or Azure Portal.

### Client ID / Client Secret

Credentials that identify your application to OAuth2 providers. The Client ID is public; the Client Secret must be kept confidential.

### Service Account

A special type of Google account for applications (not users) that can access resources without user interaction. Requires Google Workspace and domain-wide delegation.

### Domain-Wide Delegation

A feature that allows a service account to impersonate any user in a Google Workspace domain. Configured by organization admins.

### Two-Legged OAuth2

OAuth2 flow where the application authenticates directly without user interaction, typically using service accounts.

### Three-Legged OAuth2

Standard OAuth2 flow involving user consent, where the user authorizes the application through a consent screen.

## EmailEngine Terms

### Account

An email account registered with EmailEngine. Each account represents a connection to one email address and can be accessed via the EmailEngine API.

### Account ID

A unique identifier for an account in EmailEngine. Can be auto-generated or specified during account creation. Used in API endpoints like `/v1/account/{accountId}/...`.

### Account State

The `state` value an account reports: `init`, `unset`, `connecting`, `syncing`, `connected`, `disconnected`, `connectError`, `authenticationError` or `paused`. `connected` is the healthy steady state. `unset` means the account is not syncing, either because no IMAP or OAuth2 configuration is set or because syncing was switched off. See [Account states](/docs/reference/quick-reference#account-connection-states).

### Authentication Failure Safety Net

The mechanism that stops an account from retrying forever once its credentials have been rejected continuously for longer than `EENGINE_MAX_IMAP_AUTH_FAILURE_TIME` (3 days by default). It sets `imap.disabled`, records the time in the read-only `authFailureDisabledAt` field, closes the connection and sends an `authenticationError` webhook; the account then reports `unset`. Supplying working credentials (re-authorizing an OAuth2 account, or saving new IMAP settings) lifts it, as does the "Resume syncing" button on the account page in the admin interface. Since 2.79.3 it covers OAuth2 accounts as well as password accounts. See [Max IMAP auth failure time](/docs/configuration/environment-variables#max-imap-auth-failure-time).

### Send-Only Account

An account whose `imap.disabled` flag is set by the operator so that it sends mail but does not sync a mailbox. The account reports `sendOnly: true` and state `unset`, and `authFailureDisabledAt` stays `null`, which is what distinguishes it from an account the safety net switched off.

### Message ID

EmailEngine's unique identifier for a message, formatted as a Base64url-encoded string. Different from the email's Message-ID header.

### Sub-connection

A dedicated IMAP connection EmailEngine opens for an extra folder so changes there are detected as quickly as in the primary folder. Without one, a non-primary folder is only picked up on the periodic resync. Configured with the account's `subconnections` setting. Each one is a real IMAP connection, and providers cap how many an account may hold, so add them only for folders that genuinely need real-time updates.

### Path

The full path to a mailbox folder, such as `INBOX`, `Sent`, or `Work/Projects`. Used to identify folders in API requests.

### Special Path

A logical folder identifier that maps to actual folder names. Examples: `\Sent`, `\Trash`, `\Drafts`, `\Junk`. EmailEngine resolves these to actual paths automatically.

### Webhook

An HTTP callback that EmailEngine sends to your application when events occur (new message, status change, etc.). Configured globally or per-account.

### Webhook Event

A specific type of notification, such as `messageNew`, `messageSent`, `authenticationError`. See [Webhook Events Reference](/docs/reference/webhook-events) for complete list.

### Token (API)

An authentication credential for accessing the EmailEngine API. Created in the web interface under Integrations > Access Tokens.

### Token Scope

Permissions assigned to an API token. Valid scopes: `*` (full access), `api` (API access), `metrics` (Prometheus metrics only), `smtp` (SMTP gateway access), `imap-proxy` (IMAP proxy access), `mcp` ([MCP endpoint](/docs/mcp) access for AI agents). `POST /v1/tokens` accepts `api`, `smtp`, `imap-proxy` and `mcp`; a `metrics`-only token is issued from the admin interface or with `emailengine tokens issue -s metrics`.

### MCP (Model Context Protocol)

An open protocol that AI clients use to discover and call external tools. EmailEngine serves it at `POST /mcp`, exposing a curated tool set over the connected accounts. Off by default; see [MCP for AI Agents](/docs/mcp).

### MCP Tool

One callable operation an MCP client is offered, such as `list_messages` or `send_message`. Each tool wraps an EmailEngine API route and is dispatched with the caller's own access token, so REST permissions apply unchanged.

### MCP Access Level

The narrowing chosen when an MCP token is issued: read-only, mail agent (everything except deleting) or full access. Stored as an ordinary token permissions record.

### Service URL

The public URL where EmailEngine is accessible. Required for OAuth2 callbacks and hosted authentication forms. Set via `serviceUrl` setting.

### Prepared Settings

Configuration values that can be set via environment variables before EmailEngine starts, useful for automated deployments.

## Queue Terms

### Bull / BullMQ

The job queue system used by EmailEngine for background processing. Handles webhooks, email sending, and other async tasks.

### Queue

A named list of jobs waiting to be processed. EmailEngine uses separate queues for different task types: `notify` (webhooks), `submit` (outgoing mail) and `export` (mailbox exports).

### Job

A single task in a queue, such as sending a webhook or submitting an email. Jobs can be delayed, retried, or failed.

### Worker

A process that consumes jobs from a queue and executes them. EmailEngine runs multiple workers for parallel processing.

### Failed Jobs Set

Where failed jobs go after exhausting retry attempts. BullMQ does not have a separate dead letter queue - failed jobs remain in the queue's failed set, accessible via Bull Board for debugging.

## Gmail-Specific Terms

### Gmail API

Google's REST API for accessing Gmail, as opposed to IMAP/SMTP access. Provides native support for Gmail labels and categories.

### Cloud Pub/Sub

Google's messaging service used for Gmail API webhooks. Gmail pushes notifications to a Pub/Sub topic, which EmailEngine subscribes to.

### Gmail Labels

Gmail's tagging system for organizing emails. Unlike folders, a single email can have multiple labels. Exposed in EmailEngine webhooks for Gmail API accounts.

### Gmail Categories

Automatic inbox categorization (Primary, Social, Promotions, Updates, Forums). Available through Gmail API integration.

## Microsoft-Specific Terms

### Microsoft Graph API

Microsoft's unified API for accessing Microsoft 365 services including Outlook mail. EmailEngine can use Graph API instead of IMAP/SMTP.

### Azure AD / Entra ID

Microsoft's identity platform for OAuth2 authentication. Required for Outlook/Microsoft 365 OAuth2 integration.

### Shared Mailbox

A mailbox that multiple users can access, common in Microsoft 365. Configured using `delegatedUser` parameter in EmailEngine.

## Performance Terms

### Persistent Connection

EmailEngine maintains one persistent IMAP connection per account (plus optional subconnections for monitoring additional folders), rather than a pool of reusable connections. Keeping the connection open avoids the overhead of reconnecting and enables real-time IDLE notifications.

### Rate Limiting

Restrictions on how many requests can be made in a time period. Email providers enforce rate limits; EmailEngine handles them automatically with backoff.

### Backoff

A strategy for handling rate limits or errors by waiting progressively longer between retry attempts.

### Sync

The process of downloading message metadata and flags from an email server. EmailEngine performs initial sync when an account is added, then incremental syncs for changes.

### Full Sync

Re-downloading all message metadata for a mailbox, typically triggered by UIDVALIDITY changes or mailbox reconstruction.

## Security Terms

### TLS (Transport Layer Security)

Encryption protocol for secure communication. EmailEngine uses TLS for IMAP/SMTP connections and HTTPS for API access.

### STARTTLS

A method to upgrade a plain-text connection to TLS. Used by some IMAP/SMTP servers on standard ports.

### Encryption Secret

A key used by EmailEngine to encrypt sensitive data (passwords, tokens) stored in Redis. Set via `EENGINE_SECRET` environment variable.

### TOTP (Time-based One-Time Password)

A method for two-factor authentication using time-based codes. EmailEngine supports TOTP for admin login.

## Monitoring Terms

### Prometheus

An open-source monitoring system that collects metrics. EmailEngine exposes metrics at `/metrics` endpoint.

### Grafana

A visualization platform commonly used with Prometheus to create dashboards. Can display EmailEngine metrics.

### Health Check

An endpoint (`/health`) that returns EmailEngine's operational status, used by load balancers and monitoring systems.

### Metrics Token

An API token with only `metrics` scope, used by Prometheus to scrape the `/metrics` endpoint securely.

## See Also

- [Message IDs explained](/docs/advanced/ids-explained) - The identifiers named here, and which to store
- [Webhook events reference](/docs/reference/webhook-events) - Every event in one table
- [Configuration options](/docs/reference/configuration-options) - Every setting, by name
- [Quick reference](/docs/reference/quick-reference) - Endpoints, states, and codes at a glance

