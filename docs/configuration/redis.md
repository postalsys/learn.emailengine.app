---
title: Redis Configuration
description: Redis configuration requirements, connection settings, and performance tuning
sidebar_position: 3
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Redis Configuration

EmailEngine requires Redis as its primary data store for mailbox indexes, OAuth credentials, job queues, and webhook events. This guide covers how to configure Redis for EmailEngine and optimize its performance.

## Connecting EmailEngine to Redis

### Connection String Format

EmailEngine connects to Redis using a connection string specified via the `EENGINE_REDIS` environment variable or `--dbs.redis` command-line argument.

**Format:**
```
redis://[[username:]password@]host[:port][/database]
```

**Examples:**

```bash
# Local Redis (default port 6379, default database 8)
EENGINE_REDIS="redis://localhost:6379/8"

# Remote Redis with password
EENGINE_REDIS="redis://:mypassword@redis.example.com:6379"

# Redis with username and password (Redis 6+)
EENGINE_REDIS="redis://admin:mypassword@redis.example.com:6379"

# Redis over TLS
EENGINE_REDIS="rediss://redis.example.com:6380"
```

### Deployment Examples

<Tabs groupId="deployment-platform">
<TabItem value="docker" label="Docker Compose">

```yaml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    command: redis-server --save 60 1000 --save 300 10 --save 900 1 --maxmemory-policy noeviction
    volumes:
      - redis-data:/data
    ports:
      - "6379:6379"
    restart: unless-stopped

  emailengine:
    image: postalsys/emailengine:v2
    environment:
      - EENGINE_REDIS=redis://redis:6379
      - EENGINE_HOST=0.0.0.0
      - EENGINE_PORT=3000
    ports:
      - "3000:3000"
    depends_on:
      - redis
    restart: unless-stopped

volumes:
  redis-data:
```

</TabItem>
<TabItem value="kubernetes" label="Kubernetes">

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: emailengine-config
data:
  EENGINE_REDIS: "redis://redis-service:6379"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: emailengine
spec:
  replicas: 1
  selector:
    matchLabels:
      app: emailengine
  template:
    metadata:
      labels:
        app: emailengine
    spec:
      containers:
      - name: emailengine
        image: postalsys/emailengine:v2
        envFrom:
        - configMapRef:
            name: emailengine-config
        ports:
        - containerPort: 3000
```

</TabItem>
<TabItem value="env" label="Environment Variable">

```bash
# .env file
EENGINE_REDIS="redis://localhost:6379/8"

# Or command line
export EENGINE_REDIS="redis://localhost:6379/8"
emailengine
```

</TabItem>
<TabItem value="cli" label="CLI Argument">

```bash
# Start with CLI argument
emailengine --dbs.redis="redis://localhost:6379/8"

# With password
emailengine --dbs.redis="redis://:mypassword@redis.example.com:6379"
```

</TabItem>
</Tabs>

### Connection Options

For advanced Redis configurations, you can pass connection options as query parameters:

```bash
# Connect with TLS
EENGINE_REDIS="rediss://redis.example.com:6380"

# Specify IPv4 or IPv6
EENGINE_REDIS="redis://redis.example.com:6379?family=4"  # Force IPv4
EENGINE_REDIS="redis://redis.example.com:6379?family=6"  # Force IPv6
```

## Required Redis Configuration

EmailEngine requires specific Redis settings to function correctly.

### Memory Eviction Policy (Required)

EmailEngine requires that all keys remain in memory. Set the eviction policy to `noeviction`:

```ini
maxmemory-policy noeviction
```

**Why this matters:** If Redis evicts mailbox indexes or OAuth tokens, EmailEngine must resynchronize entire mailboxes, which is expensive and time-consuming.

**Verification:**
```bash
redis-cli CONFIG GET maxmemory-policy
```

**Set at runtime:**
```bash
redis-cli CONFIG SET maxmemory-policy noeviction
```

**No alternatives:** `noeviction` is the only supported policy. EmailEngine raises a danger-level warning on the dashboard if any other `maxmemory-policy` is configured, and a separate data-loss warning if Redis has actually evicted keys. Do not use `volatile-*` or `allkeys-*` policies - evicted keys mean lost mailbox indexes and credentials.

### Persistence Configuration (Recommended)

Enable persistence to prevent data loss on Redis restarts.

<Tabs groupId="redis-persistence">
<TabItem value="rdb" label="RDB Snapshots (Recommended)">

```ini
# Save after 1 change in 15 minutes
save 900 1

