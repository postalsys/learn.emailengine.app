---
title: License Keys
sidebar_position: 2
description: Pre-configure license keys via environment variables
---

# Prepared License

Pre-configure license keys for automated license activation.

## Manual vs Prepared Licensing

**Manual:**
1. Start EmailEngine
2. Log into the admin interface
3. Open **License** in the menu
4. Paste the license key, or click **Upload License File**
5. Click **Activate License**

**Prepared (automated):**
```bash
export EENGINE_PREPARED_LICENSE="your-license-key-here"
emailengine
```

At startup EmailEngine verifies the key and stores it in Redis, logging `License imported`. A key that does not verify, or one that has already expired, is fatal: the log shows `License import failed` and the process exits with status 13, so a wrong value stops the deployment rather than running it unlicensed. The `_FILE` form (`EENGINE_PREPARED_LICENSE_FILE=/run/secrets/license`) reads the key from a file, which is the simplest way to pass the multi-line format through a container secret.

## License Key Formats

EENGINE_PREPARED_LICENSE supports two formats:

### Format 1: Normal License Key (PEM Format)

The standard license key format provided by https://postalsys.com/:

```bash
export EENGINE_PREPARED_LICENSE="-----BEGIN LICENSE-----
Application: EmailEngine
Licensed to: Postal Systems OÜ

h6FspGM0NTSha6gwY2FlMjY2Y6Fus0V4YW1wbGUgQ29tcGFueSBJbmOhaKNBQ1OhYbFAZXhhbX
BsZS9kZW1vLWFwcKFjzwAAAZnw+ZsOoXPEK0VYQU1QTEVfU0lHTkFUVVJFX05PVF9WQUxJRF9G
T1JfUFVCTElDX0RFTU8=
-----END LICENSE-----"

emailengine
```

**This is the recommended format** - copy the license key exactly as shown in your account at https://postalsys.com/.

### Format 2: Exported License Key

A single-line form produced by the EmailEngine CLI. It is the base64url encoding of the license body, with the header lines and the `BEGIN`/`END` markers stripped, so it survives environments that cannot carry newlines:

```bash
emailengine license export --dbs.redis="redis://localhost:6379"
```

The command prints the encoded key on stdout without a trailing newline. Use it on another instance:

```bash
export EENGINE_PREPARED_LICENSE="i0-AgqFsxFWFoWvEDGC7abcdefghijklmnopqrstuvwxyz"
emailengine
```

EmailEngine tells the two formats apart by looking for `BEGIN LICENSE`: anything without that marker is treated as the encoded form.

**Use case:** Transfer license from one EmailEngine instance to another without accessing the license portal.

## Configuration Methods

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

<Tabs groupId="config-method">
<TabItem value="env" label="Environment Variable">

**PEM format (recommended):**
```bash
export EENGINE_PREPARED_LICENSE="-----BEGIN LICENSE-----
Application: EmailEngine
Licensed to: Your Company Name

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9abcdefghijklmnopqrstuvwxyz...
-----END LICENSE-----"

emailengine
```

**Exported format:**
```bash
export EENGINE_PREPARED_LICENSE="i0-AgqFsxFWFoWvEDGC7abcdefghijklmnopqrstuvwxyz"
emailengine
```

**From a file:**
```bash
export EENGINE_PREPARED_LICENSE_FILE=/etc/emailengine/license.txt
emailengine
```

</TabItem>
<TabItem value="cli" label="Command-Line">

**PEM format:**
```bash
emailengine --preparedLicense="-----BEGIN LICENSE-----
Application: EmailEngine
Licensed to: Your Company Name

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9abcdefghijklmnopqrstuvwxyz...
-----END LICENSE-----"
```

**Exported format:**
```bash
emailengine --preparedLicense="i0-AgqFsxFWFoWvEDGC7abcdefghijklmnopqrstuvwxyz"
```

The same key can be set in a TOML configuration file as `preparedLicense = "..."`.

</TabItem>
<TabItem value="docker" label="Docker">

A Dockerfile `ENV` line cannot carry newlines, so use the exported format, or mount the license file and point `EENGINE_PREPARED_LICENSE_FILE` at it:

```dockerfile
ENV EENGINE_PREPARED_LICENSE="i0-AgqFsxFWFoWvEDGC7abcdefghijklmnopqrstuvwxyz"
```

```bash
docker run -d \
  -v /srv/emailengine/license.txt:/run/secrets/license:ro \
  -e EENGINE_PREPARED_LICENSE_FILE=/run/secrets/license \
  -e EENGINE_REDIS=redis://redis:6379/8 \
  postalsys/emailengine:v2
```

</TabItem>
<TabItem value="docker-compose" label="Docker Compose">

**PEM format (multiline YAML):**
```yaml
services:
  emailengine:
    image: postalsys/emailengine:v2
    environment:
      EENGINE_PREPARED_LICENSE: |
        -----BEGIN LICENSE-----
        Application: EmailEngine
        Licensed to: Your Company Name

        eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9abcdefghijklmnopqrstuvwxyz...
        -----END LICENSE-----
```

**Exported format (single-line):**
```yaml
services:
  emailengine:
    image: postalsys/emailengine:v2
    environment:
      - EENGINE_PREPARED_LICENSE=${LICENSE_KEY}
```

```bash
# .env
LICENSE_KEY=i0-AgqFsxFWFoWvEDGC7abcdefghijklmnopqrstuvwxyz
```

</TabItem>
</Tabs>

## Verification

Check license status via API:

```bash
curl https://emailengine.example.com/v1/license \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**Response:**
```json
{
  "active": true,
  "details": {
    "application": "@postalsys/emailengine-app",
    "key": "1edf01e35e75ed3425808eba",
    "licensedTo": "Kreata OÜ",
    "hostname": "emailengine.example.com",
    "created": "2021-10-13T07:47:42.695Z"
  },
  "type": "EmailEngine License",
  "suspended": false
}
```

Without a license, `active` is `false` and `details` is `false`. The startup log also reports the outcome: `License imported` on success, and `No active license. Running in limited mode.` when nothing valid is stored.

## License Management

**Update license:**
1. Change `EENGINE_PREPARED_LICENSE` environment variable
2. Restart EmailEngine

The prepared key is imported on every startup, so the new license replaces the stored one at the next restart.

**Remove license via API:**
```bash
curl -X DELETE https://emailengine.example.com/v1/license \
  -H "Authorization: Bearer YOUR_TOKEN"
```

While `EENGINE_PREPARED_LICENSE` stays set, the key returns at the next restart. Remove the variable as well to keep the instance unlicensed.

## See Also

- [Prepared Settings](/docs/configuration/prepared-settings) - Pre-configure runtime settings
- [Prepared Access Tokens](/docs/configuration/prepared-settings/tokens) - Pre-configure API access tokens
- [CLI reference](/docs/configuration/cli#license-management) - `emailengine license export` and `import`
- [License API](/docs/api/get-v-1-license) - Reading, registering and removing the key over the API
- [Licensing](/docs/licensing) - License terms and how the key is validated
