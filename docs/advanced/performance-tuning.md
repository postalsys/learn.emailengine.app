---
title: Performance Tuning
sidebar_position: 1
description: Optimize EmailEngine performance for production workloads with proper configuration and scaling strategies
---

# Performance Tuning

Learn how to optimize EmailEngine for production workloads by tuning worker threads, Redis configuration, and implementing scaling strategies.

## Overview

When you start with EmailEngine and only have a handful of test accounts, a modest server with default configuration is usually enough. As your usage grows, however, you'll want to review both your hardware and your EmailEngine configuration.

### Rule of Thumb

- **Waiting mainly for webhooks?** A smaller server is fine
- **Issuing many API calls?** Provision more CPU/RAM and tune settings

This guide walks through the main configuration options that affect performance and how to pick sensible values.

## IMAP Configuration

### Worker Threads

EmailEngine spawns a fixed pool of worker threads that keep account connections alive and sync mail. Each thread handles a share of your accounts regardless of type - IMAP, Gmail API, and Outlook (Microsoft Graph) accounts all run on these threads.

| Setting | Default | Description |
|---------|---------|-------------|
| `EENGINE_WORKERS` | `4` | Number of account worker threads (IMAP, Gmail API, and Outlook/Graph) |

**How it works**: If you have 100 accounts and `EENGINE_WORKERS=4`, each thread handles ~25 accounts.

**Tuning guideline**: On a machine with many CPU cores (or VPS with several vCPUs), you can safely raise this value so that each core has fewer accounts to juggle.

**Example**:
```bash
# 8-core server with 400 accounts
EENGINE_WORKERS=8  # Each thread handles ~50 accounts
```

### Connection Setup Delay

Opening TCP connections and running IMAP handshakes is CPU-intensive. Doing this for hundreds or thousands of accounts at once can spike CPU usage and even trigger the host's OOM-killer.

**Solution**: Use an artificial delay so EmailEngine brings accounts online one-by-one.

```bash
EENGINE_CONNECTION_SETUP_DELAY=3s   # 3 second delay between connections
```

**Impact calculation**:
- With 3s delay and 1,000 accounts: Full warm-up takes ~50 minutes
- This is perfectly fine if you're only waiting for webhooks
- API requests for an account will fail until that account is connected

**Recommendation**:
- Small deployments (< 100 accounts): 1-2s
- Medium deployments (100-1000 accounts): 3-5s
- Large deployments (> 1000 accounts): 5-10s

### Sub-Connections for Selected Folders

If you need near real-time updates for specific folders (e.g., Inbox and Sent), enable **sub-connections**:

```json
{
  "account": "user@example.com",
  "subconnections": ["\\Sent"]
}
```

**How it works**:
- EmailEngine opens a second TCP connection dedicated to that folder
- Main connection still polls the rest of the mailbox
- Sub-connection fires webhooks instantly for the selected folder
- Saves both CPU and network traffic

**Benefits**:
- Instant notifications for critical folders
- Reduced polling overhead
- Lower latency for important emails

**Considerations**:
- Each sub-connection uses one parallel IMAP session
- Most servers limit parallel connections (typically 3-5)
- Only use for folders you genuinely need instant updates

**Ignored paths**: EmailEngine silently ignores sub-connection entries that:
- Do not exist on the server
- Are already covered by the default connection (INBOX)
- For Gmail accounts: any folder except Trash and Junk, because the All Mail folder that EmailEngine monitors already covers all other mailboxes

:::info Best Effort
Sub-connections are non-blocking. If a sub-connection fails (e.g., server rejects due to too many concurrent connections), EmailEngine continues working normally. The only impact is slightly slower webhook delivery for messages in those folders, as EmailEngine falls back to polling instead of real-time IDLE notifications.
:::

### Limiting Indexed Folders

If you never care about the rest of the mailbox, limit indexing completely:

```json
{
  "account": "user@example.com",
  "path": ["INBOX", "\\Sent"],
  "subconnections": ["\\Sent"]
}
```

**What this does:**
- EmailEngine only **syncs and monitors** the listed folders (INBOX and \Sent)
- Unlisted folders **will not trigger webhooks** when messages change
- You can still **access unlisted folders via API** (list messages, search, send to them)
- Significantly reduces resource usage by limiting active monitoring

**Use case**: Support systems that only need Inbox and Sent Mail.

## API Workers

The API worker runs the REST API and the admin UI. A single worker is enough for most deployments, but very high HTTP request volumes can be spread across several workers.

| Setting | Default | Description |
|---------|---------|-------------|
| `EENGINE_WORKERS_API` | `1` | Number of API/HTTP worker threads |

**Platform support**: Values above `1` require `SO_REUSEPORT` so the workers can share the listen port. This is only available on **Linux with Node.js 23.1 or newer**. On macOS, Windows, or older Node.js, EmailEngine falls back to a single API worker and shows a notice on the **Workers** page (`/admin/internals`).

