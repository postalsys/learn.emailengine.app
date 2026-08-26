---
title: Docker Installation
description: Run EmailEngine in Docker containers with Docker Compose
sidebar_position: 5
---

# Running EmailEngine in Docker

Running EmailEngine and its Redis from containers, with a single `docker run` or with Compose.

## Overview

EmailEngine provides official Docker images for easy deployment:

- **Pre-built images**: Published to Docker Hub (`postalsys/emailengine`) and GitHub Container Registry (`ghcr.io/postalsys/emailengine`) by the same workflow
- **Multi-architecture**: `linux/amd64` and `linux/arm64` manifests, each built on a runner of that architecture
- **Self-contained**: A Node.js Alpine base with the application and its production dependencies; Redis is not included
- **Preset for containers**: The image sets `EENGINE_HOST=0.0.0.0` so the API listens on the container's interfaces, and `EENGINE_API_PROXY=true` so client addresses are read from `X-Forwarded-For`. It runs as an unprivileged user under `dumb-init`

## Quick Start

### Basic Docker Run

Run EmailEngine with an external Redis instance:

```bash
docker run -d \
  --name emailengine \
  -p 3000:3000 \
  -e EENGINE_REDIS="redis://redis-host:6379/8" \
  -e EENGINE_SECRET="your-secret-key-at-least-32-chars" \
  postalsys/emailengine:v2
```

Test the installation:

```bash
curl http://localhost:3000/health
# Should return: {"success":true}
```

Access web interface at: `http://localhost:3000`

## Docker Compose (Recommended)

Use Docker Compose to run EmailEngine with Redis.

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

<Tabs>
<TabItem value="official" label="Official Configuration" default>

### Quick Start with Official docker-compose.yml

Download and run the official configuration:

```bash
# Download official docker-compose.yml
curl -LO https://go.emailengine.app/docker-compose.yml

# Generate a secure encryption secret
echo "EENGINE_SECRET=$(openssl rand -hex 32)" > .env

# Start EmailEngine
docker compose up -d
```

The URL redirects to the `docker-compose.yml` on the master branch of the EmailEngine repository, so what you download tracks the current file rather than a release.

#### What's Included

The shipped configuration defines two services on a private bridge network:

- **EmailEngine** from `postalsys/emailengine:latest` (override with `EMAILENGINE_VERSION`), with the API, SMTP submission, and IMAP proxy ports published on `127.0.0.1` only
- **Redis** from `redis:7-alpine` with `maxmemory-policy noeviction`, no memory limit, and a named `redis-data` volume. With `REDIS_PASSWORD` set it also passes explicit RDB snapshot rules (`save 900 1`, `save 300 10`, `save 60 10000`); without it Redis runs with its built-in defaults, which also snapshot to disk
- **Health checks** on both services (`wget --spider http://localhost:3000/health` for EmailEngine, `redis-cli ping` for Redis), and EmailEngine waits for Redis to report healthy before starting
- **Every value read from `.env`**, with a default for each
- **json-file logging** with rotation (`100m`, 10 files) and compression

The header of the file states its own sizing: 8 GB of memory and 4 cores as the production minimum, 16 GB and 8 cores recommended, because Redis in the same file gets no memory limit.

#### Access Points

After starting:

- **Web UI & API:** http://localhost:3000
- **SMTP Server:** localhost:2525. The port is published, but the SMTP submission server is off until enabled: the `.env.production` template in the repository does it through `EENGINE_SETTINGS` (see below), or use **Configuration > SMTP Server** in the admin UI
- **IMAP Proxy:** localhost:9993. Published the same way, and likewise off until `imapProxyServerEnabled` is set, or **Configuration > IMAP Proxy** turns it on

#### Environment Configuration

Customize settings in `.env` file:

