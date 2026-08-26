---
title: Generic IMAP/SMTP Accounts
sidebar_position: 3
description: Setting up email accounts using standard IMAP and SMTP protocols
---

# Generic IMAP/SMTP Accounts

This guide covers setting up email accounts using standard IMAP and SMTP protocols. This method works with virtually any email provider that supports these protocols, including self-hosted mail servers, regional providers, and services not covered by specific OAuth2 integrations.

:::warning Credential Storage
When you add IMAP/SMTP accounts, EmailEngine stores the provided passwords in Redis. By default, these are stored in cleartext. For production deployments, configure `EENGINE_SECRET` to enable encryption **before** adding accounts. [Learn more](/docs/support/security-faq)
:::

## Overview

IMAP (Internet Message Access Protocol) and SMTP (Simple Mail Transfer Protocol) are the standard protocols for email:

- **IMAP**: Used for reading and managing emails
- **SMTP**: Used for sending emails

Unlike OAuth2-based setups, IMAP/SMTP accounts require direct credentials (username and password or app-specific passwords).

## When to Use IMAP/SMTP

**Best for:**

- Email providers without OAuth2 support
- Self-hosted mail servers
- Regional email providers
- Quick testing and development
- Legacy systems

**Advantages:**

- Universal compatibility
- Immediate setup (no OAuth2 app configuration)
- Works with virtually any provider
- Simple authentication

**Disadvantages:**

- Requires storing passwords
- May not work with accounts that have 2FA without app passwords
- Some providers (Gmail, Outlook) may block standard authentication

## Supported Providers

EmailEngine's IMAP/SMTP support works with hundreds of providers. Here are common examples:

### Major Providers with App Passwords

**Gmail:**

- Requires app-specific password if 2FA is enabled
- See [Gmail IMAP guide](./gmail/gmail-imap) for OAuth2 alternative

**Outlook.com / Hotmail.com / Live.com:**

- **OAuth2 required** - Microsoft has disabled regular password authentication
- Use [Outlook OAuth2 guide](./microsoft-365/outlook-365) to set up OAuth2 for IMAP/SMTP

**Yahoo Mail:**

