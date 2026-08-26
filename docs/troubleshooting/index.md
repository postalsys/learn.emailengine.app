---
title: Troubleshooting Guide
description: Troubleshoot common issues with accounts, webhooks, and email delivery
sidebar_position: 1
---

# Troubleshooting EmailEngine

What to check when EmailEngine, an account, a webhook, or a submission is not behaving. The API calls below use `https://emailengine.example.com` for the instance and `TOKEN` for an access token; replace both.

:::tip Quick Diagnostic

1. Check the health endpoint: `curl https://emailengine.example.com/health`
2. Check the logs: `journalctl -u emailengine -n 100` or `docker logs emailengine`
3. Check Redis: `redis-cli ping`
4. Open the account in the admin interface and read its state and last error

:::

## Quick Diagnostic Checklist

Use this checklist for initial troubleshooting:

- [ ] EmailEngine service is running
- [ ] Redis is running and accessible
- [ ] Redis has available memory
- [ ] Network connectivity to IMAP/SMTP servers
- [ ] Firewall not blocking required ports
- [ ] Valid credentials for email accounts
- [ ] OAuth2 tokens not expired
- [ ] Sufficient disk space
- [ ] System has adequate RAM
- [ ] No conflicting processes on ports

## Common Issues by Category

### Connection Issues

#### EmailEngine Won't Start

**Symptom:** Service fails to start or exits immediately

**Diagnostic steps:**

```bash
# Check if process is running
ps aux | grep emailengine

# Check logs
journalctl -u emailengine -n 50
# Or for Docker
docker logs emailengine

# Check the installed version
emailengine version
```

**Common causes and solutions:**

1. **Redis connection failed**

   A Redis problem at startup is printed to the console in a boxed message rather than only logged, and the process exits. The wording names the cause:

   - `Can not connect to the database. Redis might not be running. Are you using correct hostname and port values?` for `ECONNREFUSED`
   - `Connection to the database timed out. Seems like you are firewalled. Are you using correct hostname and port values?` for `ETIMEDOUT`
   - `Redis password is required but not provided` or `Redis requires a valid password` when the server answered `NOAUTH`
   - `Provided Redis password was not accepted` when it answered `WRONGPASS`

   Anything else is logged as `Redis connection error` with the underlying error attached; a hostname that does not resolve shows up there as `ENOTFOUND`.

   **Solution:**

   ```bash
   # Check Redis status
   sudo systemctl status redis
   sudo systemctl start redis

   # Test connection
   redis-cli ping

   # Verify the URL EmailEngine uses
   echo $EENGINE_REDIS
   # Should be: redis://localhost:6379
   ```

2. **Port already in use**

   The API listener binds to `127.0.0.1:3000` by default, and a bind failure surfaces as an `EADDRINUSE` error naming that address:

   ```
   Error: listen EADDRINUSE: address already in use 127.0.0.1:3000
   ```

   **Solution:**

   ```bash
   # Find process using port
   sudo lsof -i :3000
   sudo netstat -tulpn | grep :3000

   # Kill process or use different port
   export EENGINE_PORT=3001
   ```

   `EENGINE_HOST` changes the bind address the same way. The built-in SMTP server, which starts on `127.0.0.1:2525`, and the IMAP proxy are separate listeners with their own port settings.

3. **Node.js version too old**

   EmailEngine declares Node.js 20 or newer in its `engines` field. The startup guard is older than that requirement and only refuses to run below Node.js 17, exiting immediately with `Node.js version vX.Y.Z is not supported. Please upgrade to Node.js 17 or later.`. A version between 17 and 19 starts but is untested, so treat a strange failure on one as a version problem.

   **Solution:**

   ```bash
   # Check version
   node --version

   # Update Node.js (using nvm)
   nvm install 20
   nvm use 20

   # Or install latest LTS
   nvm install --lts
   ```

   The packaged builds (Docker image, macOS installer, Linux and Windows binaries) bundle their own Node.js runtime.

