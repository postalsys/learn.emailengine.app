---
title: macOS Installation
description: Install EmailEngine on macOS using the signed PKG installer or from source
sidebar_position: 3
---

# Installing EmailEngine on macOS

Two ways to run EmailEngine on macOS: the signed and notarized PKG installer, or from source. Both work on Apple Silicon and Intel.

## Overview

EmailEngine can be installed on macOS using two methods:

1. **PKG Installer** - One package for Apple Silicon and one for Intel Macs
2. **Source Installation** - Run from source (requires Node.js 20+, recommended 24+)

### System Requirements

The figures on the [installation overview](/docs/installation#system-requirements) apply: 2 GB of memory to evaluate, 4 to 8 GB for production. The packaged binary bundles a Node.js 24 runtime, so it runs on any macOS release that Node.js 24 supports.

### Required Software

- **Redis** (install via Homebrew), stand-alone, with `maxmemory-policy noeviction`
- **Node.js 20+** (only for source installation)
- **Homebrew** (recommended for Redis installation)

### Privileges

EmailEngine does not require administrator privileges to run. You can run it as a regular user on any unprivileged port (e.g., 3000).

Administrator access is only needed during initial setup to:
- Install the PKG package to `/usr/local/bin`
- Create directories in `/opt` (if using source installation)
- Bind the SMTP or IMAP proxy to privileged ports (below 1024, such as 465 or 993)

Once installed, EmailEngine runs as an unprivileged user. For privileged ports, consider using a reverse proxy (Nginx, Caddy) to forward traffic from privileged ports to EmailEngine.

## Method 1: PKG Installer

The easiest way to install EmailEngine on macOS.

### Download Installer

**For Apple Silicon (M1 and newer):**
```bash
# Download latest version
curl -LO https://go.emailengine.app/emailengine-arm.pkg

# Or download specific version (e.g., 2.79.4)
curl -LO https://go.emailengine.app/download/v2.79.4/emailengine-arm.pkg
```

**For Intel Macs:**
```bash
# Download latest version
curl -LO https://go.emailengine.app/emailengine.pkg

# Or download specific version (e.g., 2.79.4)
curl -LO https://go.emailengine.app/download/v2.79.4/emailengine.pkg
```

The package is signed with a Developer ID Installer certificate and notarized by Apple, so Gatekeeper opens it without an override. To check before installing:

```bash
pkgutil --check-signature emailengine-arm.pkg
spctl -a -vv -t install emailengine-arm.pkg
```

The first command names the certificate and reports `Notarization: trusted by the Apple notary service`; the second answers `accepted` with `source=Notarized Developer ID`.

### Install

1. Double-click the downloaded `.pkg` file
2. Follow the installation wizard
3. The package (identifier `com.postalsys.emailengine`) installs a single file, `/usr/local/bin/emailengine`, and asks for an administrator password because that directory is not user-writable. It runs no install scripts and does not create a launch agent or any configuration; starting EmailEngine at login is set up separately below

Or install from the terminal:

```bash
sudo installer -pkg emailengine-arm.pkg -target /
```

### Verify Installation

```bash
emailengine --version
```

### Install Redis

The PKG installer includes EmailEngine but not Redis. Install Redis separately:

```bash
# Install Homebrew if not already installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Redis
brew install redis

# Start Redis
brew services start redis

# Verify
redis-cli ping  # Should return: PONG
```

### Configure and Run

Generate and save encryption secret:
```bash
# Generate a random secret (minimum 32 characters) and save to environment file
echo "EENGINE_SECRET=$(openssl rand -hex 32)" > ~/.emailengine.env
echo "EENGINE_REDIS=redis://127.0.0.1:6379/8" >> ~/.emailengine.env

# Load the environment variables
source ~/.emailengine.env
```

**Important:** Save `~/.emailengine.env` securely. You must source this file and use the same secret every time you start EmailEngine.

Start EmailEngine:
```bash
# Load environment variables first
source ~/.emailengine.env

emailengine \
  --dbs.redis="$EENGINE_REDIS" \
  --service.secret="$EENGINE_SECRET" \
  --api.port=3000
```

Test the installation:
```bash
curl http://localhost:3000/health
# Should return: {"success":true}
```

### Run as Launch Agent

For automatic startup, create a launch agent plist.

Create `~/Library/LaunchAgents/com.emailengine.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.emailengine</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/emailengine</string>
    </array>
    <key>EnvironmentVariables</key>
    <dict>
        <key>EENGINE_REDIS</key>
        <string>redis://127.0.0.1:6379/8</string>
        <key>EENGINE_SECRET</key>
        <string>your-encryption-secret-from-openssl-rand-hex-32</string>
        <key>EENGINE_WORKERS</key>
        <string>4</string>
    </dict>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/usr/local/var/log/emailengine.log</string>
    <key>StandardErrorPath</key>
    <string>/usr/local/var/log/emailengine.error.log</string>
</dict>
</plist>
```

On Apple Silicon, Homebrew lives under `/opt/homebrew` rather than `/usr/local`, and `/usr/local/var/log` may not exist. Point `StandardOutPath` and `StandardErrorPath` at a directory that does, for example `~/Library/Logs/emailengine.log`.

Load and start:
```bash
launchctl load ~/Library/LaunchAgents/com.emailengine.plist
launchctl start com.emailengine
```

Check status:
```bash
launchctl list | grep emailengine
tail -f /usr/local/var/log/emailengine.log
```

## Method 2: Source Installation

Running from source is recommended for production as it uses less memory than the binary. The binary uses a virtual filesystem that loads all files into memory at startup, while source installation keeps files on disk.

For complete source installation instructions, including Node.js setup, Launch Agent configuration, and upgrade procedures, see the dedicated **[Source Installation Guide](/docs/installation/source)**.

## See Also

- [Source installation](/docs/installation/source) - Running from source on any platform
- [Configuration](/docs/configuration) - Environment variables and prepared settings
- [Quick Start](/docs/getting-started/quick-start) - Connecting the first account
- [Troubleshooting](/docs/troubleshooting) - Common startup problems
