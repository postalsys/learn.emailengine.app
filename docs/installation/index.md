---
title: Install EmailEngine - Setup Guide for All Platforms
description: Install EmailEngine on Linux, macOS, Windows, or Docker, with one-click cloud deployments for Render, DigitalOcean, CapRover, and Heroku.
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

# Source: Production deployment
wget https://go.emailengine.app/source-dist.tar.gz
tar xzf source-dist.tar.gz && cd emailengine
node server.js
```

## Installation Methods

### By Operating System

#### [Linux Installation](/docs/installation/linux)

Install on Ubuntu, Debian, CentOS, or RHEL.

**Methods:**

- Automated installer (Ubuntu/Debian) - one-click setup
- Binary installation - standalone executable
- Source installation

**Best for:** Servers, VPS hosting, production deployments

[View Linux guide →](/docs/installation/linux)

---

#### [macOS Installation](/docs/installation/macos)

Install on macOS (Apple Silicon or Intel).

**Methods:**

- PKG installer - signed macOS package
- Homebrew + binary
- Source installation

**Best for:** Development, testing, local deployments

[View macOS guide →](/docs/installation/macos)

---

#### [Windows Installation](/docs/installation/windows)

Install on Windows 10+ (native or WSL2).

**Methods:**

- Windows executable - standalone .exe
- WSL2 installation - recommended for production
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

**Note:** Request SMTP port unblocking from DigitalOcean support.

#### CapRover

Deploy via One-Click Apps in CapRover dashboard. Search for "EmailEngine".

#### Heroku

[![Deploy to Heroku](https://www.herokucdn.com/deploy/button.svg)](https://heroku.com/deploy?template=https://github.com/postalsys/emailengine)

**Note:** Pick a dyno with at least 4 GB of memory. EmailEngine holds one connection per account, and the smaller dynos run out of memory before they run out of connections.

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
