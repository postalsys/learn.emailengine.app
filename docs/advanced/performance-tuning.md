---
title: Performance Tuning
sidebar_position: 1
description: Optimize EmailEngine performance for production workloads with proper configuration and scaling strategies
---

# Performance Tuning

This page covers the settings that decide how much work one EmailEngine instance can do: worker thread counts, connection pacing, which folders get watched, queue concurrency, and the Redis it all depends on.

## Overview

With a handful of test accounts, a modest server on the default configuration is enough. As the account count grows, two questions decide what to tune:

- **Waiting mainly for webhooks?** A smaller server is fine; the cost is mostly idle IMAP connections
- **Issuing many API calls?** Provision more CPU and RAM, because every API call that touches a mailbox runs on the account's worker thread

The worker and queue settings below are read from an environment variable or the configuration file at startup and cannot be changed at runtime; see [Environment variables](/docs/configuration/environment-variables#worker-threads) for the full list. The per-account settings (`subconnections`, `path`, `imapIndexer`) and `exportMaxGlobalConcurrent` are runtime settings changed through the API or the admin UI.

## IMAP Configuration

### Worker Threads

EmailEngine spawns a fixed pool of account worker threads that keep connections alive and sync mail. Each thread handles a share of the accounts regardless of type: IMAP, Gmail API and Microsoft Graph accounts all run on these threads.

| Setting | Default | Description |
|---------|---------|-------------|
| `EENGINE_WORKERS` | `4` | Number of account worker threads. The value `cpus` means one thread per CPU core |

With 100 accounts and `EENGINE_WORKERS=4`, each thread handles about 25 accounts. On a machine with several cores, raise the count so that each core has fewer accounts to juggle:

```bash
# 8-core server with 400 accounts: about 50 accounts per thread
EENGINE_WORKERS=8
```

### Connection Setup Delay

Opening TCP connections and running IMAP handshakes is CPU-intensive. Doing this for hundreds or thousands of accounts at once after a restart spikes CPU and can push the host into swapping.

`EENGINE_CONNECTION_SETUP_DELAY` inserts a pause between assigning one account to a worker and the next. The default is `0`; the value accepts a duration string:

```bash
EENGINE_CONNECTION_SETUP_DELAY=3s
```

The trade-off is warm-up time. With a 3 second delay and 1,000 accounts, the last account comes online about 50 minutes after startup. That is fine if you are only waiting for webhooks, but API requests for an account fail until that account is connected.

Starting points:

- Under 100 accounts: 1-2s
- 100 to 1,000 accounts: 3-5s
- Over 1,000 accounts: 5-10s

### Sub-Connections for Selected Folders

By default one IMAP connection per account sits in `IDLE` on INBOX (on "All Mail" for Gmail) and every other folder is polled. To get push notifications for another folder, list it in the account's `subconnections`:

```json
{
  "subconnections": ["\\Sent"]
}
```

Each entry is a folder path or a special-use flag (`\\Sent`, `\\Junk`, `\\Trash`, `\\Drafts`). For each one EmailEngine opens an additional IMAP session dedicated to that folder; the main connection keeps polling the rest of the mailbox.

**What you get:**
- Webhooks for the selected folder fire as soon as the server reports the change
- The folder is no longer waiting on the polling interval

**What it costs:**
- One more concurrent IMAP session per entry, against the provider's per-account connection limit
- A `subconnections` change reconnects the account

**Entries EmailEngine leaves disabled:** the account page (**Accounts** > the account > **Subconnections**) lists every configured path with a state badge. A disabled entry shows **Disabled**, and hovering over the badge shows the reason:

| Reason shown | When |
| ------------ | ---- |
| `Mailbox folder not found` | The path or special-use flag matched nothing in the folder listing. The entry is retried if the folder appears later |
| `Covered by the primary connection` | The path is the first entry of the account's `path` setting, which the main connection already watches |
| `Covered by the "All Mail" folder` | Gmail account watching all folders: only `\\Trash` and `\\Junk` are accepted, because "All Mail" already covers everything else |
| `Can not use the default folder` | Non-Gmail account watching all folders: INBOX is already the main connection's folder |

:::info Best Effort
Sub-connections are non-blocking. If a sub-connection fails, for instance because the server rejects one more concurrent session, the account keeps working on its primary connection and the folder falls back to polling.
:::

### Limiting Indexed Folders

If you never need the rest of the mailbox, limit what EmailEngine syncs with the account's `path` setting:

```json
{
  "path": ["INBOX", "\\Sent"],
  "subconnections": ["\\Sent"]
}
```

**What this does:**
- EmailEngine syncs and watches only the listed folders
- Changes in unlisted folders produce no webhooks
- Unlisted folders stay fully reachable through the API: listing, searching, fetching and moving messages all work
- The account keeps far fewer folders open and indexed, which is the main cost of a large mailbox

The default is `["*"]`, every folder. A support desk that only needs INBOX and Sent Mail is the typical case for narrowing it.

For a large mailbox where you do need every folder, the [`imapIndexer`](/docs/accounts/imap-indexers) setting is the other lever: the `fast` indexer skips tracking of flag changes and deletions and is much cheaper than `full`.

## API Workers

The API worker runs the REST API and the admin UI. One worker is enough for most deployments; a very high HTTP request volume can be spread across several.

| Setting | Default | Description |
|---------|---------|-------------|
| `EENGINE_WORKERS_API` | `1` | Number of API/HTTP worker threads. Accepts `cpus` like `EENGINE_WORKERS` |

**Platform support:** more than one API worker requires `SO_REUSEPORT` so the workers can share the listen port. That needs **Linux with Node.js 23.1 or newer**. Elsewhere EmailEngine starts a single API worker, logs why, and shows the reason on the **Workers** page (`/admin/internals`). Added in v2.69.0.

```bash
# Spread the REST API across 4 workers (Linux, Node.js 23.1+)
EENGINE_WORKERS_API=4
```

## Webhook Configuration

EmailEngine queues every event, whether or not webhooks are enabled; the webhook worker decides at delivery time whether there is anywhere to send it. By default one worker processes the queue one job at a time.

| Setting | Default | Description |
|---------|---------|-------------|
| `EENGINE_WORKERS_WEBHOOKS` | `1` | Number of webhook worker threads |
| `EENGINE_NOTIFY_QC` | `1` | Jobs each worker processes concurrently |

The number of webhook requests in flight at once is the product of the two:

```text
in-flight webhooks = EENGINE_WORKERS_WEBHOOKS x EENGINE_NOTIFY_QC
```

```bash
# Default: one delivery at a time
EENGINE_WORKERS_WEBHOOKS=1
EENGINE_NOTIFY_QC=1

# 8 concurrent deliveries
EENGINE_WORKERS_WEBHOOKS=4
EENGINE_NOTIFY_QC=2

# 32 concurrent deliveries
EENGINE_WORKERS_WEBHOOKS=8
EENGINE_NOTIFY_QC=4
```

Raising either value means events can reach your endpoint out of order. Make sure the handler copes with a `messageUpdated` arriving before the `messageNew` for the same message.

### Webhook Handler Best Practices

Keep the handler tiny: write the payload to a queue of your own (Kafka, SQS, Postgres, a job table) in a few milliseconds and return `2xx`, leaving the heavy lifting to downstream workers. That keeps EmailEngine's Redis usage predictable, because a slow endpoint shows up as a growing `notify` queue in Redis memory.

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

Queued messages live in Redis, so RAM usage scales with the size and number of messages waiting. Like webhooks, submissions are handled by a worker pool:

| Setting | Default | Description |
|---------|---------|-------------|
| `EENGINE_WORKERS_SUBMIT` | `1` | Number of submission worker threads |
| `EENGINE_SUBMIT_QC` | `1` | Messages each worker sends concurrently |

```text
concurrent sends = EENGINE_WORKERS_SUBMIT x EENGINE_SUBMIT_QC
```

```bash
# Default: one message at a time
EENGINE_WORKERS_SUBMIT=1
EENGINE_SUBMIT_QC=1

# 4 concurrent sends
EENGINE_WORKERS_SUBMIT=2
EENGINE_SUBMIT_QC=2

# 16 concurrent sends
EENGINE_WORKERS_SUBMIT=4
EENGINE_SUBMIT_QC=4
```

Be conservative with `EENGINE_SUBMIT_QC`. Each active submission loads the full RFC 822 message into the worker's heap, so 16 concurrent 10 MB messages need 160 MB of heap in that worker before anything else.

## Export Configuration

Bulk exports run on their own worker threads:

| Setting | Default | Description |
|---------|---------|-------------|
| `--workers.export` | `1` | Number of export worker threads. Set on the command line or as `export` under `[workers]` in the configuration file. The Workers page lists this value under the name `EENGINE_WORKERS_EXPORT`, but as of v2.79.4 that environment variable is not read; use the command-line flag or the configuration file |
| `EENGINE_EXPORT_QC` | `1` | Export jobs each worker runs concurrently |

```text
concurrent exports = export workers x EENGINE_EXPORT_QC
```

The runtime setting `exportMaxGlobalConcurrent` (default `8`) caps the number of exports running at once across the whole instance, whatever the worker configuration allows.

```bash
# Default: one export at a time
EENGINE_EXPORT_QC=1 emailengine --workers.export=1

# 4 concurrent exports
EENGINE_EXPORT_QC=2 emailengine --workers.export=2
```

Each export batch holds message data in memory while it is written, so raise the concurrency together with the worker's memory. See [Exporting messages](/docs/receiving/exporting).

## Redis

Redis is EmailEngine's only data store, and its latency is added to nearly every operation. [Redis](/docs/configuration/redis) covers sizing, persistence and compatible servers in full; the points that decide performance are:

1. **Keep Redis close.** Same host or same LAN. Cross-region Redis puts tens of milliseconds on every mailbox operation and is not a workable setup
2. **Budget 1-2 MiB per account** and provision twice that, so snapshots and growth have room
3. **Never use an `allkeys-*` eviction policy.** EmailEngine needs everything it stores. `noeviction` is the setting to use
4. **Leave `tcp-keepalive` at its default.** Setting it to `0` leads to half-open connections after network hiccups
5. **Watch `redis_evicted_keys_total` and `redis_memory_used_bytes`** in the [metrics](/docs/advanced/monitoring#redis-metrics); evictions mean data loss

## Complete Configuration Example

A configuration for a medium deployment of around 500 accounts:

```bash
# Listener
EENGINE_HOST=0.0.0.0
EENGINE_PORT=3000

# Redis
EENGINE_REDIS=redis://redis.internal:6379

# Account workers: 8 threads, paced startup
EENGINE_WORKERS=8
EENGINE_CONNECTION_SETUP_DELAY=3s

# Webhooks: 4 workers x 2 = 8 concurrent deliveries
EENGINE_WORKERS_WEBHOOKS=4
EENGINE_NOTIFY_QC=2

# Sending: 2 workers x 2 = 4 concurrent sends
EENGINE_WORKERS_SUBMIT=2
EENGINE_SUBMIT_QC=2

# Credential encryption
EENGINE_SECRET=your-encryption-secret-here

# Logging
EENGINE_LOG_LEVEL=info
```

## Scaling EmailEngine

:::warning No Horizontal Scaling
EmailEngine does not support horizontal scaling. Two instances pointed at the same Redis database each try to sync every account, and the result is duplicate connections, duplicate webhooks and conflicting state.

In Kubernetes or any other orchestrator, run exactly one replica and do not attach a Horizontal Pod Autoscaler or an auto-scaling group. For more capacity, scale vertically or shard manually as described below.
:::

### Vertical Scaling (Recommended)

Give the single instance more resources:

- More CPU cores, and raise `EENGINE_WORKERS` to match
- More RAM, for more concurrent accounts and larger queues
- A faster network path to Redis and to the mail servers

```bash
EENGINE_WORKERS=16
EENGINE_WORKERS_WEBHOOKS=8
EENGINE_NOTIFY_QC=4
EENGINE_WORKERS_SUBMIT=4
EENGINE_SUBMIT_QC=2
```

One instance configured this way handles several thousand accounts.

### Manual Sharding (Advanced Workaround)

Beyond what one instance can carry, split the accounts across fully independent EmailEngine deployments:

:::danger Manual Sharding Requirements
- Each instance has its own Redis database
- Each instance manages a disjoint set of accounts
- Your application routes every API request to the instance that owns the account
- There is no failover or coordination between instances
:::

```bash
# Instance A
EENGINE_REDIS=redis://redis-a:6379
EENGINE_PORT=3000
EENGINE_SETTINGS='{"serviceUrl":"https://ee-a.example.com"}'

# Instance B
EENGINE_REDIS=redis://redis-b:6379
EENGINE_PORT=3001
EENGINE_SETTINGS='{"serviceUrl":"https://ee-b.example.com"}'
```

`serviceUrl` must be unique per instance, because it is the base of the OAuth2 callback URL and of tracking links. `EENGINE_REDIS_PREFIX` is only needed if two instances have to share one Redis database; with separate databases it can stay unset.

**OAuth2 for sharded deployments:** the same OAuth2 application in Google Cloud Console or Microsoft Entra can serve every instance. Register each instance's callback URL on the application:

```text
https://ee-a.example.com/oauth
https://ee-b.example.com/oauth
```

Your application must keep the account-to-shard mapping, route every request accordingly, and handle an instance outage itself. This is considerably more to operate than one larger instance; scale vertically first.

## Monitoring and Metrics

The figures to watch while tuning, all available from the [Prometheus endpoint](/docs/advanced/monitoring#prometheus-metrics):

- **Accounts by state** - `imap_connections`, to see how long warm-up takes and whether accounts drop
- **Webhook queue depth and request time** - `queue_size{queue="notify"}` and `webhook_req`, to size the webhook workers
- **Submission queue depth** - `queue_size{queue="submit"}`, to size the submit workers
- **Redis memory, latency and evictions** - `redis_memory_used_bytes`, `redis_latency`, `redis_evicted_keys_total`

`GET /health` answers without authentication once every account worker is up; `GET /v1/stats` returns the same state counts and queue depths as JSON. Both are described on [Monitoring](/docs/advanced/monitoring#health-check-endpoints).

## Performance Troubleshooting

### High CPU Usage

**Possible causes:**
1. **Memory exhaustion.** The most common cause of a constant 100% CPU load is running out of RAM: once free memory is gone, the host spends its time swapping. Check `free -h` or `docker stats` before anything else
2. Too many accounts per worker thread
3. Accounts reconnecting in a loop
4. Heavy API request load

**Solutions:**
- Add RAM, or reduce the account count on this instance
- Raise `EENGINE_WORKERS`
- Add `EENGINE_CONNECTION_SETUP_DELAY` so restarts do not connect everything at once
- Find looping accounts with `imap_connections{status="connecting"}` and the per-account log

### High Memory Usage

**Possible causes:**
1. Redis memory growth from a queue that is not draining
2. A large submission queue holding message bodies
3. Too many concurrent submissions or exports

**Solutions:**
- Lower `EENGINE_SUBMIT_QC` and `EENGINE_EXPORT_QC`
- Fix or speed up the webhook endpoint so the `notify` queue drains
- Give Redis more memory

### Slow Webhook Processing

**Possible causes:**
1. The webhook endpoint is slow
2. Not enough webhook workers for the event rate
3. Network problems between EmailEngine and the endpoint

**Solutions:**
- Make the handler enqueue and return, as shown above
- Raise `EENGINE_WORKERS_WEBHOOKS` and `EENGINE_NOTIFY_QC`
- Check `webhook_req` for where the time goes

### API Request Timeouts

**Possible causes:**
1. The account is not connected yet
2. Redis latency
3. A slow IMAP server

**Solutions:**
- Check the account state before issuing mailbox requests after a restart
- Move Redis closer
- Look at the account's [per-account log](/docs/advanced/logging#per-account-logs) for slow IMAP commands

## See Also

- [Monitoring](/docs/advanced/monitoring) - Measuring before and after a change
- [Redis](/docs/configuration/redis) - Memory, persistence, and capacity planning
- [Queue management](/docs/advanced/queue-management) - Concurrency for webhook and submission workers
- [IMAP indexers](/docs/accounts/imap-indexers) - The cheapest lever on a large account count
- [Environment variables](/docs/configuration/environment-variables#worker-threads) - Where the worker counts are set
