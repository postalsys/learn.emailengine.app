---
title: SystemD Service
description: Run EmailEngine as a SystemD service on Linux servers with automatic restart
sidebar_position: 3
---

# SystemD Service Deployment

Run EmailEngine as a background service on Linux systems using SystemD.

:::info Prerequisites
- Linux system with SystemD (Ubuntu 16.04+, Debian 8+, CentOS 7+, RHEL 7+)
- Node.js 20+ installed (24+ recommended) for source installation
- Redis 6.0+ installed and running
- EmailEngine binary installed in `/usr/local/bin/` or `/opt/emailengine`
:::

## Overview

SystemD is the standard init system for most modern Linux distributions. Running EmailEngine as a SystemD service provides:

- **Automatic startup** on system boot
- **Process management** (restart on failure)
- **Log management** with journald
- **Resource limits** and security isolation
- **Service dependencies** (start after Redis)

EmailEngine is well-suited for SystemD because it:
- Doesn't fork itself (runs in foreground)
- Logs to stdout/stderr (captured by journald)
- Responds to SIGTERM for graceful shutdown

## Basic Setup

### 1. Install EmailEngine

Choose one of the following installation methods:

**Option A: Download Binary (Recommended)**

```bash
# Download and extract
wget https://go.emailengine.app/emailengine.tar.gz
tar xzf emailengine.tar.gz
sudo mv emailengine /usr/local/bin/
sudo chmod +x /usr/local/bin/emailengine

# Verify installation
emailengine --version
```

**Option B: Using Install Script (Ubuntu/Debian)**

```bash
# Download and run install script
wget https://go.emailengine.app -O install.sh
chmod +x install.sh
sudo ./install.sh

# Verify installation
emailengine --version
```

**Option C: From Source**

```bash
# Download source distribution
wget https://go.emailengine.app/source-dist.tar.gz
tar xzf source-dist.tar.gz
cd emailengine

# Install dependencies
npm install --omit=dev

# Make globally available
sudo npm link

# Verify installation
emailengine --version
```

### 2. Create Service User

Create dedicated user for security:

```bash
# Create system user
sudo useradd --system --no-create-home --shell /bin/false emailengine

# Or with home directory for config files
sudo useradd --system --home /opt/emailengine --shell /bin/false emailengine
```

### 3. Create Configuration Directory

```bash
# Create directories
sudo mkdir -p /etc/emailengine
sudo mkdir -p /var/log/emailengine

# Set permissions
sudo chown emailengine:emailengine /etc/emailengine
sudo chown emailengine:emailengine /var/log/emailengine
```

### 4. Configuration Options

EmailEngine can be configured via environment variables (recommended for SystemD) or a TOML configuration file.

:::info Environment Variables Recommended
For SystemD deployments, environment variables in the service file are simpler and more secure than config files. See the service file example below.
:::

**Optional: Create TOML configuration file** `/etc/emailengine/config.toml`:

```toml
[dbs]
redis = "redis://localhost:6379/8"

[api]
port = 3000
host = "127.0.0.1"

[workers]
imap = 4

[log]
level = "info"
```

**Set permissions:**
```bash
sudo chown emailengine:emailengine /etc/emailengine/config.toml
sudo chmod 640 /etc/emailengine/config.toml
```

:::warning TOML Format Required
EmailEngine uses TOML configuration files, NOT JSON. If using a config file, it must have `.toml` extension and use TOML syntax.
:::

### 5. Create SystemD Service File

Create `/etc/systemd/system/emailengine.service`:

```ini
[Unit]
Description=EmailEngine Email API Service
Documentation=https://emailengine.app
After=network.target redis.service
Requires=redis.service

[Service]
Type=simple
User=emailengine
Group=emailengine

# Working directory (where EmailEngine is installed)
WorkingDirectory=/opt/emailengine

# Start command - use binary directly, no --config flag
ExecStart=/usr/local/bin/emailengine

# Restart policy
Restart=always
RestartSec=5
StartLimitInterval=300
StartLimitBurst=5

# Environment variables (recommended configuration method)
Environment="NODE_ENV=production"
Environment="EENGINE_REDIS=redis://localhost:6379/8"
Environment="EENGINE_SECRET=your-secret-key-at-least-32-characters"
Environment="EENGINE_WORKERS=4"

# Optional: Use config file instead of environment variables
# Environment="NODE_CONFIG_PATH=/etc/emailengine/config.toml"

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=emailengine

# Security hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/log/emailengine

# Resource limits
LimitNOFILE=500000
LimitNPROC=500000
LimitFSIZE=infinity

[Install]
WantedBy=multi-user.target
```