**Example**:
```bash
# Spread the REST API across 4 workers (Linux + Node.js >= 23.1)
EENGINE_WORKERS_API=4
```

## Webhook Configuration

EmailEngine enqueues every event, even if webhooks are disabled. By default, the queue is processed serially by one worker.

| Setting | Default | Description |
|---------|---------|-------------|
| `EENGINE_WORKERS_WEBHOOKS` | `1` | Number of webhook worker threads |
| `EENGINE_NOTIFY_QC` | `1` | Concurrency per worker |

**Maximum in-flight webhooks**:
```
ACTIVE_WH = EENGINE_WORKERS_WEBHOOKS × EENGINE_NOTIFY_QC
```

**Example configurations**:

```bash
# Configuration 1: Single threaded (default)
EENGINE_WORKERS_WEBHOOKS=1
EENGINE_NOTIFY_QC=1
# Result: 1 webhook at a time

# Configuration 2: Multi-threaded
EENGINE_WORKERS_WEBHOOKS=4
EENGINE_NOTIFY_QC=2
# Result: 8 concurrent webhooks

# Configuration 3: High concurrency
EENGINE_WORKERS_WEBHOOKS=8
EENGINE_NOTIFY_QC=4
# Result: 32 concurrent webhooks
```

**Important**: Ensure your webhook handler can cope with events arriving out-of-order if you raise either value.

### Webhook Handler Best Practices

**Keep the handler tiny**: Ideally it writes the payload to an internal queue (Kafka, SQS, Postgres, etc.) in a few milliseconds and returns `2xx`, leaving the heavy lifting to downstream workers.

**Benefits**:
- Predictable EmailEngine Redis memory usage
- Fast webhook processing
- Better error handling
- Easier to scale processing independently

**Example lightweight handler**:

```javascript
// Endpoint: accept, hand off, return. Nothing slow on this path
app.post('/webhooks', express.json(), async (req, res) => {
  await queue.add('webhook', req.body); // your own queue, not EmailEngine's
  res.sendStatus(200);
});

// Worker: a separate process, scaled independently of the endpoint
worker.process('webhook', async job => {
  await heavyProcessing(job.data);
});
```

Enqueue before responding rather than after. Returning `200` first and then queueing means a crash between the two silently drops the event, and EmailEngine has already been told the delivery succeeded.

## Email Sending Configuration

Queued messages live in Redis, so RAM usage scales with the size and number of messages. Like webhooks, email submissions are handled by a worker pool:

| Setting | Default | Description |
|---------|---------|-------------|
| `EENGINE_WORKERS_SUBMIT` | `1` | Number of submission worker threads |
| `EENGINE_SUBMIT_QC` | `1` | Concurrency per worker |

**Maximum concurrent submissions**:
```
ACTIVE_SUBMIT = EENGINE_WORKERS_SUBMIT × EENGINE_SUBMIT_QC
```

**Example configurations**:

```bash
# Low volume (default)
EENGINE_WORKERS_SUBMIT=1
EENGINE_SUBMIT_QC=1
# Result: 1 email sending at a time

# Medium volume
EENGINE_WORKERS_SUBMIT=2
EENGINE_SUBMIT_QC=2
# Result: 4 concurrent email sends

# High volume
EENGINE_WORKERS_SUBMIT=4
EENGINE_SUBMIT_QC=4
# Result: 16 concurrent email sends
```

**Important**: Be conservative when increasing `EENGINE_SUBMIT_QC`. Each active submission loads the full RFC 822 message into the worker's heap.

**Memory impact**: With average email size of 1MB and `EENGINE_SUBMIT_QC=16`, you need at least 16MB heap just for active submissions.

## Export Configuration

Bulk exports are handled by dedicated worker threads:

| Setting | Default | Description |
|---------|---------|-------------|
| `EENGINE_WORKERS_EXPORT` | `1` | Number of export worker threads |
| `EENGINE_EXPORT_QC` | `1` | Concurrency per worker |

**Maximum concurrent exports**:
```
ACTIVE_EXPORTS = EENGINE_WORKERS_EXPORT x EENGINE_EXPORT_QC
```

This is further limited by the `exportMaxGlobalConcurrent` setting (default: 8) to prevent system overload.

**Example configurations**:

```bash
# Default (1 export at a time)
EENGINE_WORKERS_EXPORT=1
EENGINE_EXPORT_QC=1

# Medium throughput (4 concurrent)
EENGINE_WORKERS_EXPORT=2
EENGINE_EXPORT_QC=2

# High throughput (8 concurrent)
EENGINE_WORKERS_EXPORT=4
EENGINE_EXPORT_QC=2
```

