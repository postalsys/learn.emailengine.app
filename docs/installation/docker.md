---
title: Docker Installation
description: Run EmailEngine in Docker containers with Docker Compose
sidebar_position: 5
---

# Running EmailEngine in Docker

Running EmailEngine and its Redis from containers, with a single `docker run` or with Compose.

## Overview

EmailEngine provides official Docker images for easy deployment:

- **Pre-built images**: Available on Docker Hub and GitHub Container Registry
- **Multi-architecture**: Supports AMD64 and ARM64 (Apple Silicon compatible)
- **Self-contained**: Includes all dependencies except Redis
- **Production-ready**: Suitable for containerized deployments

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
docker-compose up -d
```

#### What's Included

The official configuration includes:

- **EmailEngine** with API, SMTP, and IMAP proxy ports
- **Redis** with production settings and persistence
- **Health checks** and automatic restarts
- **Environment variable** configuration via `.env` file
- **Logging** with rotation and compression

#### Access Points

After starting:

- **Web UI & API:** http://localhost:3000
- **SMTP Server:** localhost:2525 (for message submission)
- **IMAP Proxy:** localhost:9993 (optional IMAP access)

#### Environment Configuration

Customize settings in `.env` file:

```bash
# Required: Encryption secret (generate with: openssl rand -hex 32)
EENGINE_SECRET=your-generated-secret-here

# Recommended: Redis password for security
REDIS_PASSWORD=your-redis-password

# Optional: Version pinning (default: latest)
EMAILENGINE_VERSION=latest

# Optional: Custom port bindings (default: 127.0.0.1)
EMAILENGINE_API_BIND=0.0.0.0
EMAILENGINE_API_PORT=3000
EMAILENGINE_SMTP_BIND=0.0.0.0
EMAILENGINE_SMTP_PORT=2525
EMAILENGINE_IMAP_BIND=0.0.0.0
EMAILENGINE_IMAP_PORT=9993

# Optional: Performance tuning
EENGINE_WORKERS=4
EENGINE_LOG_LEVEL=info

# Optional: Restart policy
RESTART_POLICY=unless-stopped
```

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
docker-compose up -d

# View logs
docker-compose logs -f

# View specific service logs
docker-compose logs -f emailengine
docker-compose logs -f redis

# Check status
docker-compose ps
```

### Stop Services

```bash
# Stop containers
docker-compose stop

# Stop and remove containers
docker-compose down

# Stop and remove containers and volumes
docker-compose down -v
```

## Production Deployment

For production deployments, we recommend using the official docker-compose.yml (see "Official Configuration" tab above) with a properly configured `.env` file.

### Production Environment Configuration

Create a production-ready `.env` file:

```bash
# Required: Encryption secret (generate with: openssl rand -hex 32)
EENGINE_SECRET=your-generated-secret-here

# Recommended: Redis password for security
REDIS_PASSWORD=your-secure-redis-password

# Version pinning for stability
EMAILENGINE_VERSION=v2.79.3

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
2. **Pin versions:** Specify exact versions (e.g., `EMAILENGINE_VERSION=v2.79.3`)
3. **Bind to localhost:** If using reverse proxy, keep default `127.0.0.1` binding
4. **Restrict access:** Use firewall rules to limit port access
5. **Enable TLS:** Use reverse proxy (Nginx/Caddy) with HTTPS

### Resource Requirements

The [general figures](/docs/installation#system-requirements) apply here too: 2 GB of memory to evaluate, 4 to 8 GB for production, and more once the account count climbs. Redis in the same compose file competes for the same memory, so size for both.

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
# Create backup
docker-compose exec redis redis-cli SAVE
docker cp emailengine-redis:/data/dump.rdb ./backup-$(date +%Y%m%d).rdb

# Restore backup
docker-compose stop redis
docker cp ./backup-20231201.rdb emailengine-redis:/data/dump.rdb
docker-compose start redis
```

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

If using Redis password, update `.env`:

```bash
EENGINE_REDIS=redis://:your-redis-password@redis:6379
```

### With Reverse Proxy (Nginx)

Add Nginx to `docker-compose.yml`:

```yaml
services:
  # ... redis and emailengine services ...

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

Create `nginx.conf`:

```nginx
events {
    worker_connections 1024;
}

http {
    upstream emailengine {
        server emailengine:3000;
    }

    server {
        listen 80;
        server_name emailengine.example.com;

        location / {
            proxy_pass http://emailengine;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # WebSocket support
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }
}
```

## Docker Images

### Available Tags

Every release publishes four tags, and master publishes one:

| Tag | Points at | Use it when |
|-----|-----------|-------------|
| `v2.79.3` | One exact release | You want the image to change only when you change it. The safest choice for production |
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
docker pull postalsys/emailengine:v2.79.3
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
docker pull postalsys/emailengine:v2.79.3

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

# Test health endpoint from inside container
docker exec emailengine curl -f http://localhost:3000/health
```

## Upgrading

### Docker Compose Upgrade

```bash
# Pull latest tagged release
docker-compose pull

# Restart with new image
docker-compose up -d

# View logs to verify
docker-compose logs -f emailengine
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
docker pull postalsys/emailengine:v2.79.3

# In docker-compose.yml
services:
  emailengine:
    image: postalsys/emailengine:v2.79.3
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
EENGINE_HOST=0.0.0.0          # Bind address

# Logging
EENGINE_LOG_LEVEL=info        # trace, debug, info, warn, error, fatal, silent
EENGINE_LOG_RAW=false         # Log raw IMAP/SMTP traffic

# Metrics available at /metrics endpoint (requires token with 'metrics' scope)

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