#### Accounts Stay Disconnected

**Symptom:** Most or all accounts show `disconnected` or `connectError` and do not recover

**Diagnostic steps:**

```bash
# Check every account's state
curl https://emailengine.example.com/v1/accounts \
  -H "Authorization: Bearer TOKEN" | jq '.accounts[] | {account, state}'

# Check one account
curl https://emailengine.example.com/v1/account/user@example.com \
  -H "Authorization: Bearer TOKEN" | jq '{state, lastError, authFailureDisabledAt}'

# Check logs for connection errors
journalctl -u emailengine | grep -i "ECONNREFUSED\|ETIMEDOUT\|ENOTFOUND"
```

**Common causes:**

1. **Redis out of memory**

   **Check Redis memory:**

   ```bash
   redis-cli INFO memory | grep used_memory_human
   redis-cli INFO memory | grep maxmemory_human
   ```

   **Solution:** with no `maxmemory` set, Redis uses whatever the host has, so the fix is more RAM on the host. With one set, raise it in `/etc/redis/redis.conf` or remove the directive, then restart:

   ```bash
   sudo systemctl restart redis
   ```

   Check the eviction policy while you are there. EmailEngine needs everything it stores, so an `allkeys-*` policy silently discards live data; `noeviction` is the setting to use:

   ```bash
   redis-cli CONFIG GET maxmemory-policy
   ```

   Redis holds every account's credentials and sync state, so `FLUSHDB` is not a cleanup option; it deletes the accounts. Size the instance with [Performance tuning](/docs/advanced/performance-tuning#redis).

