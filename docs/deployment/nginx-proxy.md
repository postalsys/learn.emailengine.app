---
title: Nginx Reverse Proxy
description: Configure Nginx reverse proxy for EmailEngine with HTTPS and SSL termination
sidebar_position: 5
---

# Nginx Reverse Proxy Setup

Configure Nginx as a reverse proxy in front of EmailEngine to terminate HTTPS, restrict who reaches the admin interface, and rate-limit abusive clients.

:::info Why Use a Reverse Proxy
- **SSL/TLS termination** - EmailEngine itself listens on plain HTTP by default
- **Failover** - Route to a cold standby instance when the primary is down
- **Access control** - Restrict `/admin` and `/metrics` by network before EmailEngine sees the request
- **Rate limiting** - Slow down abusive clients at the edge
:::

Two things about EmailEngine shape the configuration below:

- **Two endpoints stream.** `GET /v1/changes` (and the dashboard's `/admin/changes`) is a Server-Sent Events feed, and `/mcp` streams notifications to a subscribed agent. A proxy that buffers responses holds those events back until the connection closes, so both locations run with buffering off and long read timeouts. EmailEngine does not use WebSockets, so no `Upgrade` handling is needed.
- **EmailEngine reads `X-Forwarded-For` only if told to trust it,** and only from the peers you name. See [Tell EmailEngine to trust the proxy](#tell-emailengine-to-trust-the-proxy) below.

## Quick Start

### 1. Install Nginx

**Ubuntu/Debian:**
```bash
sudo apt-get update
sudo apt-get install nginx -y
```

**CentOS/RHEL:**
```bash
sudo yum install nginx -y
```

**Verify installation:**
```bash
nginx -v
sudo systemctl status nginx
```

### 2. Create Dummy SSL Certificates

Create temporary certificates for initial setup:

```bash
sudo openssl req -subj "/CN=emailengine.example.com/O=My Company/C=US" \
  -new -newkey rsa:2048 -days 365 -nodes -x509 \
  -keyout privkey.pem -out fullchain.pem

# Set permissions
sudo chmod 0600 privkey.pem

# Move to SSL directory
sudo mv privkey.pem /etc/ssl/private/emailengine-privkey.pem
sudo mv fullchain.pem /etc/ssl/certs/emailengine-fullchain.pem
```

:::tip Why Dummy Certificates?
Nginx refuses to start a `ssl` listener without a certificate. The self-signed pair lets it start now; Step 4 replaces it with a Let's Encrypt certificate.
:::

### 3. Configure Nginx Virtual Host

Create virtual host configuration:

```bash
sudo nano /etc/nginx/sites-available/emailengine.conf
```

**Basic configuration:**

```nginx
server {
    listen 80;
    listen 443 ssl http2;

    server_name emailengine.example.com;  # Change this

    ssl_certificate_key /etc/ssl/private/emailengine-privkey.pem;
    ssl_certificate /etc/ssl/certs/emailengine-fullchain.pem;

    # Server-Sent Events feed (API and admin dashboard): no gzip, no buffering
    location ~ ^/(admin|v1)/changes {
        gzip off;
        proxy_http_version 1.1;
        proxy_set_header Connection '';
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header Host $http_host;
        proxy_pass http://127.0.0.1:3000;
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 24h;
        chunked_transfer_encoding off;
    }

    # MCP endpoint - a subscription request answers with an event stream,
    # so this location must not be buffered either
    location = /mcp {
        gzip off;
        proxy_http_version 1.1;
        proxy_set_header Connection '';
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header Host $http_host;
        proxy_pass http://127.0.0.1:3000;
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 1h;
        chunked_transfer_encoding off;
    }

    location / {
        client_max_body_size 50M;  # EmailEngine accepts message uploads up to 50 MB by default (EENGINE_MAX_BODY_SIZE)
        proxy_http_version 1.1;
        proxy_redirect off;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header Host $http_host;
        proxy_pass http://127.0.0.1:3000;  # EmailEngine port
    }

    # Enforce HTTPS
    if ($scheme != "https") {
        return 301 https://$host$request_uri;
    }
}
```

### Tell EmailEngine to trust the proxy

Setting the `X-Forwarded-*` headers is half of the job. EmailEngine resolves the client address from `X-Forwarded-For` only when its **Behind Reverse Proxy** setting is on, and it decides which peers may set that header from `EENGINE_API_PROXY_ADDRESSES`.

**The setting.** Behind Reverse Proxy is a runtime setting (`enableApiProxy` in the [settings API](/docs/api/post-v-1-settings), the checkbox under **Configuration > General** in the admin interface). At the first start, when nothing is stored yet, EmailEngine seeds it from `EENGINE_API_PROXY` (or `--api.proxy`), and to `true` when that variable is not set at all. The official Docker image also sets `EENGINE_API_PROXY=true`. After the first start the stored setting is what counts; changing the environment variable later has no effect, so flip it in the admin interface or over the API instead.

**The trusted peers.** With the setting on and `EENGINE_API_PROXY_ADDRESSES` unset, the left-most `X-Forwarded-For` entry is taken as the client address, which any caller that can reach the port can forge. That is acceptable for the address shown in logs and token usage records, but not for the two controls that key off the address: the [admin interface allowlist](/docs/deployment/security#admin-interface-access-control) and per-token `restrictions.addresses`. EmailEngine logs a warning at startup in that state. Name the proxies (since EmailEngine v2.75.0), and the header is honored only from them, with the entries they appended stripped:

```bash
# Nginx on the same host
EENGINE_API_PROXY_ADDRESSES=127.0.0.1

# Several proxies, IPs or CIDR ranges, comma-separated
EENGINE_API_PROXY_ADDRESSES=10.0.0.0/8,192.168.1.10
```

The forwarded scheme and host do not set EmailEngine's public URL. That comes from the `serviceUrl` setting (Configuration > General), which the hosted authentication form, OAuth2 redirects and tracking links are built from; set it to the address Nginx serves, `https://emailengine.example.com` here. An `https:` service URL is also what makes EmailEngine mark its admin session cookie `Secure`.

**Enable configuration:**

```bash
# Create symbolic link
sudo ln -s /etc/nginx/sites-available/emailengine.conf /etc/nginx/sites-enabled/

# Test configuration
sudo nginx -t

# Should output:
# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
# nginx: configuration file /etc/nginx/nginx.conf test is successful

# Reload Nginx
sudo systemctl reload nginx
```

A reload with a broken configuration file fails and leaves the running Nginx as it was, so `nginx -t` first is what keeps a typo from turning into an outage on a full restart.

### 4. Provision SSL Certificates with Let's Encrypt

Install and configure acme.sh for automatic certificate management:

**Install acme.sh:**

```bash
sudo su
cd ~
curl https://get.acme.sh | sh -s email=your@email.com
```

**Issue certificates:**

```bash
/root/.acme.sh/acme.sh --issue --nginx --server letsencrypt \
    -d emailengine.example.com \
    --key-file       /etc/ssl/private/emailengine-privkey.pem  \
    --ca-file        /etc/ssl/certs/emailengine-chain.pem \
    --fullchain-file /etc/ssl/certs/emailengine-fullchain.pem \
    --reloadcmd     "/bin/systemctl reload nginx"
```

acme.sh installs a cron job that renews the certificate before it expires and runs the reload command afterwards.

**Verify SSL:**

```bash
# Check certificate
openssl s_client -connect emailengine.example.com:443 -servername emailengine.example.com

# Or use online tools
# https://www.ssllabs.com/ssltest/
```

## Production Configuration

### Complete Nginx Configuration

Create `/etc/nginx/sites-available/emailengine.conf`:

```nginx
# Rate limiting zone, keyed by client address
limit_req_zone $binary_remote_addr zone=emailengine_limit:10m rate=10r/s;

# Upstream definition
upstream emailengine_backend {
    server 127.0.0.1:3000 max_fails=3 fail_timeout=30s;
    # For failover (NOT scaling), add a cold standby:
    # server emailengine-standby.internal:3000 backup;
    # EmailEngine does NOT support two instances on one Redis database
}

# HTTP server - redirect to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name emailengine.example.com;

    # ACME challenge for Let's Encrypt
    location ^~ /.well-known/acme-challenge/ {
        default_type "text/plain";
        root /var/www/html;
    }

    # Redirect all other traffic to HTTPS
    location / {
        return 301 https://$server_name$request_uri;
    }
}

# HTTPS server
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name emailengine.example.com;

    # SSL certificates
    ssl_certificate /etc/ssl/certs/emailengine-fullchain.pem;
    ssl_certificate_key /etc/ssl/private/emailengine-privkey.pem;
    ssl_trusted_certificate /etc/ssl/certs/emailengine-chain.pem;

    # SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # SSL session cache
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_session_tickets off;

    # OCSP stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;

    # Security headers come from EmailEngine itself (Content-Security-Policy, X-Frame-Options,
    # X-Content-Type-Options, Referrer-Policy, Permissions-Policy, and Strict-Transport-Security
    # once the service URL setting is https), so do not add them here as well: a second copy is
    # redundant at best, and two Content-Security-Policy headers are both enforced. Only add
    # HSTS at the proxy when you want includeSubDomains or preload, and then replace
    # EmailEngine's copy rather than stacking a second one:
    # proxy_hide_header Strict-Transport-Security;
    # add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

    # Logging
    access_log /var/log/nginx/emailengine-access.log combined;
    error_log /var/log/nginx/emailengine-error.log warn;

    # Server-Sent Events feed (API and admin dashboard): no gzip, no buffering
    location ~ ^/(admin|v1)/changes {
        gzip off;
        proxy_pass http://emailengine_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection '';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 24h;
        chunked_transfer_encoding off;
    }

    # MCP endpoint (no buffering - subscription requests stream notifications)
    location = /mcp {
        gzip off;
        proxy_pass http://emailengine_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection '';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 1h;
        chunked_transfer_encoding off;
    }

    # Main location block
    location / {
        # Rate limiting (per client address; raise the rate or exempt your
        # application's address if it makes many API calls from one host)
        limit_req zone=emailengine_limit burst=20 nodelay;

        # Proxy settings
        proxy_pass http://emailengine_backend;
        proxy_http_version 1.1;
        proxy_redirect off;

        # Client body size: EmailEngine accepts message uploads up to 50 MB by default
        # (EENGINE_MAX_BODY_SIZE); other requests are capped at 1 MB
        client_max_body_size 50M;

        # Timeouts: a message fetch or a large upload can take longer than
        # the 60s Nginx default
        proxy_connect_timeout 90s;
        proxy_send_timeout 90s;
        proxy_read_timeout 90s;

        # Headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;

        # Buffering
        proxy_buffering off;
        proxy_request_buffering off;
    }

    # Health check endpoint (no rate limiting)
    location /health {
        proxy_pass http://emailengine_backend;
        access_log off;
    }

    # Metrics endpoint: restrict by network here, and EmailEngine still
    # requires a token with the metrics scope
    location /metrics {
        allow 127.0.0.1;
        allow 10.0.0.0/8;   # Your internal network
        deny all;

        proxy_pass http://emailengine_backend;
        access_log off;
    }
}
```

## Advanced Features

### High Availability with Failover

:::warning No Horizontal Scaling
EmailEngine does NOT support horizontal scaling with load balancing. Multiple instances connecting to the same Redis will cause conflicts. The configuration below is for failover/high availability only.
:::

**Failover configuration (primary + cold standby):**

```nginx
upstream emailengine_backend {
    server emailengine-primary.internal:3000 max_fails=3 fail_timeout=30s;
    server emailengine-standby.internal:3000 backup;  # Only used if primary fails
}
```

This configuration:
- Routes all traffic to the primary instance
- Automatically fails over to standby if primary is down
- Both instances should NOT run simultaneously (standby should be stopped unless primary fails)
- Both need separate Redis instances OR the standby stays stopped

### IP Whitelisting

EmailEngine has its own allowlist for the admin interface, [`EENGINE_ADMIN_ACCESS_ADDRESSES`](/docs/deployment/security#admin-interface-access-control), which works without a proxy and is the one to reach for first. Nginx rules add a second layer that stops the request before it reaches EmailEngine at all:

```nginx
# Admin interface
location /admin {
    allow 203.0.113.0/24;  # Office network
    allow 198.51.100.42;   # VPN server
    deny all;

    proxy_pass http://emailengine_backend;
}

# API access
location /v1/ {
    # Whitelist
    allow 203.0.113.0/24;
    allow 198.51.100.0/24;
    deny all;

    proxy_pass http://emailengine_backend;
}
```

A `location /admin` block of its own needs the same `proxy_set_header` lines as `location /`. It does not capture `/admin/changes`: Nginx picks the longest matching prefix and then still tries the regular-expression locations, and the first regex that matches wins, so the streaming block keeps serving the dashboard feed. Declaring the prefix as `location ^~ /admin` would switch that regex check off, so leave the `^~` out.

### Custom Error Pages

**Branded error pages:**

```nginx
server {
    # Error pages
    error_page 500 502 503 504 /50x.html;
    error_page 404 /404.html;

    location = /50x.html {
        root /var/www/emailengine/errors;
        internal;
    }

    location = /404.html {
        root /var/www/emailengine/errors;
        internal;
    }
}
```

**Create custom error page `/var/www/emailengine/errors/50x.html`:**

```html
<!DOCTYPE html>
<html>
<head>
    <title>EmailEngine - Service Unavailable</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; padding: 50px; }
        h1 { font-size: 50px; }
    </style>
</head>
<body>
    <h1>503</h1>
    <p>EmailEngine is temporarily unavailable.</p>
    <p>Please try again in a few moments.</p>
</body>
</html>
```

API clients receive these pages instead of JSON when the upstream is down, so keep `error_page` for the admin interface or accept that a 502 during a restart comes back as HTML.

## Security Hardening

### Basic Authentication

EmailEngine's admin interface has its own login, with optional two-factor authentication and passkeys. HTTP basic authentication in Nginx adds a second prompt in front of it, which is useful when the admin interface must not be reachable at all without a shared credential:

```bash
# Create password file
sudo apt-get install apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd admin
```

```nginx
location /admin {
    auth_basic "EmailEngine Admin";
    auth_basic_user_file /etc/nginx/.htpasswd;
    proxy_pass http://emailengine_backend;
}
```

Do not apply `auth_basic` to `/v1/`: API clients authenticate with the `Authorization: Bearer` header, which basic authentication would intercept.

### Fail2Ban Integration

**Create Nginx filter `/etc/fail2ban/filter.d/nginx-emailengine.conf`:**

```ini
[Definition]
failregex = ^<HOST> -.*"(GET|POST|PUT|DELETE|HEAD).*HTTP.*" (401|403)
ignoreregex =
```

Match on 401 and 403 only. A 404 is a normal API answer (a message that no longer exists, a mailbox that was renamed), and banning on it would lock out a working integration.

**Configure jail `/etc/fail2ban/jail.local`:**

```ini
[nginx-emailengine]
enabled = true
port = http,https
filter = nginx-emailengine
logpath = /var/log/nginx/emailengine-access.log
maxretry = 5
bantime = 3600
```

## Monitoring and Logging

### Log Rotation

**Configure logrotate `/etc/logrotate.d/nginx-emailengine`:**

```
/var/log/nginx/emailengine-*.log {
    daily
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
    endscript
}
```

### Access Log Analysis

**Useful commands:**

```bash
# Top IPs
awk '{print $1}' /var/log/nginx/emailengine-access.log | sort | uniq -c | sort -rn | head -10

# Status codes
awk '{print $9}' /var/log/nginx/emailengine-access.log | sort | uniq -c | sort -rn

# Top endpoints
awk '{print $7}' /var/log/nginx/emailengine-access.log | sort | uniq -c | sort -rn | head -10
```

### Metrics Export

**Export metrics to Prometheus:**

```nginx
# Install nginx-prometheus-exporter
# https://github.com/nginxinc/nginx-prometheus-exporter

location /nginx_status {
    stub_status on;
    access_log off;
    allow 127.0.0.1;
    deny all;
}
```

EmailEngine's own metrics are separate: `/metrics` on the EmailEngine port, read with a `metrics`-scoped token. See [Monitoring](/docs/advanced/monitoring).

## Performance Optimization

### Connection Optimization

```nginx
# nginx.conf
worker_processes auto;
worker_rlimit_nofile 65535;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    # Keep-alive
    keepalive_timeout 65;
    keepalive_requests 100;

    # Buffers
    client_body_buffer_size 128k;
    client_max_body_size 50m;  # Allow large email submissions
    client_header_buffer_size 1k;
    large_client_header_buffers 4 16k;

    # Timeouts
    client_body_timeout 12;
    client_header_timeout 12;
    send_timeout 10;
}
```

### Compression

```nginx
http {
    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        application/rss+xml
        font/truetype
        font/opentype
        application/vnd.ms-fontobject
        image/svg+xml;
    gzip_disable "msie6";

    # Brotli compression (if the module is installed)
    brotli on;
    brotli_comp_level 6;
    brotli_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
}
```

The streaming locations above turn `gzip off` for themselves; a compressed event stream would be held in the compressor's buffer just like a proxied one.

## Example Configurations

### Single Instance

```nginx
upstream emailengine {
    server 127.0.0.1:3000;
}

server {
    listen 443 ssl http2;
    server_name emailengine.example.com;

    ssl_certificate /etc/ssl/certs/emailengine-fullchain.pem;
    ssl_certificate_key /etc/ssl/private/emailengine-privkey.pem;

    # Server-Sent Events feed (no gzip, no buffering)
    location ~ ^/(admin|v1)/changes {
        gzip off;
        proxy_pass http://emailengine;
        proxy_http_version 1.1;
        proxy_set_header Connection '';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 24h;
    }

    location / {
        client_max_body_size 50M;
        proxy_pass http://emailengine;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### High Availability (Failover)

:::warning
This is for automatic failover only, NOT load balancing. Only the primary instance should be running. The standby should be stopped and manually started if the primary fails.
:::

```nginx
upstream emailengine {
    server emailengine-primary.internal:3000 max_fails=3 fail_timeout=30s;
    server emailengine-standby.internal:3000 backup;  # Backup instance (keep stopped)
}

server {
    listen 443 ssl http2;
    server_name emailengine.example.com;

    location / {
        proxy_pass http://emailengine;
        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries 2;
    }
}
```

**Note:** The standby instance should remain stopped under normal operation. Nginx's `backup` directive means it will only be contacted if the primary is unavailable.

## See Also

- [Security](/docs/deployment/security) - TLS, headers, and access restrictions around the proxy
- [Linux installation](/docs/installation/linux) - The same proxy configuration alongside the install
- [Environment variables](/docs/configuration/environment-variables) - `EENGINE_API_PROXY` and the trusted proxy addresses
- [Change stream](/docs/api/get-v-1-changes) - The Server-Sent Events endpoint the unbuffered location exists for
- [MCP protocol](/docs/mcp/protocol) - Extra proxy requirements for the streaming endpoint
