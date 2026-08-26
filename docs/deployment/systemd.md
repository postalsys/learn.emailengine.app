---
title: SystemD Service
description: Run EmailEngine as a systemd service on Linux, with the unit file, environment file, hardening and restart policy
sidebar_position: 3
---

# SystemD Service Deployment

Run EmailEngine as a background service on Linux under systemd, so it starts at boot, restarts on failure and logs to the journal.

:::info Prerequisites
- A Linux distribution that uses systemd
- Redis running on the host or reachable from it
- EmailEngine installed: the binary from [Linux installation](/docs/installation/linux), or a source checkout with Node.js 20+ (24+ recommended) from [Source installation](/docs/installation/source)
:::

## Overview

EmailEngine fits systemd's `Type=simple` model without adaptation:

- It stays in the foreground and does not fork
- It writes JSON log lines to stdout, which journald captures
- It shuts down on `SIGTERM`

There are three layouts in use, and the unit file differs for each:

| Layout | Binary or code | Unit file | Documented in |
|--------|----------------|-----------|---------------|
| Installer script | `/opt/emailengine` | Written by the installer | [Automated installer](/docs/installation/linux#method-1-automated-installer-ubuntudebian), summarized [below](#the-installer-layout) |
| Manual binary install | `/usr/local/bin/emailengine` | The unit on this page | This page |
| Source checkout | A clone plus `npm install --omit=dev` | `systemd/emailengine.service` in the repository | [Source installation](/docs/installation/source), summarized [below](#the-source-checkout-layout) |

## The Installer Layout

`install.sh`, the script served at `https://go.emailengine.app`, writes its own unit file and there is nothing to add to it. What it produces:

- The binary at `/opt/emailengine`, owned by a system user named `emailengine`
- `/etc/systemd/system/emailengine.service` with `WorkingDirectory=/opt`, `ExecStart=/opt/emailengine`, `User=emailengine`, `After=redis-server`, `Restart=always` and `SyslogIdentifier=emailengine`
- The configuration inline in the unit as `Environment=` lines: `EENGINE_REDIS` (database 8, with the Redis password it generated), `EENGINE_PORT=3000`, `EENGINE_SECRET`, `EENGINE_API_PROXY=true`, `EENGINE_WORKERS=8`, `EENGINE_LOG_LEVEL=info` and `EENGINE_INSTALL_SCRIPT=true`, which labels the install method in the [license beacon](/docs/licensing#what-a-licensed-instance-sends-home)
- Redis configured with `requirepass` and `maxmemory-policy noeviction`
- Caddy in front of port 3000, terminating TLS for the domain you gave it
- The generated Redis password and `EENGINE_SECRET` in `/root/emailengine-credentials.txt`
- An upgrade helper at `/opt/upgrade-emailengine.sh`

The service commands in the rest of this page apply to that unit as well. To change its environment, edit the unit with `sudo systemctl edit --full emailengine` and restart.

## Manual Setup

### 1. Install the Binary

```bash
wget https://go.emailengine.app/emailengine.tar.gz
tar xzf emailengine.tar.gz
sudo mv emailengine /usr/local/bin/
sudo chmod +x /usr/local/bin/emailengine

emailengine --version
```

### 2. Create the Service User

```bash
sudo useradd --system --home /var/lib/emailengine --create-home --shell /usr/sbin/nologin emailengine
```

The home directory is the unit's working directory. EmailEngine reads a `.env` file from its working directory if one exists.

### 3. Write the Environment File

Keeping the secrets in a root-only file rather than in the unit itself matters because unit files are world-readable. systemd reads `EnvironmentFile=` as root before dropping to the service user, so the file needs no permissions for `emailengine` at all.

```bash
sudo mkdir -p /etc/emailengine
sudo tee /etc/emailengine/emailengine.env > /dev/null <<EOF
EENGINE_REDIS=redis://127.0.0.1:6379/8
EENGINE_SECRET=$(openssl rand -hex 32)
EENGINE_WORKERS=4
EENGINE_LOG_LEVEL=info
EOF
sudo chmod 600 /etc/emailengine/emailengine.env
```

Add `EENGINE_API_PROXY=true` when a reverse proxy sits in front of EmailEngine, so the client address is read from `X-Forwarded-For`, and `EENGINE_API_PROXY_ADDRESSES` naming the proxy; see [Trusted Proxy Addresses](/docs/configuration/environment-variables#trusted-proxy-addresses). The API binds to `127.0.0.1` unless `EENGINE_HOST` says otherwise, which is the right default behind a proxy.

:::danger Keep EENGINE_SECRET
`EENGINE_SECRET` encrypts the account credentials stored in Redis. Back it up with the Redis data: without it the stored credentials cannot be decrypted. See [Secret Encryption](/docs/advanced/encryption).
:::

### 4. Create the Unit File

Create `/etc/systemd/system/emailengine.service`:

```ini
[Unit]
Description=EmailEngine
Documentation=https://learn.emailengine.app/
# On Debian and Ubuntu the Redis unit is redis-server.service; on most other
# distributions it is redis.service. Name the one that exists on this host.
After=network-online.target redis-server.service
Wants=network-online.target
# Give up after five restarts within five minutes rather than looping
StartLimitIntervalSec=300
StartLimitBurst=5

[Service]
Type=simple
User=emailengine
Group=emailengine
WorkingDirectory=/var/lib/emailengine
EnvironmentFile=/etc/emailengine/emailengine.env
Environment="NODE_ENV=production"
ExecStart=/usr/local/bin/emailengine

Restart=always
RestartSec=5

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=emailengine

# Limits, matching the example unit in the EmailEngine repository
LimitNOFILE=500000
LimitNPROC=500000
LimitFSIZE=infinity

# Hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/emailengine
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
RestrictRealtime=true
LockPersonality=true

[Install]
WantedBy=multi-user.target
```

`ProtectSystem=strict` makes the whole file system read-only for the service except the paths in `ReadWritePaths`. [Exports](/docs/receiving/exporting) are written to the temporary directory, which `PrivateTmp` keeps writable; if you point `EENGINE_EXPORT_PATH` or the `exportPath` setting somewhere else, add that directory to `ReadWritePaths`. Do not add `MemoryDenyWriteExecute=true`: Node.js needs writable executable memory for its JIT compiler and cannot run with it set.

### 5. Enable and Start

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now emailengine
sudo systemctl status emailengine
```

### 6. Verify

```bash
curl http://127.0.0.1:3000/health
```

A healthy instance answers `{"success":true}`. The endpoint needs no token and returns `500` when not every IMAP worker thread is running or a Redis write-read-delete round trip fails.

## The Source Checkout Layout

The EmailEngine repository carries `systemd/emailengine.service`, an example unit for a checkout that was prepared with `npm install --omit=dev`. Its notable differences from the unit above:

- `WorkingDirectory` points at the checkout, and `ExecStart=/usr/bin/npm start` runs it (`which npm` gives the path on your host)
- The commented alternative `ExecStart=/usr/bin/npx emailengine@<version>` downloads the named version on first start, so no checkout is needed; upgrading is a matter of changing the version and restarting
- It runs as `www-data`, which already exists on most systems; any unprivileged user with read access to the checkout works
- Configuration is passed as `Environment=` lines, or through a configuration file named in `NODE_CONFIG_PATH`

[Source installation](/docs/installation/source) has the full procedure.

## Configuration File Instead of Environment Variables

Environment variables cover every option, but EmailEngine also reads a configuration file whose values are merged over the built-in defaults. The file may be TOML, JSON or JavaScript, and is named either with the `--config` argument on `ExecStart` or with the `NODE_CONFIG_PATH` environment variable:

```ini
[Service]
Environment="NODE_CONFIG_PATH=/etc/emailengine/emailengine.toml"
```

A file equivalent to the environment file above:

```toml
[dbs]
redis = "redis://127.0.0.1:6379/8"

[service]
secret = "your-generated-secret"

[api]
port = 3000
host = "127.0.0.1"

[workers]
imap = 4

[log]
level = "info"
```

Make it readable by the service user, since EmailEngine rather than systemd opens it:

```bash
sudo chown root:emailengine /etc/emailengine/emailengine.toml
sudo chmod 640 /etc/emailengine/emailengine.toml
```

The keys are listed under [Configuration Files](/docs/configuration/cli#configuration-files). When a value is set in both places, the environment variable wins.

## Service Management

```bash
sudo systemctl start emailengine
sudo systemctl stop emailengine
sudo systemctl restart emailengine
sudo systemctl status emailengine

sudo systemctl is-active emailengine
sudo systemctl is-enabled emailengine
sudo systemctl show emailengine -p NRestarts
```

EmailEngine has no configuration reload; after changing the environment file or the unit, run `sudo systemctl daemon-reload` and restart.

**Example status output:**

```text
● emailengine.service - EmailEngine
     Loaded: loaded (/etc/systemd/system/emailengine.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-08-24 10:00:00 UTC; 2h 30min ago
       Docs: https://learn.emailengine.app/
   Main PID: 12345 (emailengine)
      Tasks: 15 (limit: 4915)
     Memory: 512.5M
     CGroup: /system.slice/emailengine.service
             └─12345 /usr/local/bin/emailengine
```

## Logs

EmailEngine writes one JSON object per line to stdout, and journald stores them:

```bash
# Follow
sudo journalctl -u emailengine -f

# Since a point in time
sudo journalctl -u emailengine --since "2026-08-24 10:00:00"
sudo journalctl -u emailengine --since "1 hour ago"

# Last 100 lines
sudo journalctl -u emailengine -n 100

# Errors only
sudo journalctl -u emailengine -p err

# Raw lines, for piping into jq
sudo journalctl -u emailengine -o cat | jq .
```

`EENGINE_LOG_LEVEL` in the environment file sets the verbosity; the default is `trace`, which is far more than a production host wants. [Logging](/docs/advanced/logging) describes the levels and the fields.

### Journal Retention

Journald rotates on its own. To bound what it keeps, edit `/etc/systemd/journald.conf`:

```ini
[Journal]
SystemMaxUse=1G
SystemMaxFileSize=100M
MaxRetentionSec=7day
```

Then `sudo systemctl restart systemd-journald`.

### Log Files

EmailEngine does not write log files itself. If a file is required, for example for a collector that cannot read the journal, let systemd append stdout to one:

```ini
[Service]
StandardOutput=append:/var/log/emailengine/emailengine.log
StandardError=inherit
```

Create the directory writable by the service user and add it to `ReadWritePaths`. Rotate with `logrotate` using `copytruncate`, since the process keeps the file open and cannot be told to reopen it.

## Resource Limits and Memory

### Restart on Memory Threshold

`MemoryMax` makes the kernel kill the service when its cgroup exceeds the limit, and `Restart=always` brings it back:

```ini
[Service]
MemoryMax=4G
Restart=always
RestartSec=5
```

The journal records the kill:

```text
emailengine.service: A process of this unit has been killed by the OOM killer.
emailengine.service: Main process exited, code=killed, status=9/KILL
emailengine.service: Scheduled restart job, restart counter is at 1.
Started emailengine.service - EmailEngine.
```

Memory use scales with the number of accounts and the size of their mailboxes. Watch `systemctl status emailengine` for a while before choosing the limit, and leave headroom for Redis and the operating system. `StartLimitIntervalSec` and `StartLimitBurst` in the `[Unit]` section above stop a service that dies immediately after every restart from looping; `systemctl reset-failed emailengine` clears the counter once the cause is fixed.

### CPU

```ini
[Service]
CPUQuota=200%
```

Caps the service at two cores' worth of CPU time. EmailEngine's IMAP work is spread across `EENGINE_WORKERS` threads, so a quota lower than the worker count leaves threads waiting.

## Multiple Instances on One Host

Each instance needs its own Redis database and its own port. Two instances on the same Redis database both sync every account and corrupt each other's state.

A template unit reads a per-instance environment file. Create `/etc/systemd/system/emailengine@.service` as a copy of the unit above with these lines changed:

```ini
[Service]
EnvironmentFile=/etc/emailengine/%i.env
WorkingDirectory=/var/lib/emailengine/%i
ReadWritePaths=/var/lib/emailengine/%i
```

Then give each instance a file that names a distinct database and port:

```bash
# /etc/emailengine/tenant-a.env
EENGINE_REDIS=redis://127.0.0.1:6379/8
EENGINE_PORT=3000
EENGINE_SECRET=3f0c9d7a2b1e4c8f9a6d5e4b3c2a1f0e9d8c7b6a5f4e3d2c1b0a9f8e7d6c5b4a

# /etc/emailengine/tenant-b.env
EENGINE_REDIS=redis://127.0.0.1:6379/9
EENGINE_PORT=3001
EENGINE_SECRET=a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5
```

```bash
sudo mkdir -p /var/lib/emailengine/tenant-a /var/lib/emailengine/tenant-b
sudo chown emailengine:emailengine /var/lib/emailengine/tenant-*
sudo systemctl enable --now emailengine@tenant-a emailengine@tenant-b
```

## Shutdown

On `systemctl stop`, systemd sends `SIGTERM` and EmailEngine closes its IMAP connections and exits. Give it long enough to do so on a host with many accounts:

```ini
[Service]
TimeoutStopSec=30
KillMode=mixed
```

`KillMode=mixed` sends `SIGTERM` to the main process only and `SIGKILL` to anything left in the cgroup once the timeout expires.

## Monitoring

### Prometheus

`GET /metrics` serves Prometheus metrics to a token holding the `metrics` scope. The CLI writes tokens straight into Redis, so give it the same connection string the service uses:

```bash
emailengine tokens issue -d "Prometheus" -s "metrics" --dbs.redis="redis://127.0.0.1:6379/8"
```

```bash
curl -H "Authorization: Bearer YOUR_TOKEN" http://127.0.0.1:3000/metrics
```

[Monitoring](/docs/advanced/monitoring) lists the metrics.

### Commands

```bash
# CPU and memory of the service cgroup
sudo systemd-cgtop

# Units in the failed state
sudo systemctl list-units --failed

# Restart count since boot
sudo systemctl show emailengine -p NRestarts
```

## Updates

Replace the binary and restart. The service is down for the duration of the restart only.

```bash
# Latest release
wget https://go.emailengine.app/emailengine.tar.gz

# Or a specific release, for example
# wget https://go.emailengine.app/download/v2.79.4/emailengine.tar.gz

tar xzf emailengine.tar.gz
sudo mv emailengine /usr/local/bin/
sudo chmod +x /usr/local/bin/emailengine
sudo systemctl restart emailengine

emailengine --version
```

Installer-managed hosts run `/opt/upgrade-emailengine.sh` instead, which downloads the latest release, compares versions and restarts the service only when they differ.

## Backup

The state is in Redis and the secret is in the environment file. Back up both:

```bash
sudo tar czf emailengine-config-$(date +%Y%m%d).tar.gz \
  /etc/emailengine/ \
  /etc/systemd/system/emailengine.service

sudo cp /var/lib/redis/dump.rdb /backup/emailengine-$(date +%Y%m%d).rdb
```

`dump.rdb` is only as current as the last Redis snapshot; see [Redis](/docs/configuration/redis#persistence-configuration-recommended) for the persistence settings.

## See Also

- [Linux installation](/docs/installation/linux) - Getting the binary in place first, or letting the installer do all of this
- [Source installation](/docs/installation/source) - The unit file for a source deployment
- [Configuration files](/docs/configuration/cli#configuration-files) - Every key the TOML file accepts
- [Logging](/docs/advanced/logging) - Reading what the unit writes to the journal
- [Security](/docs/deployment/security) - What to lock down once the service is running