:::warning Security
Replace `your-secret-key-at-least-32-characters` with a strong random string.
:::

### 6. Enable and Start Service

```bash
# Reload systemd configuration
sudo systemctl daemon-reload

# Enable service to start on boot
sudo systemctl enable emailengine

# Start service
sudo systemctl start emailengine

# Check status
sudo systemctl status emailengine
```

## Configuration Options

### Environment Variables

**Method 1: In service file**

```ini
[Service]
Environment="EENGINE_REDIS=redis://localhost:6379/8"
Environment="EENGINE_SECRET=your-secret-key"
Environment="EENGINE_WORKERS=4"
```

**Method 2: Multiple environment variables in service file**

Edit `/etc/systemd/system/emailengine.service`:

```ini
[Service]
Environment="EENGINE_REDIS=redis://localhost:6379/8"
Environment="EENGINE_SECRET=your-secret-key-at-least-32-characters"
Environment="EENGINE_WORKERS=4"
Environment="EENGINE_LOG_LEVEL=info"
```

**Apply changes:**
```bash
sudo systemctl daemon-reload
sudo systemctl restart emailengine
```

### Configuration File (TOML)

**Optional: Complete `/etc/emailengine/config.toml`:**

```toml
[dbs]
redis = "redis://localhost:6379/8"

[api]
port = 3000
host = "127.0.0.1"
proxy = true

[workers]
imap = 4

[log]
level = "info"
```

:::tip Environment Variables Preferred
For sensitive values like secrets and OAuth credentials, use environment variables in the service file rather than config files. This is more secure and easier to manage.
:::

## Service Management

### Basic Commands

```bash
# Start service
sudo systemctl start emailengine

# Stop service
sudo systemctl stop emailengine

# Restart service
sudo systemctl restart emailengine

# Reload configuration (if supported)
sudo systemctl reload emailengine

# Check status
sudo systemctl status emailengine

# Enable auto-start on boot
sudo systemctl enable emailengine

# Disable auto-start
sudo systemctl disable emailengine
```

### Check Service Status

```bash
# Detailed status
sudo systemctl status emailengine

# Check if running
sudo systemctl is-active emailengine

# Check if enabled
sudo systemctl is-enabled emailengine

# Show service properties
sudo systemctl show emailengine
```

**Example status output:**

```
● emailengine.service - EmailEngine Email API Service
   Loaded: loaded (/etc/systemd/system/emailengine.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2025-10-13 10:00:00 UTC; 2h 30min ago
     Docs: https://emailengine.app
 Main PID: 12345 (node)
    Tasks: 15 (limit: 4915)
   Memory: 512.5M (limit: 2.0G)
   CGroup: /system.slice/emailengine.service
           └─12345 /usr/bin/node /usr/local/bin/emailengine
```

## Log Management

### View Logs with Journalctl

```bash
# View recent logs
sudo journalctl -u emailengine

# Follow logs in real-time
sudo journalctl -u emailengine -f

# View logs from last boot
sudo journalctl -u emailengine -b

# View logs from specific time
sudo journalctl -u emailengine --since "2025-10-13 10:00:00"
sudo journalctl -u emailengine --since "1 hour ago"

# View last 100 lines
sudo journalctl -u emailengine -n 100

# View logs with priority (errors only)
sudo journalctl -u emailengine -p err

# Export logs to file
sudo journalctl -u emailengine > emailengine.log
```

### Log Rotation

**SystemD automatically rotates journal logs**, but you can configure retention:

Edit `/etc/systemd/journald.conf`:

```ini
[Journal]
SystemMaxUse=1G
SystemMaxFileSize=100M
MaxRetentionSec=7day
```