- Requires app-specific password
- Generate at [Yahoo Account Security](https://login.yahoo.com/account/security)

**iCloud Mail:**

- Requires app-specific password
- Generate at [Apple ID Account](https://appleid.apple.com/)

### Self-Hosted Servers

IMAP/SMTP works with any standard-compliant mail server:

- **Postfix + Dovecot**
- **Microsoft Exchange**
- **Zimbra**
- **MDaemon**
- **MailEnable**
- **hMailServer**
- _etc._

## Server Settings Auto-Detection

EmailEngine can **automatically detect IMAP/SMTP server settings** for most email providers.

### Via Hosted Authentication Form

When users add accounts through the [hosted authentication form](/docs/accounts/hosted-authentication), EmailEngine attempts to automatically detect the correct server settings based on the email address. In most cases, users only need to enter their email and password. For self-hosted servers or less common providers where auto-detection fails, manual server configuration is required.

### Via API

Use the [autoconfig endpoint](/docs/api/get-v-1-autoconfig) to detect server settings programmatically:

```bash
curl "https://emailengine.example.com/v1/autoconfig?email=user@example.com" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

Response includes detected IMAP and SMTP settings:

```json
{
  "imap": {
    "host": "imap.example.com",
    "port": 993,
    "secure": true
  },
  "smtp": {
    "host": "smtp.example.com",
    "port": 587,
    "secure": false
  }
}
```

You can then use these settings when [registering an account](/docs/api/post-v-1-account).

:::warning Outlook.com / Hotmail.com / Live.com
Microsoft has **disabled regular password authentication** for consumer accounts. You must use OAuth2 authentication. See the [Outlook OAuth2 guide](./microsoft-365/outlook-365) for setup instructions.
:::

:::info Understanding the "secure" Setting
The `secure` setting controls how TLS is established:

- **`secure: true`** (ports 993/465) - TLS is used from the start of the connection (implicit TLS)
- **`secure: false`** (ports 143/587) - Connection starts unencrypted, then EmailEngine automatically upgrades to TLS using STARTTLS if the server supports it

Setting `secure: false` does **not** mean the connection is unencrypted. EmailEngine will use TLS in both cases - the difference is only in how TLS is negotiated.
:::

## Authentication Methods

### Standard Password

For providers that allow it, use your regular account password.

```json
{
  "account": "user123",
  "name": "John Doe",
  "email": "john@example.com",
  "imap": {
    "host": "imap.example.com",
    "port": 993,
    "secure": true,
    "auth": {
      "user": "john@example.com",
      "pass": "your-password"
    }
  },
  "smtp": {
    "host": "smtp.example.com",
    "port": 587,
    "secure": false,
    "auth": {
      "user": "john@example.com",
      "pass": "your-password"
    }
  }
}
```

### App-Specific Passwords

For providers requiring app passwords (Gmail, Yahoo, iCloud, AOL):

1. Enable 2FA on your account if not already enabled
2. Generate an app-specific password from your account settings
3. Use this app password instead of your main password

```json
{
  "account": "user123",
  "name": "John Doe",
  "email": "john@gmail.com",
  "imap": {
    "host": "imap.gmail.com",
    "port": 993,
    "secure": true,
    "auth": {
      "user": "john@gmail.com",
      "pass": "abcd efgh ijkl mnop"
    }
  },
  "smtp": {
    "host": "smtp.gmail.com",
    "port": 587,
    "secure": false,
    "auth": {
      "user": "john@gmail.com",
      "pass": "abcd efgh ijkl mnop"
    }
  }
}
```

### Different IMAP and SMTP Credentials

Some providers use different credentials for IMAP and SMTP:

```json
{
  "account": "user123",
  "imap": {
    "auth": {
      "user": "john@example.com",
      "pass": "imap-password"
    }
  },
  "smtp": {
    "auth": {
      "user": "john@example.com",
      "pass": "smtp-password"
    }
  }
}
```

## SSL/TLS Configuration

### Port and Security Options

**Port 993 with SSL/TLS (IMAP):**

```json
{
  "imap": {
    "port": 993,
    "secure": true
  }
}
```

**Port 143 with STARTTLS (IMAP):**

```json
{
  "imap": {
    "port": 143,
    "secure": false
  }
}
```

**Port 465 with SSL/TLS (SMTP):**

```json
{
  "smtp": {
    "port": 465,
    "secure": true
  }
}
```

**Port 587 with STARTTLS (SMTP):**

```json
{
  "smtp": {
    "port": 587,
    "secure": false
  }
}
```

### Self-Signed Certificates

For development or self-hosted servers with self-signed certificates, you can disable certificate verification.

**Per-account setting:**

Set `tls.rejectUnauthorized: false` when registering the account:

```json
{
  "imap": {
    "host": "mail.example.com",
    "port": 993,
    "secure": true,
    "tls": {
      "rejectUnauthorized": false
    }
  },
  "smtp": {
    "host": "mail.example.com",
    "port": 587,
    "secure": false,
    "tls": {
      "rejectUnauthorized": false
    }
  }
}
```

**Global setting for all accounts:**

To disable certificate verification for all IMAP/SMTP connections, set the `ignoreMailCertErrors` setting:

```bash
curl -X POST https://emailengine.example.com/v1/settings \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "ignoreMailCertErrors": true
  }'
```

Or via environment variable: `EENGINE_SETTINGS='{"ignoreMailCertErrors": true}'`

:::warning Security Warning
Only disable certificate verification for development/testing. In production, use proper SSL/TLS certificates (e.g., from Let's Encrypt).
:::

## Adding Accounts via API

### Basic IMAP/SMTP Account

Add accounts using the [Register Account API endpoint](/docs/api/post-v-1-account):

```bash
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user123",
    "name": "John Doe",
    "email": "john@example.com",
    "imap": {
      "host": "imap.example.com",
      "port": 993,
      "secure": true,
      "auth": {
        "user": "john@example.com",
        "pass": "password"
      }
    },
    "smtp": {
      "host": "smtp.example.com",
      "port": 587,
      "secure": false,
      "auth": {
        "user": "john@example.com",
        "pass": "password"
      }
    }
  }'
```

### IMAP-Only Account

For read-only access without sending capability:

```bash
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user123",
    "name": "John Doe",
    "email": "john@example.com",
    "imap": {
      "host": "imap.example.com",
      "port": 993,
      "secure": true,
      "auth": {
        "user": "john@example.com",
        "pass": "password"
      }
    }
  }'
```

### SMTP-Only Account (Send-Only)

For send-only access without IMAP:

```bash
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user123",
    "name": "John Doe",
    "email": "john@example.com",
    "smtp": {
      "host": "smtp.example.com",
      "port": 587,
      "secure": false,
      "auth": {
        "user": "john@example.com",
        "pass": "password"
      }
    }
  }'
