---
title: macOS Installation
description: Install EmailEngine on macOS using PKG installer or Homebrew
sidebar_position: 3
---

# Installing EmailEngine on macOS

Two ways to run EmailEngine on macOS: the signed PKG installer, or from source. Both work on Apple Silicon and Intel.

## Overview

EmailEngine can be installed on macOS using two methods:

1. **PKG Installer** - Graphical installer for both Apple Silicon and Intel Macs
2. **Source Installation** - Run from source (requires Node.js 20+, recommended 24+)

### System Requirements

**Minimum (development/testing):**
- macOS 11 (Big Sur) or later
- 2 GB RAM
- 10 GB storage

**Recommended (production):**
- macOS 12 (Monterey) or later
- 4-8 GB RAM or more
- 20+ GB SSD storage

### Required Software

- **Redis 6.0+** (install via Homebrew)
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

# Or download specific version (e.g., 2.79.3)
curl -LO https://go.emailengine.app/download/v2.79.3/emailengine-arm.pkg
```

**For Intel Macs:**
```bash
# Download latest version
curl -LO https://go.emailengine.app/emailengine.pkg

# Or download specific version (e.g., 2.79.3)
curl -LO https://go.emailengine.app/download/v2.79.3/emailengine.pkg
```

### Install

1. Double-click the downloaded `.pkg` file
2. Follow the installation wizard
3. Installer will place binary at `/usr/local/bin/emailengine`

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
        <string>redis://127.0.0.1:6379</string>
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