```bash
# Required: Encryption secret (generate with: openssl rand -hex 32).
# Without it the file falls back to "default-dev-secret-change-in-production"
EENGINE_SECRET=your-generated-secret-here

# Recommended: Redis password. Setting it only configures the Redis container;
# EmailEngine's own connection URL must carry the same password, so set both.
# The default EENGINE_REDIS is redis://redis:6379/2 (database 2)
REDIS_PASSWORD=your-redis-password
EENGINE_REDIS=redis://:your-redis-password@redis:6379/2

# Optional: Version pinning (default: latest)
EMAILENGINE_VERSION=latest
REDIS_VERSION=7-alpine

# Optional: Custom port bindings (default: 127.0.0.1)
EMAILENGINE_API_BIND=0.0.0.0
EMAILENGINE_API_PORT=3000
EMAILENGINE_SMTP_BIND=0.0.0.0
EMAILENGINE_SMTP_PORT=2525
EMAILENGINE_IMAP_BIND=0.0.0.0
EMAILENGINE_IMAP_PORT=9993

# Optional: Settings written to the instance on startup. This is how the
# repository's .env.production template turns on the SMTP submission server
EENGINE_SETTINGS={"smtpServerEnabled": true, "smtpServerPort": 2525, "smtpServerHost": "0.0.0.0", "smtpServerAuthEnabled": true, "smtpServerPassword": "change-me"}

# Optional: Performance tuning
EENGINE_WORKERS=4
EENGINE_LOG_LEVEL=info

# Optional: Restart policy and health check timing
RESTART_POLICY=unless-stopped
HEALTHCHECK_START_PERIOD=40s
```

The repository ships two templates for this file, `.env.development` (binds to all interfaces, no Redis password, the default secret) and `.env.production` (localhost binding, password required), plus `setup-production.sh`, which generates the secrets and writes `.env` for you.

</TabItem>
<TabItem value="custom" label="Custom Configuration">

### Custom docker-compose.yml

If you prefer to create your own configuration:

Create `docker-compose.yml`:

```yaml
services:
  redis:
    image: redis:7-alpine
    container_name: emailengine-redis
    command: redis-server --maxmemory-policy noeviction --save 900 1 --save 300 10 --save 60 10000
    volumes:
      - redis-data:/data
    networks:
      - emailengine
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3

  emailengine:
    image: postalsys/emailengine:v2
    container_name: emailengine
    ports:
      - "3000:3000"
    environment:
      - EENGINE_REDIS=redis://redis:6379
      - EENGINE_SECRET=change-this-to-a-random-secret-at-least-32-chars
      - EENGINE_WORKERS=4
      - EENGINE_LOG_LEVEL=info
    depends_on:
      redis:
        condition: service_healthy
    networks:
      - emailengine
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  emailengine:
    driver: bridge

volumes:
  redis-data:
    driver: local
```

**Generate secure secret:**

```bash
openssl rand -hex 32
```

Update `EENGINE_SECRET` in the docker-compose.yml with the generated value.

</TabItem>
</Tabs>

### Start Services

```bash
# Start in background
docker compose up -d

# View logs
docker compose logs -f

# View specific service logs
docker compose logs -f emailengine
docker compose logs -f redis

# Check status
docker compose ps
```

The commands use the Compose plugin (`docker compose`). The standalone `docker-compose` binary reads the same file.

### Stop Services

```bash
# Stop containers
docker compose stop

# Stop and remove containers
docker compose down

# Stop and remove containers and volumes
docker compose down -v
```

## Production Deployment

For production deployments, we recommend using the official docker-compose.yml (see "Official Configuration" tab above) with a properly configured `.env` file.

### Production Environment Configuration

Create a production-ready `.env` file:

```bash
# Required: Encryption secret (generate with: openssl rand -hex 32)
EENGINE_SECRET=your-generated-secret-here

# Recommended: Redis password, and the connection URL that carries it
REDIS_PASSWORD=your-secure-redis-password
EENGINE_REDIS=redis://:your-secure-redis-password@redis:6379/2

# Version pinning for stability
EMAILENGINE_VERSION=v2.79.4

# Bind to all interfaces (if behind reverse proxy)
EMAILENGINE_API_BIND=0.0.0.0
EMAILENGINE_SMTP_BIND=0.0.0.0
EMAILENGINE_IMAP_BIND=0.0.0.0

# Performance tuning (adjust based on server resources)
EENGINE_WORKERS=8
EENGINE_LOG_LEVEL=warn

# Automatic restart policy
RESTART_POLICY=unless-stopped
```