# Save after 10 changes in 5 minutes
save 300 10

# Save after 10000 changes in 1 minute
save 60 10000
```

**Why recommended:** RDB creates periodic snapshots with minimal performance impact. Best for EmailEngine's write-heavy workload.

**Verification:**
```bash
redis-cli CONFIG GET save
```

</TabItem>
<TabItem value="aof" label="Append-Only File (Use with Caution)">

```ini
appendonly yes
appendfsync everysec
```

:::danger AOF and EmailEngine's write pattern
An initial sync writes one index entry per message, so a large mailbox produces a burst of small writes. With AOF every one of them is appended to disk, and the rewrite that periodically compacts the log competes with the same burst.

What that costs depends entirely on the storage underneath. On a slow disk it shows up as Redis latency, a growing AOF, and a sync that never seems to finish; on fast local NVMe it may not be noticeable at all. The failure is gradual rather than sudden, which is what makes it worth deciding deliberately.

The [official docker-compose.yml](/docs/installation/docker) uses RDB snapshots and no AOF, and that is the configuration this page recommends. If you turn AOF on, watch `latency_percentiles_usec` for write commands and the AOF rewrite duration in `INFO persistence` under a real initial sync before calling it good.
:::

**Verification:**
```bash
redis-cli CONFIG GET appendonly
redis-cli CONFIG GET appendfsync
redis-cli INFO persistence | grep aof_rewrite_in_progress
```

</TabItem>
<TabItem value="both" label="Both (Not Recommended)">

```ini
# RDB snapshots
save 900 1
save 300 10
save 60 10000

# AOF
appendonly yes
appendfsync everysec
```

:::warning Performance Impact
Using both RDB and AOF provides maximum durability but doubles the I/O overhead.

**Not recommended for EmailEngine** due to the extremely high write volume. The AOF overhead will likely cause performance issues.

Use RDB only unless you have enterprise-grade storage (NVMe SSDs with 20,000+ IOPS).
:::

**Verification:**
```bash
redis-cli CONFIG GET save
redis-cli CONFIG GET appendonly
```

</TabItem>
</Tabs>

### TCP Keep-Alive (Recommended)

Configure TCP keep-alive to maintain long-lived Pub/Sub connections:

```ini
tcp-keepalive 300
```

**Why this matters:** EmailEngine uses Redis Pub/Sub for real-time updates. Without keep-alive, NAT devices or load balancers may silently drop idle connections.

## Redis Configuration File Example

Create a `redis.conf` file with EmailEngine-optimized settings:

```ini
# Bind to all interfaces (adjust for security)
bind 0.0.0.0

# Port
port 6379

# Memory limit (adjust based on your needs)
maxmemory 2gb

# Eviction policy - REQUIRED for EmailEngine
maxmemory-policy noeviction

# Persistence - RDB snapshots (RECOMMENDED)
save 900 1
save 300 10
save 60 10000

# Persistence - AOF (NOT RECOMMENDED for EmailEngine)
# Only enable if you have high-performance storage (20,000+ IOPS)
# and understand the performance impact
appendonly no

# TCP keep-alive
tcp-keepalive 300

# Log level
loglevel notice

# Log file
logfile /var/log/redis/redis.log

