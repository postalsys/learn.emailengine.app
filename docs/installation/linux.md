---
title: Linux Installation
description: Install EmailEngine on Linux using binary, automated installer, or from source
sidebar_position: 2
---

# Installing EmailEngine on Linux

Three ways to run EmailEngine on Linux: the automated installer for a fresh Ubuntu or Debian server, the standalone binary on any distribution, or from source.

## Overview

EmailEngine can be installed on Linux using three methods:

1. **Automated Installer** (Ubuntu/Debian) - A script for fresh, public-facing servers
2. **Binary Installation** - Standalone x86_64 executable (any distribution)
3. **Source Installation** - Run from source for production (requires Node.js 20+, recommended 24+)

### System Requirements

The figures on the [installation overview](/docs/installation#system-requirements) apply: 2 GB of memory to evaluate, 4 to 8 GB for production.

### Required Software

- **Redis** - a stand-alone instance with `maxmemory-policy noeviction` and persistence enabled; Redis Cluster and ElastiCache are not supported. The EmailEngine README states that any Redis version works
- **Node.js 20+** (only for source installation, recommended 24+)
- **OpenSSL** (for generating secrets)

### Privileges

EmailEngine does not require root or administrator privileges to run. You can run it as any unprivileged user (e.g., a dedicated `emailengine` user) on any unprivileged port (e.g., 3000).

Root access is only needed during initial setup to:
- Create the SystemD service file
- Create a dedicated system user
- Bind the SMTP or IMAP proxy to privileged ports (below 1024, such as 465 or 993)

Once installed, EmailEngine runs as an unprivileged user. For privileged ports, instead of running as root, consider these safer alternatives:
- Use a reverse proxy (Nginx, Caddy) to forward traffic
- Use `setcap` to grant port binding capabilities: `sudo setcap 'cap_net_bind_service=+ep' /path/to/emailengine`
- Use iptables/nftables to redirect ports

## Method 1: Automated Installer (Ubuntu/Debian)

A script that installs and wires up everything a single-server deployment needs. It does not check which distribution it is running on; what it needs is `apt-get`, systemd, a `redis-server` package that reads `/etc/redis/redis.conf`, and Caddy's Debian package repository, which is what Ubuntu and Debian provide. It has to run as root and, for a new installation, on a server reachable from the public internet, because Caddy provisions the TLS certificate during the run.

### What It Installs

- EmailEngine binary at `/opt/emailengine`, running as the `emailengine` system user
- Redis server from the distribution's package, with a generated password, `maxmemory-policy noeviction`, and RDB snapshots appended to `/etc/redis/redis.conf`
- Caddy reverse proxy with automatic HTTPS, configured in `/etc/caddy/Caddyfile` with a redirect from port 80, a 100 MB request body limit, and a set of security response headers (HSTS among them)
- SystemD service at `/etc/systemd/system/emailengine.service`, with `EENGINE_REDIS` (database 8, with the password), `EENGINE_SECRET`, `EENGINE_PORT=3000`, `EENGINE_API_PROXY=true`, `EENGINE_WORKERS=8`, and `EENGINE_LOG_LEVEL=info` set in the unit. It also sets `EENGINE_INSTALL_SCRIPT=true`, which makes the admin **Upgrade** page show the instructions for this layout
- Upgrade helper script at `/opt/upgrade-emailengine.sh`

:::note This layout differs from the manual install below
The installer keeps the binary at `/opt/emailengine`. The manual instructions further down place it at `/usr/local/bin/emailengine` instead. Pick one approach and stay with it, since the upgrade helper only knows about the installer's layout.
:::

### Features

- Supports both fresh installations and upgrades
- Automatically detects existing installations
- Preserves Redis configuration during upgrades
- Can install specific versions
- Generates secure credentials

### Installation Steps

#### 1. Download Installer

```bash
wget https://go.emailengine.app -O install.sh
# or
curl -L https://go.emailengine.app -o install.sh
```

#### 2. Run Installer

```bash
chmod +x install.sh
sudo ./install.sh example.com
```

Replace `example.com` with the hostname EmailEngine will be served on. If you omit it, the script asks for one, and leaving that prompt empty makes it request an auto-generated hostname from `api.nodemailer.com`, which is enough to try the installation without DNS of your own. On an existing installation the script takes the hostname from the current Caddyfile instead of asking.

**Install specific version:**
```bash
sudo ./install.sh example.com 2.79.4
```

The version is accepted with or without a leading `v`.

#### 3. Wait for Completion

The script will:
1. Install dependencies (Redis, Caddy, `curl`, `wget`, `openssl`)
2. Generate a Redis password and the `EENGINE_SECRET`
3. Configure and start Redis
4. Download the EmailEngine binary to `/opt/emailengine` and create the `emailengine` user
5. Create and start the SystemD service
6. Write the Caddyfile and reload Caddy, then wait up to 60 seconds for `https://example.com/` to answer
7. Write the credentials file

#### 4. Access EmailEngine

Once complete, open `https://example.com` to create your admin account.

**Credentials saved to:** `/root/emailengine-credentials.txt`

### Important Notes

- **Fresh servers only** for new installations: the script overwrites `/etc/caddy/Caddyfile` and appends to `/etc/redis/redis.conf`
- **Supports upgrades** on existing installations: when `emailengine.service` is already active it reads the existing Redis password and secret from the unit, skips the Redis and unit configuration, and only replaces the binary
- **Public-facing server required** for a new installation, because Caddy provisions the TLS certificate during the run

### Upgrading

For servers installed with the automated installer:

```bash
# Upgrade to latest
sudo /opt/upgrade-emailengine.sh

# Or re-run installer for specific version
sudo ./install.sh example.com 2.79.4
```

`/opt/upgrade-emailengine.sh` downloads the latest release, compares its version with the installed binary, and either reports that nothing changed or swaps the binary and restarts the service. Re-running `install.sh` on an existing installation shows the current and target versions, asks for confirmation, keeps the Redis configuration and the unit file, and replaces the binary.

## Method 2: Binary Installation

Manual installation using the standalone binary.

### Step 1: Install Redis

**Ubuntu/Debian:**
```bash
sudo apt update
sudo apt install redis-server

# Start and enable Redis
sudo systemctl start redis-server
sudo systemctl enable redis-server

# Verify
redis-cli ping  # Should return: PONG
```

**CentOS/RHEL:**
```bash
sudo yum install redis
sudo systemctl start redis
sudo systemctl enable redis
```

### Step 2: Configure Redis

Edit `/etc/redis/redis.conf`:

```bash
sudo nano /etc/redis/redis.conf
```

Add or modify:
```
# Production settings
maxmemory-policy noeviction

# Persistence
save 900 1
save 300 10
save 60 10000
```

Restart Redis:
```bash
sudo systemctl restart redis
```

### Step 3: Download EmailEngine

```bash
# Download latest binary
wget https://go.emailengine.app/emailengine.tar.gz

# Or download specific version (e.g., 2.79.4)
wget https://go.emailengine.app/download/v2.79.4/emailengine.tar.gz

# Extract
tar xzf emailengine.tar.gz
rm emailengine.tar.gz

# Install to system path
sudo mv emailengine /usr/local/bin/
sudo chmod +x /usr/local/bin/emailengine

# Verify
emailengine --version
```

The archive contains one file, a self-contained executable built for x86_64 with a bundled Node.js 24 runtime. There is no ARM build of the Linux binary; on ARM servers use the [Docker image](/docs/installation/docker), which has an `arm64` manifest, or run from [source](/docs/installation/source).

### Step 4: Create Configuration

**Generate and save encryption secret:**
```bash
# Generate a random secret (minimum 32 characters) and save to .env file
mkdir -p /etc/emailengine
echo "EENGINE_SECRET=$(openssl rand -hex 32)" > /etc/emailengine/.env
echo "EENGINE_REDIS=redis://127.0.0.1:6379/8" >> /etc/emailengine/.env

# Secure the file
chmod 600 /etc/emailengine/.env
```

**Important:** Save this file permanently. You must use the same secret every time EmailEngine starts.

### Step 5: Test Run

```bash
# Load environment variables
source /etc/emailengine/.env

# Start EmailEngine
emailengine \
  --dbs.redis="$EENGINE_REDIS" \
  --service.secret="$EENGINE_SECRET" \
  --api.port=3000

# In another terminal, test
curl http://localhost:3000/health
# Should return: {"success":true}
```

### Step 6: Run as Service

See [SystemD Service Guide](/docs/deployment/systemd) for production setup.

**Quick SystemD setup:**

```bash
# Create service file
sudo nano /etc/systemd/system/emailengine.service
```

```ini
[Unit]
Description=EmailEngine
# Debian and Ubuntu name the unit redis-server.service; adjust both lines to match
After=redis.service
Requires=redis.service

[Service]
Type=simple
User=emailengine
Group=emailengine
WorkingDirectory=/opt/emailengine

Environment="EENGINE_REDIS=redis://127.0.0.1:6379/8"
Environment="EENGINE_SECRET=your-secret-here"
Environment="EENGINE_WORKERS=4"

ExecStart=/usr/local/bin/emailengine
Restart=always
RestartSec=10

StandardOutput=journal
StandardError=journal
SyslogIdentifier=emailengine

LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```bash
# Create user
sudo useradd --system --home /opt/emailengine --shell /bin/false emailengine

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable emailengine
sudo systemctl start emailengine
sudo systemctl status emailengine
```

### Upgrading Binary Installation

```bash
# Download latest
wget https://go.emailengine.app/emailengine.tar.gz

# Or download specific version (e.g., 2.79.4)
wget https://go.emailengine.app/download/v2.79.4/emailengine.tar.gz

# Extract and replace
tar xzf emailengine.tar.gz
sudo systemctl stop emailengine
sudo mv emailengine /usr/local/bin/
sudo chmod +x /usr/local/bin/emailengine

# Restart
sudo systemctl start emailengine

# Verify
emailengine --version
```

## Method 3: Source Installation (Production)

Running from source is recommended for production as it uses less memory than the binary. The binary uses a virtual filesystem that loads all files into memory at startup, while source installation keeps files on disk.

For complete source installation instructions, including Node.js setup, SystemD service configuration, and upgrade procedures, see the dedicated **[Source Installation Guide](/docs/installation/source)**.

## Post-Installation

### 1. Set Up Reverse Proxy with HTTPS

For production, use Nginx or Caddy as a reverse proxy with automatic HTTPS. In the configurations below, `emailengine.example.com` is the public hostname and `localhost:3000` is EmailEngine itself, listening on the same machine.

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

<Tabs>
<TabItem value="nginx" label="Nginx + acme.sh" default>

#### Install Nginx

```bash
sudo apt install nginx
```

#### Create Nginx Configuration

Create `/etc/nginx/sites-available/emailengine`:

```nginx
server {
    listen 80;
    server_name emailengine.example.com;

    # Allow large email submissions with attachments
    client_max_body_size 100M;
    client_body_timeout 90s;

    # EventSource endpoint for admin UI updates
    location ~ ^/(admin|v1)/changes {
        proxy_pass http://localhost:3000;

        # Disable gzip for EventSource streaming
        gzip off;

        # HTTP/1.1 required for EventSource
        proxy_http_version 1.1;
        proxy_set_header Connection '';

        # Disable buffering for real-time updates
        proxy_buffering off;
        proxy_cache off;

        # Keep connection alive for long-polling
        proxy_read_timeout 24h;

        # Disable chunked encoding
        chunked_transfer_encoding off;

        # Standard proxy headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # All other requests
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts for large uploads
        proxy_connect_timeout 90s;
        proxy_send_timeout 90s;
        proxy_read_timeout 90s;
    }
}
```

Enable the configuration:

```bash
sudo ln -s /etc/nginx/sites-available/emailengine /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

#### Install and Configure acme.sh

```bash
# Install acme.sh
curl https://get.acme.sh | sh -s email=admin@example.com

# Reload shell to enable acme.sh
source ~/.bashrc

# Set Let's Encrypt as the default CA (instead of ZeroSSL)
~/.acme.sh/acme.sh --set-default-ca --server letsencrypt

# Create SSL directory
sudo mkdir -p /etc/nginx/ssl

# Issue certificate (Nginx mode)
sudo ~/.acme.sh/acme.sh --issue -d emailengine.example.com --nginx

# Install certificate to Nginx
sudo ~/.acme.sh/acme.sh --install-cert -d emailengine.example.com \
  --key-file /etc/nginx/ssl/emailengine.key \
  --fullchain-file /etc/nginx/ssl/emailengine.crt \
  --reloadcmd "systemctl reload nginx"
```

#### Update Nginx for HTTPS

Edit `/etc/nginx/sites-available/emailengine` to add SSL configuration:

```nginx
server {
    listen 80;
    server_name emailengine.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name emailengine.example.com;

    # SSL certificates
    ssl_certificate /etc/nginx/ssl/emailengine.crt;
    ssl_certificate_key /etc/nginx/ssl/emailengine.key;

    # SSL security settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # Allow large email submissions with attachments
    client_max_body_size 100M;
    client_body_timeout 90s;

    # EventSource endpoint for admin UI updates
    location ~ ^/(admin|v1)/changes {
        proxy_pass http://localhost:3000;

        # Disable gzip for EventSource streaming
        gzip off;

        # HTTP/1.1 required for EventSource
        proxy_http_version 1.1;
        proxy_set_header Connection '';

        # Disable buffering for real-time updates
        proxy_buffering off;
        proxy_cache off;

        # Keep connection alive for long-polling
        proxy_read_timeout 24h;

        # Disable chunked encoding
        chunked_transfer_encoding off;

        # Standard proxy headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # All other requests
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts for large uploads
        proxy_connect_timeout 90s;
        proxy_send_timeout 90s;
        proxy_read_timeout 90s;
    }
}
```

Reload Nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

</TabItem>
<TabItem value="caddy" label="Caddy (Automatic HTTPS)">

#### Install Caddy

```bash
# Add Caddy repository
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

#### Configure Caddy

Create `/etc/caddy/Caddyfile`:

```caddy
emailengine.example.com {
    # Automatic HTTPS via Let's Encrypt

    # Allow large email submissions with attachments
    request_body {
        max_size 100MB
    }

    # EventSource endpoints: the admin dashboard feed and the API change stream
    @eventsource path /admin/changes /v1/changes
    handle @eventsource {
        reverse_proxy localhost:3000 {
            # Disable buffering for EventSource streaming
            flush_interval -1

            # Long timeout for EventSource
            transport http {
                read_timeout 24h
            }
        }
    }

    # All other requests
    reverse_proxy localhost:3000 {
        # Standard headers
        header_up Host {host}
        header_up X-Real-IP {remote}
        header_up X-Forwarded-For {remote}
        header_up X-Forwarded-Proto {scheme}

        # Timeouts for large uploads
        transport http {
            dial_timeout 90s
            response_header_timeout 90s
            read_timeout 90s
        }
    }
}
```

#### Start Caddy

```bash
sudo systemctl enable caddy
sudo systemctl start caddy
sudo systemctl status caddy
```

Caddy will automatically obtain and renew SSL certificates from Let's Encrypt.

</TabItem>
</Tabs>

### 2. Configure Firewall

```bash
# Allow HTTP/HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Block direct access to EmailEngine
sudo ufw deny 3000/tcp

# Enable firewall
sudo ufw enable
```

### 3. Verify Installation

```bash
# Check service
sudo systemctl status emailengine

# Check logs
sudo journalctl -u emailengine -f

# Test health endpoint
curl http://localhost:3000/health

# Check Redis
redis-cli ping
```

## Performance Tuning

### Optimize Redis

```bash
sudo nano /etc/redis/redis.conf
```

```
# Memory
maxmemory-policy noeviction

# Persistence
save 900 1
save 300 10
save 60 10000

# Performance
tcp-backlog 511
timeout 300
tcp-keepalive 300

# Limits
maxclients 10000
```

### Optimize EmailEngine

```bash
# IMAP worker threads (default 4); see the performance tuning page for sizing
EENGINE_WORKERS=8

# Increase file descriptor limit
# In service file:
LimitNOFILE=65536
```

### Monitor Performance

```bash
# System resources
sudo systemctl status emailengine | grep -E 'CPU|Memory'

# Redis stats
redis-cli INFO stats

# Prometheus metrics (available on API port with metrics token)
curl http://localhost:3000/metrics -H "Authorization: Bearer YOUR_METRICS_TOKEN"
```

## See Also

- [SystemD service](/docs/deployment/systemd) - The production service unit in full
- [Source installation](/docs/installation/source) - Running from source instead of the binary
- [Nginx reverse proxy](/docs/deployment/nginx-proxy) - The proxy configuration on its own page
- [Redis configuration](/docs/configuration/redis) - Persistence, memory policy, and connection URLs
- [Security](/docs/deployment/security) - Hardening a production deployment
