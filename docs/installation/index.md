---
title: Install EmailEngine - Setup Guide for All Platforms
description: Install EmailEngine on Linux, macOS, Windows, or Docker, with one-click cloud deployments for Render, DigitalOcean, and Heroku.
sidebar_position: 1
keywords:
  - install EmailEngine
  - EmailEngine Docker
  - EmailEngine setup
  - self-hosted email API installation
  - email gateway setup
---

# Installing EmailEngine

Choose your installation method based on your operating system and deployment requirements.

## Quick Start

```bash
# npm: Install globally (requires Node.js 20+)
npm install -g emailengine-app
emailengine --dbs.redis="redis://127.0.0.1:6379/8"

# Linux: Download binary
wget https://go.emailengine.app/emailengine.tar.gz
tar xzf emailengine.tar.gz
sudo mv emailengine /usr/local/bin/

# Docker: Run container
docker run -p 3000:3000 --env EENGINE_REDIS="redis://host.docker.internal:6379/8" postalsys/emailengine:v2

# Source: Production deployment (the tarball has no top-level directory, so make one)
wget https://go.emailengine.app/source-dist.tar.gz
mkdir emailengine && tar xzf source-dist.tar.gz -C emailengine && cd emailengine
node server.js
```

## Installation Methods

### By Operating System

#### [Linux Installation](/docs/installation/linux)

Install on any distribution.

**Methods:**

- Automated installer (Ubuntu/Debian) - installs Redis, Caddy, and a SystemD service in one run
- Binary installation - standalone x86_64 executable
- Source installation

**Best for:** Servers, VPS hosting, production deployments

[View Linux guide →](/docs/installation/linux)

---

#### [macOS Installation](/docs/installation/macos)

Install on macOS (Apple Silicon or Intel).

**Methods:**

- PKG installer - signed and notarized package, one per architecture
- Source installation

**Best for:** Development, testing, local deployments

[View macOS guide →](/docs/installation/macos)

---

#### [Windows Installation](/docs/installation/windows)

Install on Windows (native or WSL2).

**Methods:**

- Windows executable - standalone .exe with a Redis-compatible server such as Memurai
- WSL2 installation - the Linux build inside Windows
- Docker Desktop

**Best for:** Development, testing, Windows servers

[View Windows guide →](/docs/installation/windows)

### By Deployment Type

#### [Docker Installation](/docs/installation/docker)

Run in containers with Docker or Docker Compose.

**Features:**

- Isolated environment
- Easy scaling
- Quick updates
- Consistent across platforms

**Best for:** Containerized infrastructure, Kubernetes, cloud deployments

[View Docker guide →](/docs/installation/docker)

---

#### [Source Installation](/docs/installation/source)

Run from source code (Node.js 20+ required, 24+ recommended).

[View source guide →](/docs/installation/source)

## System Requirements

### Minimum (Development/Testing)

- **CPU:** 1-2 cores
- **RAM:** 2 GB
- **Storage:** 10 GB
- **Network:** Stable internet connection

### Recommended (Production)

- **CPU:** 4+ cores
- **RAM:** 4-8 GB (or more for high-volume)
- **Storage:** 20+ GB SSD
- **Network:** Low-latency, high-bandwidth connection

## Cloud Platforms

### One-Click Deployments

EmailEngine is available on popular cloud platforms:

#### Render.com

[![Deploy to Render](https://render.com/images/deploy-to-render-button.svg)](https://render.com/deploy?repo=https://github.com/postalsys/emailengine)

Automatic setup with managed Redis.

[View Render guide →](/docs/deployment/render)

#### DigitalOcean Marketplace

[![DigitalOcean](/img/external/QBubXuGF1M.svg)](https://marketplace.digitalocean.com/apps/emailengine?refcode=90a107552b31)

One-click droplet with everything pre-configured.

**Note:** DigitalOcean blocks outbound SMTP ports 25, 465, and 587 on Droplets (per [Why is SMTP blocked?](https://docs.digitalocean.com/support/why-is-smtp-blocked/), checked 2026-08-26), so an EmailEngine on a Droplet can receive mail but cannot reach most providers' SMTP servers to send. Sending needs an SMTP relay on a port that is not blocked, or a host that allows SMTP.

#### Heroku

[![Deploy to Heroku](https://www.herokucdn.com/deploy/button.svg)](https://heroku.com/deploy?template=https://github.com/postalsys/emailengine)

The button reads the `app.json` in the EmailEngine repository. It provisions a `heroku-redis` add-on and sets `EENGINE_WORKERS=1` because the Heroku Redis add-on caps the number of client connections; raise the worker count only together with a larger Redis plan. It also sets `NODE_TLS_REJECT_UNAUTHORIZED=0`, which the template needs to reach the Heroku Redis add-on over TLS and which disables certificate validation for every outbound TLS connection EmailEngine makes.

[View all deployment guides →](/docs/deployment)

## Post-Installation Steps

After installing EmailEngine:

### 1. Verify Installation

```bash
# Check health endpoint
curl http://localhost:3000/health

# Expected response:
# {"success":true}
```

### 2. Access Web Interface

Open `http://localhost:3000` in your browser and create your admin account.

### 3. Configure OAuth2

Set up OAuth2 credentials for Gmail and Microsoft 365:

[OAuth2 Configuration Guide →](/docs/accounts/oauth2-setup)

### 4. Add Your First Account

Register an email account via the web interface or API:

[Account Setup Guide →](/docs/accounts)

### 5. Set Up Webhooks

Configure webhooks to receive real-time email notifications:

[Webhooks Guide →](/docs/webhooks/overview)

### 6. Secure Your Deployment

For production deployments, follow security best practices:

[Security Guide →](/docs/deployment/security)

## Getting Help

If you encounter issues during installation:

1. **Check platform-specific guide** for detailed instructions
2. **View troubleshooting documentation** for common problems
3. **Check GitHub issues** for known problems
4. **Contact support** for assistance

[Support page →](/docs/support)

## See Also

- [Quick Start](/docs/getting-started/quick-start) - What to do once it is running
- [Configuration](/docs/configuration) - Environment variables, Redis, and prepared settings
- [Deployment](/docs/deployment) - Reverse proxies, SystemD, Kubernetes, and hardening
- [Troubleshooting](/docs/troubleshooting) - Common install and startup problems