# Working directory
dir /var/lib/redis
```

**Start Redis with config file:**
```bash
redis-server /etc/redis/redis.conf
```

## Verifying Connection

### Test Redis Directly

```bash
# Test connectivity
redis-cli -h localhost -p 6379 ping
# Expected: PONG
```

### Check EmailEngine Logs

EmailEngine uses JSON logging (pino). Log levels: `60`=FATAL, `50`=ERROR, `40`=WARN, `30`=INFO, `20`=DEBUG, `10`=TRACE.

**Successful connection:**
```text
{"level":30,"time":1762176419767,"pid":93728,"msg":"EmailEngine starting up","version":"2.79.3"}
{"level":20,"time":1762176421071,"pid":93728,"msg":"Started API server thread","port":3000,"host":"127.0.0.1"}
```

No explicit "Redis connected" message. If "Started API server thread" appears (at debug level), Redis is connected.

**Connection failure:**
```
============================================================================================================
Failed to establish connection to Redis using "redis://127.0.0.1:16379"
Can not connect to the database. Redis might not be running.
============================================================================================================
```

**Common errors:**

Connection refused:
```json
{"level":60,"time":1762176637410,"pid":2625,"msg":"EmailEngine starting up"}
```

Invalid hostname:
```text
{"level":60,"msg":false,"err":{"message":"getaddrinfo ENOTFOUND invalid-host","code":"ENOTFOUND"}}
{"level":10,"msg":"Connection retry","times":1,"delay":1000}
```

**Pretty format (development):**
```bash
emailengine | pino-pretty
```

## Data Stored in Redis

| Data Type | Typical Size |
|-----------|--------------|
| Mailbox indexes | 1-2 MiB/account |
| OAuth tokens | ~1 KiB/account |
| Webhook queue | Varies |
| Outbox queue | Varies |
| Account metadata | ~2 KiB/account |
| Web sessions | ~500 bytes/session |

**Check memory usage:**
```bash
redis-cli INFO memory | grep used_memory_human
```

## Capacity Planning

### Memory Sizing

Allocate **1-2 MiB of RAM per account** and provision **twice the calculated baseline** to accommodate:
- Copy-on-write memory during RDB snapshots
- Webhook and outbox queue bursts
- Keep usage below **80%** of provisioned memory

**Example:**

| Accounts | Base RAM | Provision | Target Usage |
|----------|----------|-----------|--------------|
| 100 | 100-200 MiB | 400 MiB | < 320 MiB |
| 1,000 | 1-2 GiB | 4 GiB | < 3.2 GiB |
| 10,000 | 10-20 GiB | 40 GiB | < 32 GiB |

### Network Latency

Deploy Redis and EmailEngine in the same availability zone. Target RTT < 5ms (ideally < 1ms).

**Measure latency:**
```bash
redis-cli --latency --raw -h <redis-host>
```

### Backups

Back up `dump.rdb` regularly:

```bash
#!/bin/bash
set -euo pipefail
BACKUP_DIR="/backup/redis"
DATE=$(date +%Y%m%d_%H%M%S)

# Remember the previous save so we can tell when this one finished
PREV=$(redis-cli LASTSAVE)
redis-cli BGSAVE
while [ "$(redis-cli LASTSAVE)" = "$PREV" ]; do sleep 1; done