### Security Recommendations

1. **Use Redis password:** Set `REDIS_PASSWORD` for production
2. **Pin versions:** Specify exact versions (e.g., `EMAILENGINE_VERSION=v2.79.4`)
3. **Bind to localhost:** If using reverse proxy, keep default `127.0.0.1` binding
4. **Restrict access:** Use firewall rules to limit port access
5. **Enable TLS:** Use reverse proxy (Nginx/Caddy) with HTTPS

### Resource Requirements

The [general figures](/docs/installation#system-requirements) apply to the EmailEngine container: 2 GB of memory to evaluate, 4 to 8 GB for production, and more once the account count climbs. Redis in the same compose file competes for the same memory and has no limit of its own, which is why the shipped file's header asks for 8 GB and 4 cores as a production minimum. Size for both.

Watch what the container actually uses before deciding: `docker stats emailengine`

### Volumes and Persistence

With the official configuration, redis data is stored in `redis-data` volume:

```yaml
volumes:
  redis-data:
    driver: local
```

To backup redis data:

```bash
# Create backup (add -a with the password when REDIS_PASSWORD is set)
docker compose exec redis redis-cli SAVE
docker compose cp redis:/data/dump.rdb ./backup-$(date +%Y%m%d).rdb

# Restore backup
docker compose stop redis
docker compose cp ./backup-20231201.rdb redis:/data/dump.rdb
docker compose start redis
```

`docker compose cp` addresses the service by name, so it works whether the container is called `emailengine-redis-1` (the shipped file) or `emailengine-redis` (the custom file above).

### With Custom Redis Configuration

Create `redis.conf`:

```conf
# Persistence. RDB snapshots rather than AOF, see the Redis page for why
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes

# Memory
maxmemory-policy noeviction

# Performance
tcp-backlog 511
timeout 300
tcp-keepalive 300

# Limits
maxclients 10000

# Security (if needed)
# requirepass your-redis-password
```

If using Redis password, update `.env` (keep the database number the shipped file uses):

```bash
EENGINE_REDIS=redis://:your-redis-password@redis:6379/2
```

### With Reverse Proxy (Nginx)

Add an `nginx` service next to `redis` and `emailengine` in `docker-compose.yml`:

```yaml
services:
  nginx:
    image: nginx:alpine
    container_name: emailengine-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - emailengine
    networks:
      - emailengine
    restart: unless-stopped
```

For `nginx.conf`, use the server block on the [Nginx reverse proxy](/docs/deployment/nginx-proxy) page with one change: the upstream is the Compose service name, so `proxy_pass http://emailengine:3000;` instead of `http://localhost:3000`. That page also carries the settings the `/admin/changes` and `/v1/changes` EventSource streams need, which a plain proxy block breaks.

## Docker Images

### Available Tags

Every release publishes four tags, and master publishes one:

| Tag | Points at | Use it when |
|-----|-----------|-------------|
| `v2.79.4` | One exact release | You want the image to change only when you change it. The safest choice for production |
| `v2.79` | Newest patch of that minor | You want patch fixes automatically, without minor upgrades |
| `v2` | Newest release of the v2 line | You want releases automatically. There is no v3, so this is currently the newest stable build |
| `latest` | Newest release, same build as `v2` today | Convenience. It will follow a future major version, so avoid pinning production to it |
| `edge` | Latest commit on master | Testing unreleased changes. Not for production |

:::warning `edge` is the development tag, not `latest`
`latest` is published by the release workflow and always points at a tagged release. Builds from master are published as `edge`. If you want to try unreleased work, pull `edge` explicitly.
:::

### Image Sources

**Docker Hub (primary):**

```bash
docker pull postalsys/emailengine:v2
docker pull postalsys/emailengine:v2.79.4
```

**GitHub Container Registry (alternative):**

```bash
docker pull ghcr.io/postalsys/emailengine:v2
```

### Multi-Architecture Support

Images support both AMD64 and ARM64 architectures:

- Intel/AMD processors (x86_64)
- Apple Silicon (M1 and newer)
- ARM servers

Docker automatically pulls the correct architecture.

## Docker Commands

### Container Management

```bash
# View running containers
docker ps

# View all containers
docker ps -a

# Start container
docker start emailengine

# Stop container
docker stop emailengine

# Restart container
docker restart emailengine

# Remove container
docker rm emailengine

# View logs
docker logs emailengine
docker logs -f emailengine  # Follow logs
docker logs --tail 100 emailengine  # Last 100 lines
```

### Image Management

```bash
# List images
docker images

# Pull latest tagged release
docker pull postalsys/emailengine:v2

# Pull specific version
docker pull postalsys/emailengine:v2.79.4

# Remove image
docker rmi postalsys/emailengine:v2

# Remove unused images
docker image prune
```

### Inspect Container

```bash
# View container details
docker inspect emailengine

# View environment variables
docker exec emailengine env

# Access container shell
docker exec -it emailengine sh

# Test health endpoint from inside container (the Alpine image has wget, not curl)
docker exec emailengine wget -qO- http://localhost:3000/health
```

## Upgrading

### Docker Compose Upgrade

```bash
# Pull latest tagged release
docker compose pull

# Restart with new image
docker compose up -d

# View logs to verify
docker compose logs -f emailengine
```

### Docker Run Upgrade

```bash
# Pull latest tagged release
docker pull postalsys/emailengine:v2

# Stop and remove old container
docker stop emailengine
docker rm emailengine

# Start new container with same configuration
docker run -d \
  --name emailengine \
  -p 3000:3000 \
  -e EENGINE_REDIS="redis://redis-host:6379/8" \
  -e EENGINE_SECRET="your-secret-key" \
  postalsys/emailengine:v2
```

### Specific Version

```bash
# Use specific version tag
docker pull postalsys/emailengine:v2.79.4

# In docker-compose.yml
services:
  emailengine:
    image: postalsys/emailengine:v2.79.4
```

## Environment Variables

### Required Variables

```bash
EENGINE_REDIS=redis://redis:6379
EENGINE_SECRET=your-secret-key-at-least-32-chars
```

### Optional Variables

```bash
# Performance
EENGINE_WORKERS=4              # Number of worker processes

# API settings
EENGINE_PORT=3000              # HTTP port
EENGINE_HOST=0.0.0.0          # Bind address (already set to 0.0.0.0 in the image)
EENGINE_API_PROXY=true        # Trust X-Forwarded-For (already set in the image)

# Logging
EENGINE_LOG_LEVEL=info        # trace, debug, info, warn, error, fatal, silent
EENGINE_LOG_RAW=false         # Log raw IMAP/SMTP traffic

# Max attachment size (default: 5MB)
EENGINE_MAX_SIZE=5M
```

See [Configuration Options](/docs/configuration) for complete list.

## Volume Management

### Persistent Data

Redis data is stored in Docker volumes to persist across container restarts.

```bash
# List volumes
docker volume ls

# Inspect volume
docker volume inspect emailengine_redis-data

# Backup volume
docker run --rm -v emailengine_redis-data:/data -v $(pwd):/backup alpine tar czf /backup/redis-backup.tar.gz /data

# Restore volume
docker run --rm -v emailengine_redis-data:/data -v $(pwd):/backup alpine tar xzf /backup/redis-backup.tar.gz -C /
```

## See Also

- [Installation overview](/docs/installation) - Every other way to run EmailEngine
- [Kubernetes](/docs/deployment/kubernetes) - Running the same image under an orchestrator
- [Redis configuration](/docs/configuration/redis) - Persistence, memory policy, and connection URLs
- [Environment variables](/docs/configuration/environment-variables) - Everything configurable at startup
- [Security](/docs/deployment/security) - Hardening a production deployment