```

## Send-Only Accounts

EmailEngine supports send-only accounts that can submit emails but cannot read or sync messages. This is useful for:

- Transactional email systems that only send notifications
- SMTP relay configurations
- Applications with limited OAuth2 scopes (e.g., `gmail.send` only)
- Accounts where IMAP access is not needed or not available

### Types of Send-Only Accounts

EmailEngine detects and handles three types of send-only configurations:

**1. SMTP-Only (No IMAP Configured)**

The simplest form - an account with only SMTP credentials and no IMAP section:

```bash
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "smtp-only-account",
    "name": "Notification Sender",
    "email": "notifications@example.com",
    "smtp": {
      "host": "smtp.example.com",
      "port": 587,
      "secure": false,
      "auth": {
        "user": "notifications@example.com",
        "pass": "password"
      }
    }
  }'
```

**2. IMAP Explicitly Disabled**

An account with IMAP credentials but explicitly disabled:

```bash
curl -X POST https://emailengine.example.com/v1/account \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "imap-disabled-account",
    "name": "John Doe",
    "email": "john@example.com",
    "imap": {
      "host": "imap.example.com",
      "port": 993,
      "secure": true,
      "disabled": true,
      "auth": {
        "user": "john@example.com",
        "pass": "password"
      }
    },
    "smtp": {
      "host": "smtp.example.com",
      "port": 587,
      "secure": false,
      "auth": {
        "user": "john@example.com",
        "pass": "password"
      }
    }
  }'
```

This is useful when you have full credentials but want to limit EmailEngine to sending only.

EmailEngine sets the same flag on its own when an account keeps failing authentication for three days. The account object's `authFailureDisabledAt` field (since v2.79.4) is a timestamp in that case and `null` when the operator disabled IMAP, so the two are distinguishable. In v2.79.4 such an account is reported as send-only everywhere; in releases after v2.79.4 the accounts listing and the admin interface keep its own type, because a switch-off is a fault rather than a configuration, while `GET /v1/account/{account}` still answers `sending` with `sendOnly: true`. `authFailureDisabledAt` is unambiguous on both. See [Accounts switched off after authentication failures](/docs/accounts/managing-accounts#accounts-switched-off-after-authentication-failures).

**3. OAuth2 with Send-Only Scopes**

OAuth2 accounts authenticated with only send scopes are automatically detected as send-only:

**Gmail with `gmail.send` scope only:**

When a Gmail OAuth2 application is configured with only the `gmail.send` scope (without `gmail.modify`, `gmail.readonly`, or `mail.google.com`), accounts using that application are treated as send-only.

**Outlook with `Mail.Send` scope only:**

Similarly, Outlook OAuth2 applications with only `Mail.Send` (without `Mail.Read`, `Mail.ReadWrite`, or IMAP scopes) result in send-only accounts.

EmailEngine automatically detects these scope configurations and marks accounts as send-only.

### Send-Only Account Behavior

Send-only accounts have these characteristics:

| Feature | Behavior |
|---------|----------|
| **Sending emails** | Fully supported |
| **IMAP sync** | Disabled (no connection established) |
| **Webhooks for new messages** | Not available |
| **Message search** | Not available |
| **Attachment downloads** | Not available |
| **Account state** | `unset`, since nothing is syncing |
| **Account type in API** | `sending` for an SMTP-only account; an OAuth2 account keeps `gmail` or `outlook` |
| **`sendOnly` flag** | Set to `true` in account info |

### Checking if an Account is Send-Only

The account info endpoint indicates send-only status:

```bash
curl "https://emailengine.example.com/v1/account/smtp-only-account" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

Response for send-only accounts:

```json
{
  "account": "smtp-only-account",
  "name": "Notification Sender",
  "email": "notifications@example.com",
  "type": "sending",
  "sendOnly": true,
  "state": "unset",
  "smtp": {
    "host": "smtp.example.com",
    "port": 587,
    "secure": false
  }
}
```

### Use Cases for Send-Only Accounts

**Transactional Email:**

Applications that send notifications, receipts, or alerts without needing to read responses:

```bash
curl -X POST https://emailengine.example.com/v1/account/notifications/submit \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from": {"name": "MyApp", "address": "notifications@example.com"},
    "to": [{"address": "user@customer.com"}],
    "subject": "Your order has shipped",
    "text": "Your order #12345 has been shipped and will arrive in 2-3 days."
  }'
```

