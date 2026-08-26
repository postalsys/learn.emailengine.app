---
title: Source Installation
description: Run EmailEngine from source for production deployments
sidebar_position: 6
---

# Installing EmailEngine from Source

Running EmailEngine from the source distribution, which is what we recommend in production because it holds less memory than the packaged binary.

## Why Install from Source?

Running EmailEngine from source provides several advantages over binary distributions:

### Production Benefits

**Lower Memory Usage:**
The pre-built binary is a `pkg` bundle: the application files sit in a snapshot inside the executable and are served from memory. When running from source, files stay on disk and are read as Node.js loads them, so the base memory footprint is lower.

**Your Own Runtime:**
The binary bundles a fixed Node.js version (24 for the current releases). From source you pick the Node.js version, apply patches if you need to, and can read the code that is running.

**Production Recommendation:**
For production environments, especially those managing many email accounts or running on servers with limited memory, source installation is the recommended approach.

### System Requirements

The figures on the [installation overview](/docs/installation#system-requirements) apply: 2 GB of memory to evaluate, 4 to 8 GB for production.

### Required Software

- **Node.js 20+** (the `engines` field in EmailEngine's package.json is `>=20.x`; the release builds and CI use 24)
- **Redis** - a stand-alone instance with `maxmemory-policy noeviction` and persistence enabled
- **wget/curl** (for downloading release tarballs)

### Privileges

EmailEngine does not require root or administrator privileges to run. You can run it as any unprivileged user (e.g., a dedicated `emailengine` user) on any unprivileged port (e.g., 3000).

Root access is only needed during initial setup to:
- Create directories in `/opt` and set ownership
- Create the SystemD service file or Launch Agent
- Create a dedicated system user
- Bind the SMTP or IMAP proxy to privileged ports (below 1024, such as 465 or 993)

Once installed, EmailEngine runs as an unprivileged user. For privileged ports, instead of running as root, consider these safer alternatives:
- Use a reverse proxy (Nginx, Caddy) to forward traffic
- Use `setcap` to grant port binding capabilities: `sudo setcap 'cap_net_bind_service=+ep' $(which node)`
- Use iptables/nftables to redirect ports

## Installation Methods

Choose between stable releases or development versions:

1. **Release Tarball** (recommended for production) - Stable, tested releases
2. **Git Repository** (for development) - Latest features, may be unstable

## Method 1: Release Tarball (Recommended)

### Linux Installation

#### Prerequisites

Before installing EmailEngine from source, you need Node.js 20+ (24+ recommended) and Redis:

- **Node.js & Redis setup:** Follow the [Linux Installation Guide](/docs/installation/linux) for Redis setup, and install Node.js 20+ (24+ recommended)
- Return to this guide after completing the prerequisites

#### Step 1: Setup Directory Structure

```bash
# Create directories
sudo mkdir -p /opt/emailengine/app
cd /opt/emailengine
```

#### Step 2: Configure Environment

Create and populate `.env` file in `/opt/emailengine`:

```bash
# Generate encryption secret and create .env file
sudo bash -c "cat > /opt/emailengine/.env" <<EOF
# Redis connection (database 8 is the default)
EENGINE_REDIS=redis://127.0.0.1:6379/8

# Security secret (auto-generated)
EENGINE_SECRET=$(openssl rand -hex 32)

# Performance
EENGINE_WORKERS=4

# Logging
EENGINE_LOG_LEVEL=info

# API settings
EENGINE_PORT=3000
EENGINE_HOST=0.0.0.0
EOF

# Secure the file
sudo chmod 600 /opt/emailengine/.env
```

**Important:** The `.env` file is stored outside the `app/` directory, making upgrades easier. Keep this file safe.

#### Step 3: Download and Install EmailEngine

```bash
cd /opt/emailengine

# Download latest source distribution
sudo wget https://go.emailengine.app/source-dist.tar.gz

# Or download specific version (e.g., 2.79.4)
sudo wget https://go.emailengine.app/download/v2.79.4/source-dist.tar.gz

# Extract to app directory (includes node_modules)
sudo tar xzf source-dist.tar.gz -C app --strip-components=1
sudo rm source-dist.tar.gz
```

:::tip No npm install needed
The source-dist.tar.gz includes a complete `node_modules` folder with the production dependencies already installed (`npm ci --omit=dev` at build time), so there is nothing to run before starting. The archive has no top-level directory of its own; its entries start with `./`, which is what `--strip-components=1` removes.
:::

#### Step 4: Test Installation

```bash
cd /opt/emailengine
node app/server.js
```

In another terminal:
```bash
curl http://localhost:3000/health
# Should return: {"success":true}
```

Press `Ctrl+C` to stop.

#### Step 5: Create System User

```bash
# Create dedicated user
sudo useradd --system --home /opt/emailengine --shell /bin/false emailengine

# Set ownership
sudo chown -R emailengine:emailengine /opt/emailengine
sudo chmod 600 /opt/emailengine/.env
```

#### Step 6: Set Up SystemD Service

Create service file:
```bash
sudo nano /etc/systemd/system/emailengine.service
```

Add configuration:
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
WorkingDirectory=/opt/emailengine

# Load environment variables
EnvironmentFile=/opt/emailengine/.env

# Start command
ExecStart=/usr/bin/node app/server.js

# Restart policy
Restart=always
RestartSec=10

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=emailengine

# Resource limits
LimitNOFILE=65536

# Security hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/opt/emailengine

[Install]
WantedBy=multi-user.target
```

Enable and start service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable emailengine
sudo systemctl start emailengine
sudo systemctl status emailengine
```

View logs:
```bash
sudo journalctl -u emailengine -f
```

### macOS Installation

#### Prerequisites

Before installing EmailEngine from source, you need Node.js 20+ (24+ recommended) and Redis:

- **Node.js & Redis setup:** Follow the [macOS Installation Guide](/docs/installation/macos) for Redis setup, and install Node.js 20+ (24+ recommended)
- Return to this guide after completing the prerequisites

#### Step 1: Download and Install EmailEngine

```bash
# Create directory structure
sudo mkdir -p /opt/emailengine/app
cd /opt/emailengine

# Download source (includes node_modules)
sudo curl -L https://go.emailengine.app/source-dist.tar.gz | sudo tar xz -C app --strip-components=1
```

#### Step 2: Configure Environment

Create `/opt/emailengine/.env`:

```bash
# Generate encryption secret and create .env file
sudo bash -c "cat > /opt/emailengine/.env" <<EOF
EENGINE_REDIS=redis://127.0.0.1:6379/8
EENGINE_SECRET=$(openssl rand -hex 32)
EENGINE_WORKERS=4
EENGINE_PORT=3000
EENGINE_HOST=0.0.0.0
EENGINE_LOG_LEVEL=info
EOF

# Secure the file
sudo chmod 600 /opt/emailengine/.env
```

#### Step 3: Run as Launch Agent

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
        <string>/usr/local/bin/node</string>
        <string>/opt/emailengine/app/server.js</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/opt/emailengine</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>NODE_ENV</key>
        <string>production</string>
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

On Apple Silicon, Homebrew installs Node under `/opt/homebrew/bin/node` and `/usr/local/var/log` may not exist. Run `which node` and adjust both the interpreter path and the log paths before loading the agent.

Load and start:
```bash
launchctl load ~/Library/LaunchAgents/com.emailengine.plist
```

### Windows Installation

For Windows, use **WSL2** (Windows Subsystem for Linux) and follow the Linux installation steps above.

See [Windows Installation Guide](/docs/installation/windows) for WSL2 setup.

## Alternative Process Managers

### PM2 (Production Process Manager)

PM2 is an excellent alternative to SystemD for process management.

#### Install PM2

```bash
sudo npm install -g pm2
```

#### Create PM2 Ecosystem File

Create `/opt/emailengine/ecosystem.config.js`:

```javascript
module.exports = {
  apps: [{
    name: 'emailengine',
    script: './app/server.js',
    cwd: '/opt/emailengine',
    instances: 1,
    exec_mode: 'fork',
    watch: false,
    env: {
      NODE_ENV: 'production'
    },
    error_file: '/var/log/emailengine/error.log',
    out_file: '/var/log/emailengine/out.log',
    log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
    merge_logs: true,
    autorestart: true,
    max_restarts: 10,
    min_uptime: '10s'
  }]
};
```

**Note:** EmailEngine reads a `.env` file from its working directory on startup, so `cwd: '/opt/emailengine'` is what makes `/opt/emailengine/.env` apply. PM2 runs one process here rather than a cluster: EmailEngine starts its own worker threads, and a second copy of the process would compete with the first for the same Redis state.

#### Start with PM2

```bash
# Start EmailEngine
pm2 start ecosystem.config.js

# Save PM2 configuration
pm2 save

# Setup PM2 to start on boot
pm2 startup

# View logs
pm2 logs emailengine

# Monitor
pm2 monit

# Restart
pm2 restart emailengine

# Stop
pm2 stop emailengine
```

### Docker with Source

If you want to build your own Docker image from source, the EmailEngine repository includes the `Dockerfile` the published images are built from. Clone the repository (the build copies `.git/refs/heads/master` to stamp the commit, so a source tarball without `.git` does not build) and run:

```bash
# Clone the repository
git clone https://github.com/postalsys/emailengine.git
cd emailengine

# Build the Docker image
docker build -t emailengine:custom .

# Run with Redis
docker run -d \
  --name emailengine \
  -p 3000:3000 \
  -e EENGINE_REDIS="redis://redis-host:6379" \
  -e EENGINE_SECRET="your-secret-key" \
  emailengine:custom
```

The Dockerfile installs dependencies with `npm ci --omit=dev`, runs as a non-root user, uses `dumb-init` as the entrypoint for signal handling, and presets `EENGINE_HOST=0.0.0.0` and `EENGINE_API_PROXY=true`.

## Upgrading

The directory structure keeps an upgrade to replacing the `app/` directory; the `.env` file next to it is untouched.

```bash
cd /opt/emailengine

# Stop service
sudo systemctl stop emailengine

# Backup current version (optional)
sudo mv app app.backup.$(date +%Y%m%d)

# Create new app directory
sudo mkdir -p app

# Download new version (latest)
sudo wget https://go.emailengine.app/source-dist.tar.gz

# Or download specific version (e.g., 2.79.4)
sudo wget https://go.emailengine.app/download/v2.79.4/source-dist.tar.gz

# Extract to app directory
sudo tar xzf source-dist.tar.gz -C app --strip-components=1
sudo rm source-dist.tar.gz

# Restore ownership
sudo chown -R emailengine:emailengine /opt/emailengine

# Start service
sudo systemctl start emailengine

# Verify
sudo systemctl status emailengine
curl http://localhost:3000/health
```

**Note:** Your `.env` file in `/opt/emailengine/` is preserved during upgrades.

### With PM2

The PM2 upgrade process is the same as with SystemD: replace the `app/` directory and reload PM2.

```bash
cd /opt/emailengine

# Backup current version (optional)
sudo mv app app.backup.$(date +%Y%m%d)

# Create new app directory
sudo mkdir -p app

# Download new version (latest)
sudo wget https://go.emailengine.app/source-dist.tar.gz

# Or download specific version (e.g., 2.79.4)
sudo wget https://go.emailengine.app/download/v2.79.4/source-dist.tar.gz

# Extract to app directory
sudo tar xzf source-dist.tar.gz -C app --strip-components=1
sudo rm source-dist.tar.gz

# Reload with zero-downtime
pm2 reload emailengine
```

## Configuration Options

### Environment Variables

All configuration can be set via environment variables in `.env` file:

```bash
# Core settings
EENGINE_REDIS=redis://127.0.0.1:6379/8
EENGINE_SECRET=your-encryption-secret-at-least-32-chars

# Performance
EENGINE_WORKERS=4

# API
EENGINE_PORT=3000
EENGINE_HOST=0.0.0.0

# Logging
EENGINE_LOG_LEVEL=info
EENGINE_LOG_RAW=false

# Features (5MB = 5 * 1024 * 1024)
EENGINE_MAX_SIZE=5242880
```

See [Configuration Options](/docs/configuration) for complete reference.

## Performance Optimization

### Node.js Optimization

```bash
# Set Node.js options in SystemD service
Environment="NODE_OPTIONS=--max-old-space-size=2048"

# Or in ecosystem.config.js for PM2
node_args: '--max-old-space-size=2048'
```

### Worker Configuration

```bash
# Set workers to CPU core count
EENGINE_WORKERS=4

# For high-load servers
EENGINE_WORKERS=8
```

### Redis Optimization

See [Linux Installation - Performance Tuning](/docs/installation/linux#performance-tuning) for Redis optimization.

## Monitoring

### Health Checks

```bash
# HTTP health endpoint
curl http://localhost:3000/health

# Returns: {"success":true}
```

### Prometheus Metrics

Create a token with the `metrics` scope. A source installation has no `emailengine` command on the path; the CLI is `app/bin/emailengine.js`, and it reads the same `.env` from the working directory as the server does:
```bash
cd /opt/emailengine
node app/bin/emailengine.js tokens issue -d "Prometheus" -s "metrics"
```

Access metrics at `/metrics` endpoint:
```bash
curl -H "Authorization: Bearer YOUR_TOKEN" http://localhost:3000/metrics
```

### Logs

```bash
# SystemD logs
sudo journalctl -u emailengine -f

# PM2 logs
pm2 logs emailengine

# Log files
tail -f /var/log/emailengine/*.log
```

## Security Best Practices

1. **Run as dedicated user** (not root)
2. **Secure `.env` file** with `chmod 600`
3. **Use strong secrets** (32+ characters)
4. **Enable firewall** and restrict port access
5. **Use reverse proxy** with TLS/HTTPS
6. **Keep Node.js updated** to latest LTS version
7. **Regular backups** of Redis data
8. **Monitor logs** for suspicious activity

See [Security Best Practices](/docs/deployment/security) for detailed guidance.

## See Also

- [Linux installation](/docs/installation/linux) - Redis setup and the binary alternative
- [SystemD service](/docs/deployment/systemd) - The service unit and hardening options in full
- [Configuration](/docs/configuration) - Every environment variable and prepared settings
- [Performance tuning](/docs/advanced/performance-tuning) - Worker counts and connection limits
- [Monitoring](/docs/advanced/monitoring) - Prometheus metrics and what to alert on
