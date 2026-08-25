---
title: Windows Installation
description: Install EmailEngine on Windows using standalone executable or WSL2
sidebar_position: 4
---

# Installing EmailEngine on Windows

Two ways to run EmailEngine on Windows: the native executable with a Redis-compatible server, or a Linux install inside WSL2.

## Overview

EmailEngine can be installed on Windows using two methods:

1. **Windows Executable** - Native Windows binary with Memurai or Redis alternatives
2. **WSL2** - Linux installation inside Windows Subsystem for Linux

:::tip Production Recommendation
For production deployments on Windows Server, consider using **WSL2** or running EmailEngine on a dedicated Linux VM for better performance and reliability.
:::

### Privileges

EmailEngine does not require administrator privileges to run. You can run it as a regular user on any unprivileged port (e.g., 3000).

Administrator access is only needed during initial setup to:

- Install Memurai/Redis as a Windows service
- Create a Windows service for EmailEngine (optional)
- Bind the SMTP or IMAP proxy to privileged ports (below 1024, such as 465 or 993)

Once installed, EmailEngine can run as an unprivileged user. For privileged ports, consider using a reverse proxy to forward traffic from privileged ports to EmailEngine.

## Method 1: Windows Executable

Native Windows installation using the standalone executable.

### Step 1: Install Redis Alternative

EmailEngine requires Redis, which doesn't officially support Windows. Use one of these alternatives:

#### Option A: Memurai (Recommended)