**SMTP Relay:**

Using EmailEngine as an SMTP submission layer for existing applications that don't need inbox access.

**Limited OAuth2 Scopes:**

When Google or Microsoft only approves limited scopes during app verification, you can still use EmailEngine for sending while using another solution for reading emails.

## Adding Accounts via Hosted Authentication Form

Users can add their own accounts through EmailEngine's web interface:

```bash
curl -X POST https://emailengine.example.com/v1/authentication/form \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user123",
    "email": "john@example.com",
    "redirectUrl": "https://myapp.com/settings"
  }'
```

EmailEngine will generate a form URL where users can:

1. Enter their IMAP/SMTP settings
2. Test the connection
3. Complete setup

EmailEngine can auto-detect settings for many common providers.

[Learn more about hosted authentication →](/docs/accounts/hosted-authentication)

## Testing Connection

Before adding an account, verify credentials work:

```bash
curl -X POST https://emailengine.example.com/v1/verifyAccount \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "imap": {
      "host": "imap.example.com",
      "port": 993,
      "secure": true,
      "auth": {
        "user": "john@example.com",
        "pass": "password"
      }
    },
    "smtp": {
      "host": "smtp.example.com",
      "port": 587,
      "secure": false,
      "auth": {
        "user": "john@example.com",
        "pass": "password"
      }
    }
  }'
```

Response if successful:

```json
{
  "imap": {
    "success": true
  },
  "smtp": {
    "success": true
  }
}
```

## Advanced Configuration

### Custom Special Folder Paths {#custom-special-folder-paths}

EmailEngine works out which folder is Sent, Drafts, Junk, Trash, and Archive from three sources, in this order:

1. **Paths you set** (`specialUseSource: "user"`) - what this section is about, and what wins over everything else
2. **The server's SPECIAL-USE flags** (`specialUseSource: "extension"`) - the folder attributes an IMAP server advertises
3. **Folder names** (`specialUseSource: "name"`) - a guess from common names, used only when the first two say nothing

The [mailbox listing](/docs/api/get-v-1-account-account-mailboxes) reports `specialUseSource` per folder, so you can see which of the three decided a given folder before overriding anything.

Auto-detection can miss when:

- The server doesn't support the SPECIAL-USE extension and uses a folder name that EmailEngine's heuristics don't recognize
- The server uses localized folder names (e.g., Microsoft 365 over IMAP without SPECIAL-USE support may use language-specific names)
- A mail client created a folder at a different path than the server default

**Available folder path overrides:**

| Field | Purpose | Common Names |
|-------|---------|-------------|
| `sentMailPath` | Sent messages | `Sent`, `Sent Items`, `Sent Messages`, `[Gmail]/Sent Mail` |
| `draftsMailPath` | Draft messages | `Drafts`, `Draft` |
| `junkMailPath` | Spam/junk messages | `Junk`, `Spam`, `Junk E-mail`, `[Gmail]/Spam` |
| `trashMailPath` | Deleted messages | `Trash`, `Deleted Items`, `Deleted Messages`, `[Gmail]/Trash` |
| `archiveMailPath` | Archived messages | `Archive`, `Archives`, `[Gmail]/All Mail` |

Set any combination of these fields inside the `imap` object when registering or updating an account:

```json
{
  "imap": {
    "partial": true,
    "sentMailPath": "Sent Items",
    "draftsMailPath": "Drafts",
    "junkMailPath": "Junk E-mail",
    "trashMailPath": "Deleted Items",
    "archiveMailPath": "Archive"
  }
}
```

`partial` is what makes this an update rather than a replacement. Leave it out when registering a new account, where the whole `imap` object is being written anyway.

Set a field to `null` to revert to auto-detection:

```json
{
  "imap": {
    "partial": true,
    "sentMailPath": null
  }
}
```

### Resync Delay

EmailEngine performs a full mailbox resynchronization at regular intervals to ensure no messages are missed due to connection issues or IMAP protocol quirks. Between resyncs, real-time updates are handled via IMAP IDLE or periodic polling.

The default resync interval is 15 minutes (900 seconds). You can adjust this per account:

```json
{
  "imap": {
    "resyncDelay": 900
  }
}
```