**Apply changes:**
```bash
sudo systemctl restart systemd-journald
```

### File-Based Logging

**Configure via environment variables:**

```ini
[Service]
Environment="EENGINE_LOG_LEVEL=info"
```

Or in TOML config file:

```toml
[log]
level = "info"
```

**Set up logrotate:**

Create `/etc/logrotate.d/emailengine`:

```
/var/log/emailengine/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 emailengine emailengine
    sharedscripts
    postrotate
        /bin/systemctl reload emailengine > /dev/null 2>&1 || true
    endscript
}
```

## Security Hardening

### Minimal Service File

```ini
[Unit]
Description=EmailEngine Email API Service
After=network.target redis.service
Requires=redis.service

[Service]
Type=simple
User=emailengine
Group=emailengine
WorkingDirectory=/opt/emailengine
ExecStart=/usr/local/bin/emailengine
Restart=always

# Security
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/log/emailengine
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
RestrictRealtime=true
RestrictNamespaces=true
LockPersonality=true
MemoryDenyWriteExecute=true
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX

# Capabilities
CapabilityBoundingSet=
AmbientCapabilities=

# System calls
SystemCallFilter=@system-service
SystemCallFilter=~@privileged @resources
SystemCallErrorNumber=EPERM

[Install]
WantedBy=multi-user.target
```

### File Permissions

```bash
# Service file
sudo chmod 644 /etc/systemd/system/emailengine.service

# Configuration files (if using TOML config)
sudo chmod 640 /etc/emailengine/config.toml
sudo chown root:emailengine /etc/emailengine/config.toml

# Environment file (secrets)
sudo chmod 640 /etc/emailengine/environment
sudo chown root:emailengine /etc/emailengine/environment

# Log directory
sudo chmod 750 /var/log/emailengine
sudo chown emailengine:emailengine /var/log/emailengine
```

### Resource Limits

**CPU limits:**
```ini
[Service]
CPUQuota=200%          # Max 2 CPU cores
CPUWeight=100          # Priority (1-10000)
```

**Memory limits:**
```ini
[Service]
MemoryLimit=2G         # Hard limit
MemoryHigh=1.5G        # Soft limit (throttling)
```

**File descriptor limits:**
```ini
[Service]
LimitNOFILE=500000     # Max open files (matches official config)
LimitNPROC=500000      # Max processes
LimitFSIZE=infinity    # No file size limit
```

**IO limits:**
```ini
[Service]
IOWeight=500           # IO priority
IOReadBandwidthMax=/var 10M
IOWriteBandwidthMax=/var 10M
```

### Automatic Restart on Memory Threshold

SystemD can automatically restart EmailEngine when memory usage exceeds a defined threshold. This is useful for preventing runaway memory usage from causing system-wide issues and ensuring long-term stability.

**Configure memory-based automatic restart:**

```ini
[Service]
# Memory threshold - service is killed when exceeded
MemoryMax=4G

# Ensure service restarts after being killed
Restart=always
RestartSec=5
```

When EmailEngine's memory usage exceeds `MemoryMax`, SystemD terminates the process with an OOM (Out of Memory) kill signal. Combined with `Restart=always`, the service automatically restarts with fresh memory allocation.

**Complete example with memory management:**

```ini
[Unit]
Description=EmailEngine Email API Service
After=network.target redis.service
Requires=redis.service

[Service]
Type=simple
User=emailengine
Group=emailengine
WorkingDirectory=/opt/emailengine
ExecStart=/usr/local/bin/emailengine

# Memory management - kill and restart when exceeded
MemoryMax=4G

# Restart policy
Restart=always         # Always restart (including after OOM kill)
RestartSec=5           # Wait 5 seconds before restart
StartLimitInterval=300 # Rate limit: max restarts within 5 minutes
StartLimitBurst=5      # Max 5 restarts within interval

# Environment
Environment="EENGINE_REDIS=redis://localhost:6379/8"
Environment="EENGINE_SECRET=your-secret-key-at-least-32-characters"

StandardOutput=journal
StandardError=journal
SyslogIdentifier=emailengine

[Install]
WantedBy=multi-user.target
```

**How it works:**

