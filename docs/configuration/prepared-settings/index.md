---
title: Prepared Settings
sidebar_position: 5
description: Pre-configure runtime settings, access tokens, and license keys via environment variables
---

# Prepared Settings

Prepared configuration allows you to pre-configure EmailEngine settings, access tokens, and license keys before the application starts. This is essential for automated deployments, CI/CD pipelines, and containerized environments where manual configuration is impractical.

## Overview

EmailEngine supports four types of prepared configuration:

1. **Prepared Settings** (`EENGINE_SETTINGS`) - Runtime configuration such as webhooks and the service URL, described on this page
2. **Prepared Access Tokens** (`EENGINE_PREPARED_TOKEN`) - An [exported API token](/docs/configuration/prepared-settings/tokens)
3. **Prepared License Keys** (`EENGINE_PREPARED_LICENSE`) - [License activation](/docs/configuration/prepared-settings/license)
4. **Prepared Admin Password** (`EENGINE_PREPARED_PASSWORD`) - A password hash produced by [`emailengine password --hash`](/docs/configuration/reset-password)

Each of these is read from the environment variable, from the matching `_FILE` variable (`EENGINE_SETTINGS_FILE` and so on, see [Loading values from files](/docs/configuration/environment-variables#loading-values-from-files)), or from the command line and configuration file under the keys `settings`, `preparedToken`, `preparedLicense` and `preparedPassword`.

Prepared configuration is processed on every startup, after the Redis connection is up and before the workers start. Settings, the license and the password are written each time, so a value changed through the API or the admin interface reverts to the prepared value at the next restart. A prepared token is imported only if its hash is not already stored; an existing token is left untouched.

## Use Cases

**Automated Deployments:**
- Docker/Kubernetes deployments
- Infrastructure as Code (Terraform, Ansible)
- CI/CD pipelines

**Testing:**
- End-to-end automated testing
- Integration test environments
- Staging environment setup

**Multi-Environment Setup:**
- Development, staging, production configs
- Multi-tenant deployments
- Rapid environment provisioning

## Prepared Settings

Pre-configure runtime settings that would normally be set via the Settings API or web interface.

### What Can Be Pre-Configured

Any setting available via the `/v1/settings` API endpoint:

- Webhook URLs and event filters
- SMTP server configuration (built-in SMTP server)
- Service URL and base URLs
- Tracking settings (clicks, opens)
- Logging configuration
- Proxy settings
- IMAP proxy configuration
- UI branding and locale settings

### Configuration Methods

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

<Tabs groupId="config-method">
<TabItem value="env" label="Environment Variable">

Set the `EENGINE_SETTINGS` environment variable with a JSON string:

```bash
export EENGINE_SETTINGS='{"webhooks": "https://webhook.site/abc123","webhookEvents":["messageNew"]}'
emailengine
```

</TabItem>
<TabItem value="cli" label="Command-Line">

Use the `--settings` flag:

```bash
emailengine --settings='{"webhooks": "https://your-app.com/webhook","webhookEvents":["messageNew"]}'
```

</TabItem>
<TabItem value="docker" label="Docker">

**Single-line environment variable:**
```dockerfile
ENV EENGINE_SETTINGS='{"webhooks":"https://your-app.com/webhook","webhookEvents":["messageNew","messageSent"]}'
```

</TabItem>
<TabItem value="docker-compose" label="Docker Compose">

**Multi-line YAML format (recommended):**
```yaml
services:
  emailengine:
    image: postalsys/emailengine:v2
    environment:
      EENGINE_SETTINGS: >
        {
          "webhooks": "https://your-app.com/webhook",
          "webhookEvents": [
            "messageNew",
            "messageDeleted",
            "messageSent",
            "messageDeliveryError"
          ],
          "notifyText": true,
          "serviceUrl": "https://emailengine.example.com"
        }
```

**Using external file:**
```yaml
services:
  emailengine:
    image: postalsys/emailengine:v2
    env_file:
      - ./config/emailengine.env
```

```bash
# config/emailengine.env
EENGINE_SETTINGS={"webhooks":"https://your-app.com/webhook","webhookEvents":["messageNew"]}
```

</TabItem>
</Tabs>

### Examples

**Basic webhook configuration:**
```bash
EENGINE_SETTINGS='{
  "webhooks": "https://your-app.com/webhook",
  "webhookEvents": ["messageNew", "messageSent"]
}'
```

`webhookEvents` is an allowlist with no default: without it, no webhooks are delivered. `["*"]` subscribes to every event.

**Complete configuration:**
```bash
EENGINE_SETTINGS='{
  "webhooks": "https://your-app.com/webhook",
  "webhookEvents": [
    "messageNew",
    "messageDeleted",
    "messageSent",
    "messageDeliveryError"
  ],
  "notifyText": true,
  "notifyTextSize": 1000,
  "serviceUrl": "https://emailengine.example.com"
}'
```

**SMTP server and tracking:**
```bash
EENGINE_SETTINGS='{
  "smtpServerEnabled": true,
  "smtpServerPort": 2525,
  "smtpServerHost": "0.0.0.0",
  "smtpServerAuthEnabled": true,
  "trackClicks": true,
  "trackOpens": true
}'
```

:::note OAuth2 Configuration
OAuth2 provider credentials (Gmail, Outlook) are configured as [OAuth2 applications](/docs/accounts/oauth2-setup), through the admin interface or `POST /v1/oauth2`, not through settings. The legacy settings keys such as `gmailClientId` are no longer read: an `EENGINE_SETTINGS` value that still carries them starts normally, but the keys are dropped and an error is logged naming them (since v2.79.1).
:::

### Validation

The JSON is parsed and validated against the settings schema (the same one `POST /v1/settings` uses) before EmailEngine connects to anything. Two outcomes are possible:

- **Invalid JSON or an invalid value** (a `webhooks` value that is not a URL, a string where a boolean is expected): EmailEngine logs a fatal `Invalid settings configuration provided` line and exits with status 1. The log entry carries the validation details, with secret values redacted.
- **Unknown keys** (a typo, or a key from an older release): the key is dropped, an error line `Ignoring unknown keys in the prepared settings` lists the dropped keys, and startup continues with the remaining settings.

Values are coerced the way the API coerces them, so `"notifyText": "true"` is accepted as a boolean.

### Updating Prepared Settings

Prepared settings are applied on every startup, overwriting existing values. To update:

1. Update the `EENGINE_SETTINGS` environment variable and restart EmailEngine
2. Or use the Settings API for runtime changes (will be overwritten on next restart if also defined in `EENGINE_SETTINGS`)

Only the keys present in `EENGINE_SETTINGS` are written. Removing a key from the variable does not clear the stored value; clear it through the API or the admin interface.

**Update settings via API:**
```bash
curl -X POST https://emailengine.example.com/v1/settings \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "webhooks": "https://new-webhook-url.com/webhook",
    "webhookEvents": ["messageNew", "messageSent"]
  }'
```

The response is `{"updated": ["webhooks", "webhookEvents"]}`. To clear the webhook URL, send an empty string (`{"webhooks": ""}`); `webhooks` does not accept `null`.

## See Also

- [Prepared Access Tokens](/docs/configuration/prepared-settings/tokens) - Pre-configure API access tokens
- [Prepared License](/docs/configuration/prepared-settings/license) - Pre-configure license keys
- [Reset Password](/docs/configuration/reset-password) - Generating the hash for `EENGINE_PREPARED_PASSWORD`
- [Environment Variables](/docs/configuration/environment-variables) - Complete environment variable reference
- [Settings API](/docs/api/post-v-1-settings) - Every settings key, its type and default