2. **Network connectivity issues**

   **Test IMAP connection:**

   ```bash
   # Test IMAP host reachability
   telnet imap.gmail.com 993
   openssl s_client -connect imap.gmail.com:993

   # Test SMTP
   telnet smtp.gmail.com 587
   ```

   **Check firewall:**

   ```bash
   # Check if ports are blocked
   sudo iptables -L -n | grep -E "993|587|465|143"

   # Allow IMAP/SMTP ports
   sudo ufw allow out 993/tcp
   sudo ufw allow out 587/tcp
   ```

   For a complete list of domains and ports EmailEngine needs to reach, see [Outbound Connection Whitelist](/docs/deployment/security#outbound-connection-whitelist).

3. **Rate limiting by the email provider**

   Providers limit simultaneous IMAP connections per mailbox and per source address. Spread accounts across instances or egress addresses, and route the affected accounts through a proxy with the account-level `proxy` field:

   ```json
   {
     "proxy": "socks5://proxy.example.com:1080"
   }
   ```

   See [Proxying connections](/docs/accounts/proxying-connections).

4. **Too many accounts for the worker count**

   ```bash
   # Increase IMAP worker threads
   export EENGINE_WORKERS=8

   # Check resource usage
   top -p $(pgrep -f emailengine | head -1)
   ```

#### Account Switched Off After Authentication Failures

**Symptom:** An account shows `unset` in the API and a **Syncing switched off** badge in the accounts list, and `PUT /v1/account/{account}/reconnect` returns `{"reconnect": false}`

When credentials keep being rejected for longer than `EENGINE_MAX_IMAP_AUTH_FAILURE_TIME` (3 days by default), EmailEngine stops connecting the account rather than retrying a dead password or grant forever.

**Diagnostic:**

```bash
curl https://emailengine.example.com/v1/account/user@example.com \
  -H "Authorization: Bearer TOKEN" | jq '{state, authFailureDisabledAt, disabled: .imap.disabled}'
```

`authFailureDisabledAt` (since v2.79.4) is the time EmailEngine switched the account off. It is `null` when the operator disabled the account deliberately, in which case `imap.disabled` is the setting to clear. To find every parked account at once, the same field is on each entry of `GET /v1/accounts`.

**Solution:** supply working credentials, or resume with the stored ones. Re-authorizing an OAuth2 account, saving new IMAP settings, or **Resume syncing** on the account page all lift it; which forms of `PUT /v1/account/{account}` count has changed between releases. [Accounts switched off after authentication failures](/docs/accounts/managing-accounts#accounts-switched-off-after-authentication-failures) is the canonical description of the mechanism, the recovery paths and the per-version differences, and [`EENGINE_MAX_IMAP_AUTH_FAILURE_TIME`](/docs/configuration/environment-variables#max-imap-auth-failure-time) is the setting behind the window.

#### IMAP Connection Timeouts

**Symptom:** Accounts connect but frequently time out

**Diagnostic:**

```bash
# Enable protocol logging (writes the raw IMAP conversation, credentials included)
export EENGINE_LOG_RAW=true
export EENGINE_LOG_LEVEL=trace

# Check logs for timeouts
journalctl -u emailengine | grep -i timeout

# Measure network latency
ping imap.gmail.com
traceroute imap.gmail.com
```

**Solutions:**

1. **Raise the IMAP socket timeout:**

   ```bash
   # Milliseconds or a duration string; unset by default
   export EENGINE_IMAP_SOCKET_TIMEOUT=120s
   ```

2. **Check network quality:**

   ```bash
   # Test packet loss
   mtr -c 100 imap.gmail.com

   # Check firewall interference
   sudo iptables -L -v
   ```

3. **Route through a proxy if the direct path is the problem:**

   ```json
   {
     "proxy": "socks5://proxy.example.com:1080"
   }
   ```

### OAuth2 Authentication Issues

#### OAuth2 Flow Fails

**Symptom:** OAuth2 authentication page shows error or redirect fails

**Diagnostic:**

```bash
# List OAuth2 applications
curl https://emailengine.example.com/v1/oauth2 \
  -H "Authorization: Bearer TOKEN" | jq '.'

# Check one application (use the app ID from the listing)
curl https://emailengine.example.com/v1/oauth2/AAABhaBPHscAAAAH \
  -H "Authorization: Bearer TOKEN" | jq '.'

# Check that the provider's token endpoint is reachable
curl -I https://oauth2.googleapis.com/token
```

**Common causes:**

1. **Invalid client ID/secret**

   **Solution:**

   - Verify the client ID and secret in Google Cloud Console (or the Microsoft Entra app for Outlook)
   - Update the values in the EmailEngine OAuth2 application (dashboard under Integrations > OAuth2 Apps, or `PUT /v1/oauth2/{app}`). OAuth2 credentials are stored on the OAuth2 application, not in environment variables
   - Check for trailing spaces when pasting the client ID/secret into the app form
   - After updating, use the OAuth2 app's **Verify setup** action (`POST /v1/oauth2/{app}/verify`) to confirm the credentials work

2. **Incorrect redirect URI**

   **Solution:**

   Set EmailEngine's service URL correctly - it is the `serviceUrl` setting (configured in the dashboard under Configuration, or via the `EENGINE_SETTINGS` JSON), not a standalone environment variable. The OAuth2 redirect URI is derived from it.

   ```bash
   # Provide serviceUrl through prepared settings
   export EENGINE_SETTINGS='{"serviceUrl":"https://emailengine.example.com"}'

   # Verify the redirect URI registered in the OAuth2 provider
   # Should match: https://emailengine.example.com/oauth
   ```

3. **OAuth2 scopes insufficient**

   The scopes EmailEngine requests depend on the application's base scopes. The provider console has to allow them:

   **Gmail:**

   - IMAP backend: `https://mail.google.com/`
   - Gmail API backend: `https://www.googleapis.com/auth/gmail.modify`

   **Microsoft 365** (the global cloud; the GCC High, DoD, and China clouds use their own hostnames for the same scopes):

   - IMAP backend: `https://outlook.office.com/IMAP.AccessAsUser.All`, `https://outlook.office.com/SMTP.Send`, `offline_access`, `openid`, `profile`
   - Graph API backend: `https://graph.microsoft.com/Mail.ReadWrite`, `https://graph.microsoft.com/Mail.Send`, `https://graph.microsoft.com/User.Read`, `offline_access`

#### Token Refresh Fails

**Symptom:** Accounts work initially but stop after token expiry

**Diagnostic:**

```bash
# Check token expiry
curl https://emailengine.example.com/v1/account/user@example.com \
  -H "Authorization: Bearer TOKEN" | jq '.oauth2 | {expires, scope, provider}'

# Check logs for refresh errors
journalctl -u emailengine | grep -i "refresh\|invalid_grant"
```

**Solutions:**

1. **Refresh token expired or revoked:**

   - Re-authorize the account
   - An account that has been failing for three days is switched off; see [Account switched off after authentication failures](#account-switched-off-after-authentication-failures)

2. **OAuth2 app disabled:**

   - Verify app status in provider console
   - Check for security alerts

3. **Encryption secret changed:**
   - Tokens encrypted with the previous `EENGINE_SECRET` cannot be decrypted with the new one
   - Restore the previous secret, or re-encrypt the data with `emailengine encrypt` as described in [Secret encryption](/docs/advanced/encryption#changing-encryption-secret), before re-authorizing every account

### Webhook Delivery Issues

#### Webhooks Not Delivered

**Symptom:** Events occur but webhooks aren't received

**Diagnostic:**

```bash
# Check webhook configuration
curl https://emailengine.example.com/v1/settings?webhooks=true\&webhooksEnabled=true\&webhookEvents=true \
  -H "Authorization: Bearer TOKEN"

# Check webhook queue
curl https://emailengine.example.com/v1/settings/queue/notify \
  -H "Authorization: Bearer TOKEN"

# Test webhook endpoint
curl -X POST https://your-app.com/webhooks \
  -H "Content-Type: application/json" \
  -d '{"test": true}'

# Check logs
journalctl -u emailengine | grep -i webhook
```

**Common causes:**

1. **Webhook URL not set, or no events selected**

   `webhookEvents` is an allowlist with no default: with nothing selected, nothing is delivered. `["*"]` selects every event.

   **Solution:**

   ```bash
   curl -X POST https://emailengine.example.com/v1/settings \
     -H "Authorization: Bearer TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"webhooks": "https://your-app.com/webhooks", "webhooksEnabled": true, "webhookEvents": ["*"]}'
   ```

2. **Webhook endpoint unreachable**

   **Test from EmailEngine server:**

   ```bash
   curl -I https://your-app.com/webhooks

   # Check DNS
   nslookup your-app.com

   # Check firewall
   telnet your-app.com 443
   ```

3. **Webhook timeout**

   Each delivery attempt is capped at 30 seconds. A receiver that needs longer than that has to acknowledge first and do the work afterwards. If the endpoint is genuinely slow and you cannot change it, raise the cap at startup:

   ```bash
   EENGINE_WEBHOOK_TIMEOUT=60s
   ```

   The retry policy itself is fixed: 10 attempts with exponential backoff. It is not configurable.

4. **TLS certificate issues**

   The webhook destination's certificate has to validate against the system's trust store. Check the chain the endpoint serves:

   ```bash
   openssl s_client -connect your-app.com:443 -servername your-app.com
   ```

   A self-signed or incomplete chain is fixed at the endpoint; there is no setting that disables verification for webhooks.

5. **Destination refused by EmailEngine**

   The webhook error flag on the account or configuration page shows `EEGRESSBLOCKED` (the destination resolves to a blocked address, by default the link-local range used by cloud instance metadata) or `EREDIRECTNOTFOLLOWED` (the endpoint answered with a redirect, which is not followed).

   **Solution:** point the webhook at the endpoint's final, routable URL, or adjust `EENGINE_WEBHOOK_EGRESS_POLICY`. See [Blocked destinations and redirects](/docs/webhooks/overview#blocked-destinations-and-redirects).

#### Webhooks Delayed

**Symptom:** Webhooks delivered but with significant delay

**Diagnostic:**

```bash
# Check queue status
curl https://emailengine.example.com/v1/settings/queue/notify \
  -H "Authorization: Bearer TOKEN" | jq

# Check backlog
redis-cli LLEN "bull:notify:wait"

# Monitor webhook processing
journalctl -u emailengine -f | grep webhook
```

A growing `waiting` count means deliveries are being produced faster than they complete, so the fix is either fewer or cheaper deliveries, or more of them at once.

**Solutions:**

1. **Deliver in parallel:**

   One webhook worker processes one delivery at a time by default. Total concurrency is workers times per-worker concurrency:

   ```bash
   # 4 webhook worker threads
   EENGINE_WORKERS_WEBHOOKS=4

   # each handling 2 deliveries at a time, so 8 in flight
   EENGINE_NOTIFY_QC=2
   ```

   Raise this only as far as the receiving endpoint can absorb; a slow receiver saturates at its own rate whatever EmailEngine does. [Webhook configuration](/docs/advanced/performance-tuning#webhook-configuration) has the sizing guidance.

2. **Reduce the webhook payload:**

   - Turn off `notifyText` and `notifyAttachments` in the webhook settings
   - Fetch content on demand through the API instead

3. **Deliver fewer events:**

   - Narrow `webhookEvents` from `["*"]` to the events the integration acts on
   - Check Redis latency, which is added to every queue operation

   Confirm the queue is not paused at all. `GET /v1/settings/queue/notify` reports `paused`; resume it with:

   ```bash
   curl -X PUT https://emailengine.example.com/v1/settings/queue/notify \
     -H "Authorization: Bearer TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"paused": false}'
   ```

### Performance Issues

#### High Memory Usage

**Symptom:** EmailEngine consumes excessive RAM

**Diagnostic:**

```bash
# Check memory usage
ps aux | grep emailengine

# Check the Node.js heap through the Prometheus metrics endpoint.
# The token needs the "metrics" scope, which only the admin interface and
# `emailengine tokens issue` can grant.
curl https://emailengine.example.com/metrics \
  -H "Authorization: Bearer METRICS_TOKEN" | grep heap

# Check account count
curl https://emailengine.example.com/v1/accounts \
  -H "Authorization: Bearer TOKEN" | jq '.total'
```

**Solutions:**

1. **Too many accounts:**

   Budget 1-2 MiB of Redis memory per account and provision twice that, more for very large mailboxes; see [Redis sizing](/docs/advanced/performance-tuning#redis). Add instances or RAM when the budget is exceeded.

2. **Memory growth over time:**

   ```bash
   # Update to the latest release, then restart
   sudo systemctl restart emailengine
   ```

   [Installation](/docs/installation) has the update procedure for each install method. Report growth that persists on the current release as a [GitHub issue](https://github.com/postalsys/emailengine/issues) with the `/metrics` output.

#### Slow Performance

**Symptom:** API requests slow, UI sluggish

**Diagnostic:**

```bash
# Check Redis latency
redis-cli --latency
redis-cli --latency-history

# Check CPU usage
top

# Check IMAP response times
# (enable EENGINE_LOG_RAW=true and check logs)

# Test API performance
time curl https://emailengine.example.com/v1/accounts \
  -H "Authorization: Bearer TOKEN"
```

**Solutions:**

1. **Redis latency high:**

   Redis is EmailEngine's only data store, so its round-trip time is added to nearly every operation. A cross-region Redis is not a workable setup; move it onto the same host or the same LAN. Check what the current one costs:

   ```bash
   # Average round-trip in milliseconds, sampled continuously
   redis-cli --latency
   ```

   Leave `tcp-keepalive` at the Redis default. Setting it to `0` leaves half-open connections behind after a network hiccup, which shows up as stalled accounts rather than as an error.

2. **Too few workers:**

   ```bash
   # One IMAP worker thread per CPU core
   export EENGINE_WORKERS=cpus
   ```

   [Performance tuning](/docs/advanced/performance-tuning) covers how many accounts a thread can carry.

3. **Slow Redis commands:**

   ```bash
   # Log every command that takes longer than 10 ms
   redis-cli CONFIG SET slowlog-log-slower-than 10000

   # Read the ten most recent entries back
   redis-cli SLOWLOG GET 10
   ```

### Email Sync Issues

#### Messages Not Syncing

**Symptom:** New emails don't appear in EmailEngine

**Diagnostic:**

```bash
# Check account state
curl https://emailengine.example.com/v1/account/user@example.com \
  -H "Authorization: Bearer TOKEN" | jq '.state'

# Request a sync
curl -X PUT https://emailengine.example.com/v1/account/user@example.com/sync \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"sync": true}'

# Check logs
journalctl -u emailengine | grep -i "sync\|idle"
```

**Solutions:**

1. **IMAP IDLE not available:**

   Without IDLE, changes are picked up on the periodic full resync, every `imap.resyncDelay` seconds (900 by default). Lower it on the account if the delay is too long.

2. **Account not in a syncing state:**

   `connected` and `syncing` are the two healthy states. `paused` and `unset` mean no connection is maintained at all, and `authenticationError` means the credentials were rejected and have to be replaced. `disconnected` and `connectError` are retried with backoff, so those recover on their own if the underlying problem clears. [Account states](/docs/accounts/managing-accounts#account-states) lists what each one needs.

3. **Folder not monitored:**
   - Check the account's `path` setting, which lists the folders EmailEngine syncs; `"*"` means all of them
   - Add the folder, or set `path` to `"*"`

#### Deleted Messages Re-appear

**Symptom:** Deleted emails come back after sync, or turn up under a different message ID

**Cause:** `DELETE /v1/account/{account}/message/{message}` moves the message to Trash rather than erasing it, and deletes it only when it is already in Trash. A message that reappears is normally the copy now sitting in Trash, which has its own ID because the ID encodes the folder and the IMAP UID.

**Solution:** check Trash first, then delete permanently if that is what you meant:

```bash
# Find the Trash folder
curl https://emailengine.example.com/v1/account/user123/mailboxes \
  -H "Authorization: Bearer TOKEN" | jq '.mailboxes[] | select(.specialUse=="\\Trash")'

# Delete outright, wherever the message currently is (IMAP accounts only)
curl -X DELETE "https://emailengine.example.com/v1/account/user123/message/AAAAAQAACnA?force=true" \
  -H "Authorization: Bearer TOKEN"
```

`force` has no effect on a Gmail API account: those always move the message to Trash.

## Step-by-Step Diagnostic Procedures

### Procedure 1: Complete Health Check

```bash
#!/bin/bash
echo "=== EmailEngine Health Check ==="

# 1. Check service
echo "1. Service status:"
systemctl is-active emailengine

# 2. Check Redis
echo "2. Redis status:"
redis-cli ping

# 3. Check health endpoint
echo "3. Health endpoint:"
curl -s https://emailengine.example.com/health | jq

# 4. Check memory
echo "4. Redis memory:"
redis-cli INFO memory | grep -E "used_memory_human|maxmemory_human"

# 5. Check accounts
echo "5. Account status:"
curl -s https://emailengine.example.com/v1/accounts \
  -H "Authorization: Bearer TOKEN" | \
  jq '[.accounts[] | {account: .account, state: .state}]'

# 6. Check logs for errors
echo "6. Recent errors:"
journalctl -u emailengine --since "5 minutes ago" | grep '"level":50'
```

### Procedure 2: Network Connectivity Test

```bash
#!/bin/bash
echo "=== Network Connectivity Test ==="

# Test IMAP
echo "Testing IMAP (Gmail):"
timeout 5 bash -c "</dev/tcp/imap.gmail.com/993" && echo "OK" || echo "FAILED"

# Test SMTP
echo "Testing SMTP (Gmail):"
timeout 5 bash -c "</dev/tcp/smtp.gmail.com/587" && echo "OK" || echo "FAILED"

# Test Redis
echo "Testing Redis:"
redis-cli ping

# Test webhook endpoint
echo "Testing webhook:"
curl -I -s https://your-app.com/webhooks | head -1
```

### Procedure 3: Account Connection Test

```bash
#!/bin/bash
ACCOUNT="user@example.com"
TOKEN="your-api-token"

echo "=== Account Connection Test ==="

# 1. Get account info
echo "1. Account info:"
curl -s "https://emailengine.example.com/v1/account/$ACCOUNT" \
  -H "Authorization: Bearer $TOKEN" | jq '{state, lastError, authFailureDisabledAt}'

# 2. Request a reconnect (answers {"reconnect": false} for an account that was switched off)
echo "2. Testing reconnection:"
curl -X PUT "https://emailengine.example.com/v1/account/$ACCOUNT/reconnect" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"reconnect": true}'

# 3. Wait and check
sleep 10
echo "3. New state:"
curl -s "https://emailengine.example.com/v1/account/$ACCOUNT" \
  -H "Authorization: Bearer $TOKEN" | jq '.state'
```

## Log Analysis Tips

EmailEngine writes one JSON object per line. `level` is numeric: 30 is info, 40 warning, 50 error, 60 fatal.

### Useful Log Commands

```bash
# View logs in real-time
journalctl -u emailengine -f

# Last 100 lines
journalctl -u emailengine -n 100

# Errors only
journalctl -u emailengine | grep '"level":50'

# Specific time range
journalctl -u emailengine --since "1 hour ago"

# Export logs
journalctl -u emailengine --since "today" > emailengine-$(date +%Y%m%d).log

# Entries for one account
journalctl -u emailengine | grep '"account":"user@example.com"'

# Count errors
journalctl -u emailengine --since "1 hour ago" | grep -c '"level":50'
```

### Log Patterns to Look For

**Connection issues** (the `code` of the attached error):

```
grep -i "ECONNREFUSED\|ETIMEDOUT\|ENOTFOUND\|ECONNRESET" emailengine.log
```

**Authentication failures** (the IMAP client logs one line per rejected login, and the attached error carries the flag):

```
grep '"msg":"Failed to authenticate"\|"authenticationFailed":true' emailengine.log
```

**Webhook failures**:

```
grep '"msg":"Failed posting webhook"' emailengine.log
```

## Getting Help

### Information to Collect

When requesting support, provide:

1. **EmailEngine version:**

   ```bash
   emailengine version
   ```

2. **System information:**

   ```bash
   uname -a
   node --version
   redis-server --version
   ```

3. **Configuration:**

   The environment variables or the TOML file the service runs with, for example `/etc/emailengine/config.toml` on a [SystemD install](/docs/deployment/systemd). Remove `EENGINE_SECRET`, passwords, and license keys before sharing.

4. **Logs:**

   ```bash
   journalctl -u emailengine -n 200 > logs.txt
   ```

5. **Account state:**
   ```bash
   curl https://emailengine.example.com/v1/accounts \
     -H "Authorization: Bearer TOKEN" | \
     jq '[.accounts[] | {account, state, lastError}]'
   ```

### Support Channels

- **Documentation:** [learn.emailengine.app](https://learn.emailengine.app)
- **GitHub Issues:** [github.com/postalsys/emailengine/issues](https://github.com/postalsys/emailengine/issues)
- **Email support:** [support@emailengine.app](mailto:support@emailengine.app), see [Support](/docs/support)

## See Also

- [Account troubleshooting](/docs/accounts/troubleshooting) - Connection, authentication, and sync failures
- [Error codes](/docs/reference/error-codes) - What a given code means and whether to retry
- [Logging](/docs/advanced/logging) - Turning up detail on the server or one account
- [Monitoring](/docs/advanced/monitoring) - Health checks and metrics
- [Support](/docs/support) - What to include when you ask for help