**Memory impact**: Each export batch loads message data into memory. With 4 concurrent exports processing large messages, expect ~400-800 MB additional memory usage.

**Read more**: [Exporting Messages](/docs/receiving/exporting)

## Redis Optimization

Redis is critical for EmailEngine performance. Follow these best practices:

### 1. Minimize Latency

**Keep Redis and EmailEngine in the same availability zone or LAN**:
- Same datacenter: < 1ms latency
- Same region: < 5ms latency
- Cross-region: 50-200ms latency (not recommended)

**Impact**: With 1000 accounts and cross-region Redis, you'll see significant performance degradation.

### 2. Provision Enough RAM

**Aim for < 80% memory usage** in normal operation with 2× headroom for snapshots.

**Storage budget**: Plan for **1-2 MB per account** (more for very large mailboxes).

**Example calculations**:
- 100 accounts: 100-200 MB
- 1,000 accounts: 1-2 GB
- 10,000 accounts: 10-20 GB

**Redis configuration**:
```bash
# redis.conf
maxmemory-policy noeviction  # or volatile-* policy

# Note: It's generally better to leave maxmemory unset and let Redis
# use all available system memory. Only set maxmemory if you need to
# share the server with other applications.
```

### 3. Enable Persistence

**RDB Snapshots**: Enable for data durability
```bash
# redis.conf
save 900 1      # Save if 1 key changes in 15 minutes
save 300 10     # Save if 10 keys change in 5 minutes
save 60 10000   # Save if 10000 keys change in 1 minute
```

**AOF (Append Only File)**: Enable only if you have very fast disks
```bash
# redis.conf
appendonly yes
appendfsync everysec  # Good balance of safety and performance
```

**Recommendation**: Start with RDB only, add AOF if you need better durability guarantees.

### 4. Set Eviction Policy

**Critical**: Never use `allkeys-*` eviction policies. EmailEngine needs all data.

```bash
# redis.conf
maxmemory-policy noeviction  # Recommended
# or
maxmemory-policy volatile-lru  # If you use TTLs
```

**Why**: `allkeys-lru` or `allkeys-random` will evict critical account data, causing failures.

### 5. TCP Keep-Alive

Leave the default value. Setting to `0` (disabling keep-alive) may lead to half-open TCP connections.

```bash
# redis.conf
tcp-keepalive 300  # Default, recommended
```

### Redis-Compatible Alternatives

| Provider/Project | Compatible | Caveats |
|-----------------|------------|---------|
| **Upstash Redis** | Yes | 1 MB command size limit - large attachments cannot be queued. Locate EmailEngine in same GCP/AWS region. |
| **AWS ElastiCache** | Limited | Treats itself as a cache; data loss on restarts. **Not recommended**. |
| **Memurai** | Yes | Tested only in staging. |
| **Dragonfly** | Yes | Start with `--default_lua_flags=allow-undeclared-keys`. |
| **KeyDB** | Yes | Tested only in staging. |

## Complete Configuration Example

Here's a production-ready configuration for a medium deployment (500 accounts):

```bash
# config.env

# Server
EENGINE_HOST=0.0.0.0
EENGINE_PORT=3000

# Redis
EENGINE_REDIS=redis://redis.internal:6379

# Account workers
EENGINE_WORKERS=8                      # 8 worker threads
EENGINE_CONNECTION_SETUP_DELAY=3s      # 3 second startup delay

# Webhook Processing
EENGINE_WORKERS_WEBHOOKS=4             # 4 webhook workers
EENGINE_NOTIFY_QC=2                    # 2 concurrent per worker
                                       # = 8 total concurrent webhooks

# Email Sending
EENGINE_WORKERS_SUBMIT=2               # 2 submission workers
EENGINE_SUBMIT_QC=2                    # 2 concurrent per worker
                                       # = 4 total concurrent sends

# Security
EENGINE_SECRET=your-encryption-secret-here

# Logging
EENGINE_LOG_LEVEL=info

# Monitoring
```

## Scaling EmailEngine

:::warning No Horizontal Scaling
EmailEngine does NOT support horizontal scaling. Running multiple EmailEngine instances that connect to the same Redis will cause each instance to attempt syncing every account independently, leading to conflicts and increased load.

**Kubernetes/Container Orchestration:**
- Set `replicas: 1` in your Deployment - only a single pod can run
- Do NOT configure Horizontal Pod Autoscaler (HPA) for EmailEngine
- Do NOT use auto-scaling groups or similar mechanisms
- If you need more capacity, use vertical scaling (more CPU/RAM) or manual sharding (see below)
:::

### Vertical Scaling (Recommended)

Increase resources on a single EmailEngine instance:

**Hardware:**
- More CPU cores (increase `EENGINE_WORKERS`)
- More RAM (support more concurrent accounts)
- Faster network (reduce latency to Redis and IMAP/SMTP servers)