1. **MemoryMax:** When memory exceeds this value, SystemD sends SIGKILL to terminate the service
2. **Restart=always:** Ensures the service restarts after any termination, including OOM kills
3. **StartLimitBurst/Interval:** Prevents rapid restart loops if there are severe issues

**Verify memory-based restart is configured:**

```bash
# Check current memory limit
sudo systemctl show emailengine -p MemoryMax

# Monitor memory usage
sudo systemctl status emailengine | grep Memory

# View OOM kill events in logs
sudo journalctl -u emailengine | grep -i "killed\|oom"

# Check restart count
sudo systemctl show emailengine -p NRestarts
```

**Example log output after memory-based restart:**

```
emailengine.service: A process of this unit has been killed by the OOM killer.
emailengine.service: Main process exited, code=killed, status=9/KILL
emailengine.service: Scheduled restart job, restart counter is at 1.
Started EmailEngine Email API Service.
```

:::tip Choosing Memory Limits
Memory usage varies significantly based on the number of accounts, message volume, and specific usage patterns. Monitor your actual memory usage over time with `systemctl status emailengine` and set `MemoryMax` appropriately for your workload. Leave enough headroom below your server's total RAM for the operating system and other services.
:::

:::warning Detecting Memory Exhaustion
If EmailEngine's CPU usage reaches 100% and stays there persistently, this often indicates the process has exhausted available memory. When this happens, the system starts thrashing (excessive swapping), causing high CPU load while the process becomes unresponsive. Setting `MemoryMax` prevents this scenario by terminating and restarting the service before it can consume all system memory.
:::

:::warning Restart Limits
The `StartLimitBurst` and `StartLimitInterval` settings prevent infinite restart loops. If EmailEngine restarts more than 5 times within 5 minutes, SystemD stops trying. Check logs to diagnose the underlying issue.
:::

## Advanced Configuration

### Multiple Instances

Run multiple EmailEngine instances on different ports:

**Instance 1:** `/etc/systemd/system/emailengine@3001.service`

```ini
[Unit]
Description=EmailEngine Instance on port %i
After=network.target redis.service

[Service]
Type=simple
User=emailengine
ExecStart=/usr/bin/emailengine --api.port=%i
Restart=always

[Install]
WantedBy=multi-user.target
```

**Start instances:**
```bash
sudo systemctl start emailengine@3001
sudo systemctl start emailengine@3002
sudo systemctl enable emailengine@3001
sudo systemctl enable emailengine@3002
```

### Dependency Management

**Start after Redis:**

```ini
[Unit]
After=redis.service
Requires=redis.service
```

**Wait for network:**

```ini
[Unit]
After=network-online.target
Wants=network-online.target
```

**Start after file system:**

```ini
[Unit]
After=local-fs.target
RequiresMountsFor=/var/log/emailengine
```

### Graceful Shutdown

**Configure timeout:**

```ini
[Service]
TimeoutStopSec=30      # Wait 30s before SIGKILL
KillMode=mixed         # SIGTERM to main, then SIGKILL
```

**Pre-stop script:**

```ini
[Service]
ExecStop=/usr/local/bin/emailengine-stop.sh
```

**Create `/usr/local/bin/emailengine-stop.sh`:**

```bash
#!/bin/bash
# Notify monitoring system
curl -X POST https://monitor.example.com/emailengine/stopping

# Allow connections to drain
sleep 5
```

## Monitoring and Health Checks

### SystemD Health Checks

**Configure watchdog:**

```ini
[Service]
WatchdogSec=60
Restart=on-watchdog
```

**Check health endpoint:**

Create `/usr/local/bin/emailengine-health.sh`:

```bash
#!/bin/bash
response=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/health)
if [ "$response" = "200" ]; then
    exit 0
else
    exit 1
fi
```

**Add to service:**

```ini
[Service]
ExecStartPost=/usr/local/bin/emailengine-health.sh
```

### Monitoring Commands

```bash
# CPU and memory usage
sudo systemctl status emailengine | grep -E 'CPU|Memory'

# Detailed resource usage
sudo systemd-cgtop

# Service failures
sudo systemctl list-units --failed

# Restart count
sudo systemctl show emailengine -p NRestarts
```