| Value | Behavior |
|-------|----------|
| Lower (e.g., 300) | More frequent syncs, higher server load, fewer missed messages |
| Higher (e.g., 1800) | Less frequent syncs, lower server load, potential for briefly missed messages |

For most use cases, the default 15-minute interval provides a good balance.

### Disabling IMAP4rev2 {#disabling-imap4rev2}

EmailEngine uses IMAP4rev2 when the server advertises it. A few servers advertise the capability but break when it is actually used - this has been seen on Exchange, where connection setup fails. Set `imap.disableIMAP4rev2` to keep the account on IMAP4rev1 without giving up the other extensions EmailEngine negotiates:

```json
{
  "imap": {
    "disableIMAP4rev2": true
  }
}
```

In the admin interface the same option is the "Disable IMAP4rev2" checkbox on the account edit page, alongside "Allow invalid certificates" for the per-account `tls.rejectUnauthorized` override.

:::note Password accounts only
This option applies to accounts using regular password authentication. OAuth2 accounts build their connection settings from the provider configuration, so the field is ignored there.
:::

Available since EmailEngine v2.74.0.

### Path Filtering

Sync only specific folders:

```json
{
  "path": ["INBOX", "\\Sent", "\\Drafts", "Important"]
}
```

[Learn more about limiting indexed folders →](/docs/advanced/performance-tuning#limiting-indexed-folders)

### Sub-Connections

Monitor additional folders in real-time:

```json
{
  "subconnections": ["\\Sent", "Important"]
}
```

[Learn more about sub-connections →](/docs/advanced/performance-tuning#sub-connections-for-selected-folders)

### Proxy Configuration

Route EmailEngine's outbound IMAP/SMTP connections through a SOCKS or HTTP proxy server.

**Per-account proxy:**

Set the `proxy` field at the account level (not inside imap/smtp objects):

```json
{
  "account": "example",
  "proxy": "socks5://proxy.example.com:1080",
  "imap": {
    "host": "imap.example.com",
    "port": 993,
    "secure": true,
    "auth": {
      "user": "user@example.com",
      "pass": "password"
    }
  },
  "smtp": {
    "host": "smtp.example.com",
    "port": 587,
    "secure": false,
    "auth": {
      "user": "user@example.com",
      "pass": "password"
    }
  }
}
```

**Global proxy for all accounts:**

Configure a default proxy for all accounts via the settings API:

```bash
curl -X POST https://emailengine.example.com/v1/settings \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "proxyEnabled": true,
    "proxyUrl": "socks5://proxy.example.com:1080"
  }'
```

**Supported proxy protocols:**

- `socks5://` - SOCKS5 proxy (recommended)
- `socks4://` - SOCKS4 proxy
- `socks4a://` - SOCKS4a proxy
- `socks://` - Generic SOCKS proxy
- `http://` - HTTP proxy
- `https://` - HTTPS proxy

Per-account proxy settings override the global proxy setting.

:::warning IMAP and SMTP Only
The `proxyEnabled` and `proxyUrl` settings apply only to IMAP and SMTP socket connections. HTTP requests made by EmailEngine (webhooks, OAuth2 token exchange, Gmail API, Microsoft Graph API, license validation, etc.) are **not** routed through this proxy.

To proxy HTTP traffic, use the separate HTTP proxy settings instead: set `httpProxyEnabled` and `httpProxyUrl` via the settings API, or use the `EENGINE_HTTP_PROXY_ENABLED` and `EENGINE_HTTP_PROXY_URL` environment variables. Note that EmailEngine does not honor the standard `HTTP_PROXY`/`HTTPS_PROXY` environment variables.

```bash
curl -X POST https://emailengine.example.com/v1/settings \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "httpProxyEnabled": true,
    "httpProxyUrl": "http://proxy.example.com:3128"
  }'
```

For environments with strict outbound firewalls, see [Outbound Connection Whitelist](/docs/deployment/security#outbound-connection-whitelist) for a list of domains that EmailEngine needs to reach.
:::

## See Also

- [Managing accounts](/docs/accounts/managing-accounts) - Updating, pausing, and deleting an account once it exists
- [Hosted authentication](/docs/accounts/hosted-authentication) - Letting the user enter their own server settings
- [IMAP indexers](/docs/accounts/imap-indexers) - How EmailEngine detects change in a mailbox
- [Proxying connections](/docs/accounts/proxying-connections) - Routing mail traffic through a proxy
- [Troubleshooting accounts](/docs/accounts/troubleshooting) - When a connection or login fails