[Memurai](https://www.memurai.com/) is a Redis-compatible server for Windows.

1. Download Memurai from [https://www.memurai.com/get-memurai](https://www.memurai.com/get-memurai)
2. Run the installer (requires admin rights)
3. Memurai starts automatically as a Windows service

**Verify installation:**

```powershell
# Using Memurai CLI
memurai-cli ping
# Should return: PONG
```

#### Option B: Redis on WSL2

If you have WSL2 installed:

```powershell
# Start WSL2
wsl

# Install Redis in WSL2
sudo apt update
sudo apt install redis-server
sudo service redis-server start

# Exit WSL2
exit
```

Connect to WSL2 Redis from Windows using `redis://localhost:6379`

#### Option C: Redis Docker Container

```powershell
# Pull Redis image
docker pull redis:latest

# Run Redis
docker run -d --name redis -p 6379:6379 redis:latest

# Verify
docker exec redis redis-cli ping
```

### Step 2: Download EmailEngine

Open PowerShell and download the Windows executable:

```powershell
# Download latest version
Invoke-WebRequest -Uri "https://go.emailengine.app/emailengine.exe" -OutFile "emailengine.exe"

# Or for specific version (e.g., 2.79.3)
Invoke-WebRequest -Uri "https://go.emailengine.app/download/v2.79.3/emailengine.exe" -OutFile "emailengine.exe"
```

**Alternative: Browser download**

1. Visit [https://github.com/postalsys/emailengine/releases/latest](https://github.com/postalsys/emailengine/releases/latest)
2. Download `emailengine.exe`
3. Save to a permanent location (e.g., `C:\Program Files\EmailEngine\`)

### Step 3: Configure EmailEngine

Generate secrets and create `.env` file in the same directory as `emailengine.exe`:

```powershell
# Generate random secret (64 characters)
$secret = -join ((48..57) + (65..90) + (97..122) | Get-Random -Count 64 | ForEach-Object {[char]$_})

# Create .env file with configuration
@"
EENGINE_REDIS=redis://127.0.0.1:6379/8
EENGINE_SECRET=$secret
EENGINE_WORKERS=4
EENGINE_PORT=3000
EENGINE_LOG_LEVEL=info
"@ | Out-File -FilePath ".env" -Encoding UTF8

Write-Host "Configuration saved to .env file"
Write-Host "EENGINE_SECRET=$secret"
```

**Important:** The `.env` file is created in the current directory and will be automatically loaded by `emailengine.exe` when it starts. Keep this file secure and back it up.

### Step 4: Test Run

```powershell
# Run EmailEngine (it will automatically load .env file)
.\emailengine.exe
```

In another PowerShell window:

```powershell
Invoke-WebRequest -Uri "http://localhost:3000/health" | Select-Object -Expand Content
# Should return: {"success":true}
```

### Step 5: Run as Windows Service

To run EmailEngine as a Windows service, use [NSSM (Non-Sucking Service Manager)](https://nssm.cc/).

**Install NSSM:**

```powershell
# Using Chocolatey
choco install nssm

# Or download from https://nssm.cc/download
```

**Create service:**

First, ensure your `.env` file is in `C:\Program Files\EmailEngine\` directory.

```powershell
# Install service via command line
nssm install EmailEngine "C:\Program Files\EmailEngine\emailengine.exe"
nssm set EmailEngine AppDirectory "C:\Program Files\EmailEngine"
nssm set EmailEngine DisplayName "EmailEngine"
nssm set EmailEngine Description "EmailEngine IMAP/SMTP API service"
nssm set EmailEngine Start SERVICE_AUTO_START

# Start service
nssm start EmailEngine

# Check status
nssm status EmailEngine
```

**Note:** EmailEngine will automatically load configuration from the `.env` file in its working directory (`C:\Program Files\EmailEngine\`). No need to set environment variables in NSSM.

**Service management:**

```powershell
# Start service
nssm start EmailEngine

# Stop service
nssm stop EmailEngine

# Restart service
nssm restart EmailEngine

# Remove service
nssm remove EmailEngine confirm
```

### Upgrading Windows Installation

```powershell
# Stop service
nssm stop EmailEngine

# Download latest version
Invoke-WebRequest -Uri "https://go.emailengine.app/emailengine.exe" -OutFile "emailengine.exe"

# Replace existing binary
Move-Item -Path "emailengine.exe" -Destination "C:\Program Files\EmailEngine\emailengine.exe" -Force

# Start service
nssm start EmailEngine

# Verify version
& "C:\Program Files\EmailEngine\emailengine.exe" --version
```

## Method 2: WSL2 Installation (Recommended for Production)

Windows Subsystem for Linux 2 provides better performance and compatibility.

### Step 1: Enable WSL2

```powershell
# Run as Administrator
wsl --install

# Restart your computer

# Set WSL2 as default
wsl --set-default-version 2

# Install Ubuntu
wsl --install -d Ubuntu-22.04
```

### Step 2: Install EmailEngine in WSL2

```powershell
# Start WSL2
wsl
```

Inside WSL2, follow the [Linux installation guide](/docs/installation/linux):

```bash
# Quick install using automated installer
wget https://go.emailengine.app -O install.sh
chmod +x install.sh
sudo ./install.sh

# Or manual installation
sudo apt update
sudo apt install redis-server
wget https://go.emailengine.app/emailengine.tar.gz
tar xzf emailengine.tar.gz
sudo mv emailengine /usr/local/bin/
```

### Step 3: Access from Windows

EmailEngine running in WSL2 is accessible from Windows at:

- `http://localhost:3000` (if configured on port 3000)

### Step 4: Auto-start with Windows

Create a Windows startup script to launch EmailEngine in WSL2.

**Create `start-emailengine.bat` in `C:\Program Files\EmailEngine\`:**

```batch
@echo off
wsl sudo service redis-server start
wsl sudo systemctl start emailengine
```

The second line assumes EmailEngine is installed as a service inside WSL2. The automated installer above creates it; a manual binary install does not, so create the [SystemD unit](/docs/deployment/systemd) first or start the binary directly.

**Add to Windows startup:**

1. Press `Win + R`
2. Type `shell:startup` and press Enter
3. Create a shortcut to `start-emailengine.bat`

### Why WSL2

- Runs the Linux build, which is the one that gets the most testing
- Redis runs natively rather than through a compatible reimplementation
- The Linux installation guide, the SystemD unit, and the upgrade script all apply unchanged

## Configuration Options

### Using .env File (Recommended)

Create `.env` file in the same directory as `emailengine.exe`:

```bash
# Required
EENGINE_REDIS=redis://127.0.0.1:6379/8
EENGINE_SECRET=your-secret-key-at-least-32-chars

# Optional
EENGINE_WORKERS=4
EENGINE_PORT=3000
EENGINE_LOG_LEVEL=info
EENGINE_LOG_RAW=false
```

EmailEngine automatically loads this file on startup. This is the recommended approach for Windows installations.

### Alternative: TOML Configuration File

You can also use `config.toml` in the same directory as `emailengine.exe`:

```toml
[service]
secret = "your-secret-key-at-least-32-chars"

[dbs]
redis = "redis://127.0.0.1:6379/8"

[api]
port = 3000
host = "0.0.0.0"

workers = 4

[log]
level = "info"
raw = false
```

Run with config file:

```powershell
.\emailengine.exe --config="C:\Program Files\EmailEngine\config.toml"
```

**Note:** The same file can be named by the `NODE_CONFIG_PATH` environment variable instead of `--config`; both fill the same slot. A TOML file can carry every setting, `service.secret` included, so nothing else has to be passed on the command line.

## See Also

- [Linux installation](/docs/installation/linux) - What the WSL2 path follows
- [Configuration](/docs/configuration) - Environment variables, TOML files, and prepared settings
- [SystemD service](/docs/deployment/systemd) - Running as a service inside WSL2
- [Troubleshooting](/docs/troubleshooting) - Common startup problems