**Configuration:**
```bash
# Optimize for larger deployments
EENGINE_WORKERS=16                     # Match CPU cores
EENGINE_WORKERS_WEBHOOKS=8
EENGINE_NOTIFY_QC=4
EENGINE_WORKERS_SUBMIT=4
EENGINE_SUBMIT_QC=2
```

**Good for:** Up to several thousand accounts per instance

### Manual Sharding (Advanced Workaround)

If you need to support more accounts than a single instance can handle, you can manually shard accounts across completely independent EmailEngine deployments:

:::danger Manual Sharding Requirements
- Each instance must have its own separate Redis instance
- Each instance manages a different set of accounts
- Your application must route API requests to the correct instance
- No automatic failover or coordination between instances
:::

**Implementation approach:**

```bash
# Instance A - Accounts 0-999
EENGINE_REDIS_PREFIX=ee-shard-a
REDIS_URL=redis://redis-a:6379
EENGINE_PORT=3000
# Service URL must be unique per instance
EENGINE_SETTINGS='{"serviceUrl":"https://ee-a.example.com"}'

# Instance B - Accounts 1000-1999
EENGINE_REDIS_PREFIX=ee-shard-b
REDIS_URL=redis://redis-b:6379
EENGINE_PORT=3001
EENGINE_SETTINGS='{"serviceUrl":"https://ee-b.example.com"}'
```

**OAuth2 Configuration for Sharded Deployments:**

The same OAuth2 application (in Azure AD or Google Cloud Console) can be used across all EmailEngine instances. Each instance needs a unique `serviceUrl`, and all callback URLs must be registered in the OAuth2 app settings.

In your OAuth2 app configuration, add all instance redirect URLs:
```
https://ee-a.example.com/oauth
https://ee-b.example.com/oauth
https://ee-c.example.com/oauth
```

Both Azure AD and Google Cloud Console support multiple redirect URIs per OAuth2 application.

Your application must:
1. Maintain a mapping of which accounts belong to which shard
2. Route all API requests for an account to the correct instance
3. Handle instance failures manually

**Note:** This is complex and error-prone. Vertical scaling is strongly recommended instead.

## Monitoring and Metrics

### Key Metrics to Track

**IMAP Performance**:
- Connection success rate
- Average connection time
- IMAP errors per minute

**Webhook Performance**:
- Webhook queue depth
- Webhook processing time
- Webhook failure rate

**Email Sending**:
- Submission queue depth
- Send success rate
- Send latency

**Redis**:
- Memory usage
- Commands per second
- Latency
- Connection count

### Health Check Endpoint

```bash
curl http://localhost:3000/health

{
  "success": true
}
```

The health endpoint returns a simple success response. For detailed statistics, use the `/v1/stats` endpoint (requires authentication).

### Prometheus Metrics

Metrics are available at `/metrics` endpoint on the main API server.

Create a token with metrics scope:
```bash
emailengine tokens issue -d "Prometheus" -s "metrics"
```

Access metrics:
```bash
curl -H "Authorization: Bearer YOUR_TOKEN" http://localhost:3000/metrics
```

**Read more**: [Monitoring](/docs/advanced/monitoring)

## Performance Troubleshooting

### High CPU Usage

**Possible causes**:
1. **Memory exhaustion** - The most common cause of constant 100% CPU load is running out of RAM. When free memory is depleted, the system frantically tries to manage the few remaining bytes, causing CPU to spike and stay at maximum.
2. Too many accounts for available workers
3. Frequent account reconnections
4. Heavy API request load

**Solutions**:
```bash
# First: Check memory usage
free -h
# or
docker stats

# If memory is exhausted:
# - Add more RAM to the server
# - Reduce number of accounts
# - Scale up Redis memory

# If memory is fine, try:
# Increase workers
EENGINE_WORKERS=16

# Add connection delay
EENGINE_CONNECTION_SETUP_DELAY=5s

# Reduce API call frequency
```

### High Memory Usage

**Possible causes**:
1. Redis memory exhaustion
2. Large email queue
3. Too many concurrent operations

**Solutions**:
```bash
# Reduce submission concurrency
EENGINE_SUBMIT_QC=1

# Scale up Redis
# Reduce retention
```

### Slow Webhook Processing

**Possible causes**:
1. Webhook handler is slow
2. Not enough webhook workers
3. Network issues

**Solutions**:
```bash
# Increase webhook workers
EENGINE_WORKERS_WEBHOOKS=8
EENGINE_NOTIFY_QC=4

# Optimize webhook handler (queue-based)
```

### API Request Timeouts

**Possible causes**:
1. Account not connected yet
2. Redis latency issues
3. IMAP server slow

**Solutions**:
- Wait for account connection before API calls
- Reduce Redis latency
- Check IMAP server performance