### Integration with Monitoring Tools

**Prometheus metrics:**

Metrics are available at `/metrics` endpoint on the main API port. Create a token with metrics scope:

```bash
emailengine tokens issue -d "Prometheus" -s "metrics"
```

Access metrics:
```bash
curl -H "Authorization: Bearer YOUR_TOKEN" http://localhost:3000/metrics
```

**Export logs to external system:**

```bash
# Forward to syslog
sudo journalctl -u emailengine -f | logger -t emailengine

# Forward to Logstash
sudo journalctl -u emailengine -f -o json | \
  netcat logstash.example.com 5000
```

## Updates and Maintenance

### Update EmailEngine

```bash
# Stop service
sudo systemctl stop emailengine

# Download new version
wget https://go.emailengine.app/emailengine.tar.gz
tar xzf emailengine.tar.gz

# Replace binary
sudo mv emailengine /usr/local/bin/
sudo chmod +x /usr/local/bin/emailengine

# Or download specific version (e.g., v2.48.5)
# wget https://github.com/postalsys/emailengine/releases/download/v2.48.5/emailengine.tar.gz

# Reload systemd (if service file changed)
sudo systemctl daemon-reload

# Start service
sudo systemctl start emailengine

# Verify version
emailengine --version
```

### Backup Configuration

```bash
# Backup configuration files
sudo tar czf emailengine-config-$(date +%Y%m%d).tar.gz \
  /etc/emailengine/ \
  /etc/systemd/system/emailengine.service

# Backup Redis data
sudo cp /var/lib/redis/dump.rdb /backup/
```

### Service File Changes

```bash
# Edit service file
sudo systemctl edit --full emailengine

# Or manually edit
sudo nano /etc/systemd/system/emailengine.service

# Reload systemd
sudo systemctl daemon-reload

# Restart service
sudo systemctl restart emailengine
```

## Complete Example

### Production Setup

**1. Install dependencies:**
```bash
sudo apt update
sudo apt install -y redis-server

# Download and install EmailEngine
wget https://go.emailengine.app/emailengine.tar.gz
tar xzf emailengine.tar.gz
sudo mv emailengine /usr/local/bin/
sudo chmod +x /usr/local/bin/emailengine
```

**2. Create user and directories:**
```bash
sudo useradd --system --home /opt/emailengine --shell /bin/false emailengine
sudo mkdir -p /etc/emailengine /var/log/emailengine
sudo chown emailengine:emailengine /var/log/emailengine
```

**3. Generate secret:**
```bash
# Generate a random secret (minimum 32 characters) and save it
openssl rand -hex 32
```

**Save this value securely!** You'll use it in the service file below.

**4. Create service file (use the secret from step 3):**
```bash
sudo tee /etc/systemd/system/emailengine.service > /dev/null <<'EOF'
[Unit]
Description=EmailEngine Email API Service
After=network.target redis.service
Requires=redis.service

[Service]
Type=simple
User=emailengine
Group=emailengine
WorkingDirectory=/opt/emailengine

# Replace with your actual secret from step 3
Environment="EENGINE_REDIS=redis://localhost:6379/8"
Environment="EENGINE_SECRET=your-generated-secret-from-step-3"
Environment="EENGINE_WORKERS=4"
Environment="EENGINE_LOG_LEVEL=info"

ExecStart=/usr/local/bin/emailengine
Restart=always
RestartSec=10

StandardOutput=journal
StandardError=journal
SyslogIdentifier=emailengine

NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/log/emailengine

LimitNOFILE=65536
MemoryLimit=2G

[Install]
WantedBy=multi-user.target
EOF
```

**5. Enable and start:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable emailengine
sudo systemctl start emailengine
sudo systemctl status emailengine
```

**6. Verify:**
```bash
curl http://localhost:3000/health
```

## See Also

- [Linux installation](/docs/installation/linux) - Getting the binary in place first
- [Source installation](/docs/installation/source) - The unit file for a source deployment
- [Logging](/docs/advanced/logging) - Reading what the unit writes to the journal
- [Security](/docs/deployment/security) - Hardening directives worth adding to the unit