cp /var/lib/redis/dump.rdb "$BACKUP_DIR/dump-$DATE.rdb"
find "$BACKUP_DIR" -name "dump-*.rdb" -mtime +7 -delete
```

## Performance Tuning

### Write Pattern

EmailEngine writes one small index entry per message, so the load is dominated by initial syncs: a mailbox with 100,000 messages is 100,000 writes, and several accounts syncing at once multiply that. Steady-state traffic afterwards is a fraction of it.

This is why [persistence choice matters](#persistence-configuration-recommended) here more than it would for a cache, and why RDB snapshots are the recommended setting. Snapshot cost is proportional to dataset size and paid periodically; AOF cost is proportional to write count and paid continuously.

### Memory Management

**Monitor fragmentation:**
```bash
redis-cli INFO memory | grep mem_fragmentation_ratio
```

Target: 1.0-1.5. If > 1.5, restart Redis to defragment:
```bash
redis-cli SHUTDOWN SAVE && redis-server /etc/redis/redis.conf
```

### Connection Health

Normal: 2-5 connections per EmailEngine instance.

```bash
redis-cli CLIENT LIST | wc -l
```

## Managed Redis Services

### Compatibility Matrix

| Service | Status | Notes |
|---------|--------|-------|
| **Upstash Redis** | Supported with constraints | 1 MB command size limit affects large MIME blobs; free tier quotas are insufficient; must be located in the same cloud region |
| **Amazon ElastiCache** | Not supported | EmailEngine declares ElastiCache incompatible and shows a danger-level dashboard warning - using it as the database backend can result in data loss |
| **Azure Cache for Redis** | Supported | Use Basic, Standard, or Premium tier; verify persistence is enabled |
| **Google Cloud Memorystore** | Supported | Use Standard tier with replication for high availability |
| **Redis Cloud** | Supported | Native Redis service; ensure persistence and eviction policy are configured |
| **Memurai** | Experimental | Passes basic tests on Windows; no long-term performance data |
| **Dragonfly** | Experimental | Requires `--default_lua_flags=allow-undeclared-keys`; validate against production workloads |
| **KeyDB** | Experimental | Multi-threaded fork of Redis; monitor replication lag and memory stability |

**Unsupported:** Redis Cluster is incompatible with EmailEngine's Lua scripts that reference dynamic keys. Use a single Redis instance with persistence enabled.

### Service-Specific Configuration

<Tabs groupId="redis-service">
<TabItem value="upstash" label="Upstash Redis">

```bash
# Upstash connection string format
EENGINE_REDIS="rediss://:YOUR_PASSWORD@YOUR_ENDPOINT.upstash.io:6379"
```

**Limitations:**
- 1 MB command size limit (affects large attachments in outbox)
- Free tier: 10,000 commands/day is insufficient for production
- Requires same-region deployment to minimize latency

</TabItem>
<TabItem value="elasticache" label="Amazon ElastiCache">

:::danger Not supported
EmailEngine is incompatible with Amazon ElastiCache as the database backend - using it can result in data loss, and EmailEngine displays a danger-level warning on the dashboard when ElastiCache is detected. Migrate to a standard Redis deployment (self-managed Redis or a compatible managed service) instead.
:::

</TabItem>
<TabItem value="azure" label="Azure Cache for Redis">

```bash
# Azure Redis connection string
EENGINE_REDIS="rediss://:YOUR_ACCESS_KEY@your-cache.redis.cache.windows.net:6380"
```

**Recommended tier:** Standard or Premium (not Basic, which lacks persistence)

**Configuration:**
- Enable data persistence (RDB or AOF)
- Set eviction policy to `noeviction`
- Enable TLS (port 6380)
- Configure firewall rules

</TabItem>
<TabItem value="gcp" label="Google Cloud Memorystore">

```bash
# Memorystore connection string
EENGINE_REDIS="redis://10.0.0.3:6379"
```

**Recommended tier:** Standard tier with replication

**Configuration:**
- Use Standard tier (not Basic)
- Enable high availability
- Configure VPC peering with EmailEngine
- Set eviction policy to `noeviction`

</TabItem>
<TabItem value="redis-cloud" label="Redis Cloud">

```bash
# Redis Cloud connection string
EENGINE_REDIS="redis://:YOUR_PASSWORD@redis-12345.c1.region.cloud.redislabs.com:12345"
```

**Configuration:**
- Ensure persistence is enabled
- Set eviction policy to `noeviction`
- Enable TLS if required
- Monitor memory usage

</TabItem>
</Tabs>

## Monitoring

### Key Metrics

**Memory:**
```bash
redis-cli INFO memory | grep used_memory_human
```

**Latency:**
```bash
redis-cli --latency-history -i 1
```

**Persistence:**
```bash
redis-cli INFO persistence | grep rdb_last_save_time
```

### Alert Thresholds

| Metric | Warning | Critical |
|--------|---------|----------|
| Memory usage | > 70% | > 85% |
| Memory fragmentation | > 1.5 | > 2.0 |
| Latency (p99) | > 10ms | > 50ms |
| Persistence lag | > 60s | > 300s |

### EmailEngine Prometheus Metrics

```bash
curl http://localhost:3000/metrics | grep redis
```

Example output:
```
redis_version{version="v7.2.7"} 1
redis_uptime_in_seconds 369345
redis_latency 103542
redis_connected_clients 34
redis_memory_used_bytes 279341568
redis_memory_max_bytes 17179869184
redis_mem_fragmentation_ratio 1.06
redis_instantaneous_ops_per_sec 597
redis_last_save_time 1762178720
```

## Quick Reference

Deploy Redis in the same data center as EmailEngine, allocate sufficient memory, enable RDB persistence, and set eviction policy to `noeviction`.

**Essential checklist:**
- Set `maxmemory-policy noeviction`
- Enable RDB persistence (`save 60 10000 300 10 900 1`)
- Set `tcp-keepalive 300`
- Provision 2× base memory (1-2 MiB per account)
- Keep usage < 80%
- Target latency < 5ms
- Avoid AOF (too high I/O overhead)
- Avoid Redis Cluster (not supported)

**Connection strings:**
```bash
redis://localhost:6379              # Local
redis://:password@host:6379         # With password
rediss://:password@host:6380        # With TLS
redis://host:6379/8                 # Specific database
```

## See Also

- [Configuration overview](/docs/configuration) - Where the Redis URL fits among the other settings
- [Environment variables](/docs/configuration/environment-variables#redis) - `EENGINE_REDIS`, the prefix, and the connection family
- [Performance tuning](/docs/advanced/performance-tuning) - What drives the memory figures above
- [Monitoring](/docs/advanced/monitoring) - Redis metrics worth alerting on
- [Docker installation](/docs/installation/docker) - The shipped compose file and its Redis settings

