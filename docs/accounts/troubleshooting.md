---
title: Account Troubleshooting
sidebar_position: 13
description: Diagnosing connection, authentication, and sync failures on connected accounts
---

# Account Troubleshooting

This guide covers common issues when working with email accounts in EmailEngine and how to resolve them.

## Quick Diagnostics

### Check Account Status

First, check the account's current state using the [Get Account API endpoint](/docs/api/get-v-1-account-account):

```bash
curl https://emailengine.example.com/v1/account/user123 \
  -H "Authorization: Bearer YOUR_TOKEN"
```

Look for:
- `state`: Current account state
- `lastError`: Recent error messages
- `syncTime`: Last successful sync
- Connection details

### Check EmailEngine Logs

EmailEngine logs contain detailed error information:

```bash
# If running with systemd
journalctl -u emailengine -f

# If running with Docker
docker logs -f emailengine

# If running manually (logs go to stdout)
# EmailEngine uses pino for JSON logging to stdout
```

## Common Account States and Solutions

### State: authenticationError

**What it means:** Invalid or expired credentials.

#### For IMAP/SMTP Accounts

**Common Causes:**

1. **Incorrect password**
   - Password was changed on the email provider
   - Typo in password
   - Wrong username

   **Solution:**
   ```bash
   curl -X PUT https://emailengine.example.com/v1/account/user123 \
     -H "Authorization: Bearer YOUR_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{
       "imap": { "partial": true, "auth": { "user": "user@example.com", "pass": "correct-password" } },
       "smtp": { "partial": true, "auth": { "user": "user@example.com", "pass": "correct-password" } }
     }'

   # Then reconnect using the Reconnect Account API endpoint
   curl -X PUT https://emailengine.example.com/v1/account/user123/reconnect \
     -H "Authorization: Bearer YOUR_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"reconnect": true}'
   ```

2. **App password required but not used**
   - Gmail: Account passwords completely disabled, app-specific passwords required for all accounts
   - Yahoo, iCloud: App-specific passwords required if 2FA is enabled

   **Solution:**
   - Generate app-specific password from provider
   - Update account with app password instead of main password
   - [Gmail app passwords](https://myaccount.google.com/apppasswords) (requires 2FA enabled)
   - [Yahoo app passwords](https://login.yahoo.com/account/security)
   - [iCloud app passwords](https://appleid.apple.com/)

3. **Password authentication disabled (Gmail)**
   - Gmail has completely disabled account password authentication for all accounts
   - The "Less secure app access" feature is no longer available

   **Solution:**
   - Use app-specific passwords (requires 2FA): [Gmail app passwords](https://myaccount.google.com/apppasswords)
   - Or switch to OAuth2: [Gmail OAuth2 guide](./gmail/gmail-imap)

4. **IMAP/SMTP disabled (Microsoft 365)**
   - Admin may have disabled IMAP/SMTP protocols

   **Solution:**
   - Enable in [Microsoft 365 admin center](https://admin.microsoft.com/)
   - Navigate to Users → Active users → Mail settings
   - Enable IMAP and SMTP AUTH
   - Or use MS Graph API: [Outlook setup guide](./microsoft-365/outlook-365)

#### For OAuth2 Accounts

**Common Causes:**

1. **Access token expired and refresh failed**
   - Refresh token may be invalid
   - OAuth2 app credentials changed
   - User revoked access

   **Solution:**
   - Have user re-authenticate via hosted authentication form
   - Generate new authentication form URL:
   ```bash
   curl -X POST https://emailengine.example.com/v1/authentication/form \
     -H "Authorization: Bearer YOUR_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{
       "account": "user123",
       "email": "user@gmail.com",
       "redirectUrl": "https://myapp.com/settings"
     }'
   ```

2. **OAuth2 app misconfigured**
   - Client ID or secret incorrect
   - Redirect URL mismatch
   - Required scopes not configured

   **Solution:**
   - Check OAuth2 app settings in EmailEngine
   - Verify against Google Cloud Console / Azure AD
   - Look for error messages in OAuth2 app settings page
   - [Gmail OAuth2 setup guide](./gmail/gmail-imap)
   - [Outlook OAuth2 setup guide](./microsoft-365/outlook-365)

3. **Insufficient permissions**
   - Account doesn't have required scopes
   - For shared mailboxes: user doesn't have access

   **Solution:**
   - Update OAuth2 app scopes
   - Have user re-authenticate to grant new scopes
   - For shared mailboxes: verify user has permissions in Microsoft 365 admin

### State: connectError

**What it means:** Cannot reach the mail server.

**Common Causes:**

1. **Incorrect host or port**
   ```bash
   # Check current settings
   curl https://emailengine.example.com/v1/account/user123 \
     -H "Authorization: Bearer YOUR_TOKEN" | jq '.imap'
   ```

   **Solution:**
   - Verify IMAP/SMTP settings with provider documentation
   - Common Gmail settings: [Gmail IMAP](https://support.google.com/mail/answer/7126229)
   - Common Outlook settings: [Outlook IMAP](https://support.microsoft.com/en-us/office/pop-imap-and-smtp-settings-8361e398-8af4-4e97-b147-6c6c4ac95353)

2. **Firewall blocking connections**
   - EmailEngine server firewall blocks outbound connections
   - Corporate firewall blocks email ports

   **Solution:**
   ```bash
   # Test connectivity from EmailEngine server
   telnet imap.gmail.com 993
   telnet smtp.gmail.com 587
   ```

   If connection fails, check firewall rules:
   ```bash
   # Allow outbound connections to IMAP/SMTP ports
   iptables -A OUTPUT -p tcp --dport 993 -j ACCEPT  # IMAP SSL
   iptables -A OUTPUT -p tcp --dport 587 -j ACCEPT  # SMTP STARTTLS
   iptables -A OUTPUT -p tcp --dport 465 -j ACCEPT  # SMTP SSL
   ```

3. **DNS resolution failure**
   - Cannot resolve mail server hostname

   **Solution:**
   ```bash
   # Test DNS resolution
   nslookup imap.gmail.com
   dig imap.gmail.com
   ```

   If DNS fails:
   - Check `/etc/resolv.conf`
   - Verify network configuration
   - Try different DNS server (e.g., 8.8.8.8)

4. **Server is down or unreachable**
   - Mail provider having outage
   - Server maintenance

   **Solution:**
   - Check provider status page
   - Wait and retry later
   - EmailEngine will automatically retry

### State: connecting

**What it means:** Connection in progress.

**Normal Behavior:**
- Accounts usually stay in this state for 5-30 seconds
- First connection may take longer (folder sync)

**If Stuck:**

1. **Check logs** for what's happening:
   ```bash
   journalctl -u emailengine -f | grep user123
   ```

2. **Possible issues:**
   - Slow server response
   - Large mailbox syncing
   - Network latency

3. **Wait** a few minutes before intervening

4. **If stuck >5 minutes:**
   ```bash
   # Trigger reconnection
   curl -X PUT https://emailengine.example.com/v1/account/user123/reconnect \
     -H "Authorization: Bearer YOUR_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"reconnect": true}'
   ```

### State: unset

**What it means:** OAuth2 authentication not completed.

**Occurs When:**
- Hosted authentication form URL was generated
- User hasn't completed OAuth2 flow yet

**Solution:**
- User needs to visit the authentication form URL
- Complete OAuth2 consent
- Account will automatically move to `connecting` then `connected`

**If user says they completed it:**
- Check redirect URL is correct
- Verify OAuth2 app is enabled in EmailEngine
- Check for errors in OAuth2 app settings
- Generate new authentication form URL

### State: disconnected

**What it means:** Account is disconnected (manually disabled or closed).

**Solution:**
```bash
# Re-enable account if it was disabled
curl -X PUT https://emailengine.example.com/v1/account/user123 \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "imap": { "partial": true, "disabled": false } }'

# Then reconnect
curl -X PUT https://emailengine.example.com/v1/account/user123/reconnect \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"reconnect": true}'
```

## Provider-Specific Issues

### Gmail Issues

#### Account Password Authentication No Longer Supported

**Error Message:** "Please log in via your web browser" or "Invalid credentials"

:::danger Gmail Account Passwords Disabled
Gmail has completely disabled account password authentication. The "Less secure app access" feature is no longer available. You **must** use app passwords or OAuth2.
:::

**Solution:**
1. **App Passwords (for testing):** Generate an [app-specific password](https://support.google.com/accounts/answer/185833) (requires 2FA enabled)
2. **OAuth2 (recommended for production):** Follow the [Gmail OAuth2 guide](./gmail/gmail-imap)

#### Rate Limits

**Symptoms:**
- Intermittent connection failures
- Slow syncing
- Temporary authentication errors

**Gmail Limits:**
- 15 concurrent IMAP connections
- 2500 MB download/day
- 500 MB upload/day

**Solution:**
- Reduce sub-connections
- Implement path filtering
- Consider Gmail API for high-volume: [Gmail API guide](./gmail/gmail-api)
- Spread operations over time

#### OAuth2 Scope Too Wide (Public Apps)

**Error:** Google rejects your OAuth2 app verification

**Reason:** `https://mail.google.com/` scope too broad

**Solution:**
- Use narrower scopes if possible
- Justify why IMAP access is needed
- Consider Internal app (organization only)
- Consider app passwords as alternative

### Outlook/Microsoft 365 Issues

#### IMAP Not Enabled

**Error Message:** "IMAP is disabled"

**Solution:**
1. Go to [Microsoft 365 admin center](https://admin.microsoft.com/)
2. Users → Active users → Select user
3. Mail tab → Manage email apps
4. Enable IMAP
5. Wait 15-30 minutes for changes to propagate

#### OAuth2 redirect_uri Mismatch

**Error Code:** AADSTS50011

**Solution:**
1. Check redirect URI in Azure AD app registration
2. Must match exactly in EmailEngine OAuth2 settings
3. Check for:
   - http vs https
   - Trailing slashes
   - Port numbers
   - Case sensitivity

#### Admin Consent Required

**Error Message:** Need admin approval

**Solution:**
- Organization admin must grant consent
- Or admin can pre-approve app for all users
- In Azure AD → App registrations → API permissions → Grant admin consent

#### Shared Mailbox Access Denied

**Symptoms:**
- Authentication succeeds but cannot access mailbox
- "Mailbox not found" error

**Solution:**
1. Verify user has "Full Access" permission to shared mailbox
2. In Microsoft 365 admin:
   - Recipients → Shared → Select mailbox
   - Mailbox delegation → Full Access
   - Add user
3. Wait 15-30 minutes for permissions to propagate

### Yahoo/AOL/Verizon Issues

#### App Password Required

**Error Message:** "Invalid credentials"

**Cause:** 2FA enabled, app password needed

**Solution:**
1. Generate app password:
   - Yahoo: [Account Security](https://login.yahoo.com/account/security)
   - AOL: Account Security settings
2. Use app password instead of main password
3. Update account in EmailEngine

### iCloud Issues

#### App-Specific Password Required

**Error Message:** "Invalid credentials"

**Solution:**
1. Generate app-specific password:
   - Visit [appleid.apple.com](https://appleid.apple.com/)
   - Sign in → Security → App-Specific Passwords
   - Generate password
2. Use app-specific password in EmailEngine

#### Two-Factor Authentication Must Be Enabled

iCloud requires 2FA enabled to generate app-specific passwords.

**Solution:**
1. Enable 2FA on Apple ID
2. Then generate app-specific password

## Webhook Issues

### Webhooks Not Firing

**Check webhook configuration:**

```bash
curl "https://emailengine.example.com/v1/settings?webhooks=true" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  | jq '.webhooks'
```

**Common Causes:**

1. **Webhook URL not set**
   ```bash
   curl -X POST https://emailengine.example.com/v1/settings \
     -H "Authorization: Bearer YOUR_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{ "webhooks": "https://myapp.com/webhooks" }'
   ```

2. **Webhook URL unreachable**
   - Test manually:
   ```bash
   curl -X POST https://myapp.com/webhooks \
     -H "Content-Type: application/json" \
     -d '{"test": true}'
   ```
   - Check firewall allows EmailEngine IP
   - Verify SSL certificate is valid

3. **Webhook endpoint returning errors**
   - Check your webhook handler logs
   - Must return 2xx status code
   - EmailEngine will retry on failures

4. **For Gmail API/MS Graph accounts: Subscription not active**
   - Check account details for subscription status
   - For Gmail API: Verify Cloud Pub/Sub is configured
   - For MS Graph: Verify subscription is active

**Debug webhooks:**

Check webhook queue in Bull Board:
- Navigate to **System** → **Queues** in the EmailEngine dashboard (`/admin/bull-board`)
- Check the "notify" queue
- Look for failed jobs and error messages

### Webhook Delays

**Cause:** Webhook queue backed up

**Solution:**
1. Check Bull Board for queue status
2. Increase webhook workers (if needed)
3. Optimize your webhook endpoint response time
4. Implement idempotency (handle duplicate webhooks)

## Connection Issues

### Too Many Connections

**Error Message:** "Maximum number of connections reached"

**Cause:** Exceeding provider's concurrent connection limit

**Solution:**
```bash
# Check current subconnections
curl https://emailengine.example.com/v1/account/user123 \
  -H "Authorization: Bearer YOUR_TOKEN" \
  | jq '.subconnections'

# Reduce sub-connections
curl -X PUT https://emailengine.example.com/v1/account/user123 \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "subconnections": [] }'
```

**Provider Limits:**
- Gmail: 15 concurrent connections
- Outlook: 15 concurrent connections
- Yahoo: 10 concurrent connections

### SSL/TLS Certificate Errors

**Error Message:** "Certificate verification failed" or "CERT_HAS_EXPIRED"

**For Provider Servers:**

Usually indicates provider issue or misconfigured server.

**For Self-Hosted Servers:**

```bash
# Accept self-signed certificates (development only)
curl -X PUT https://emailengine.example.com/v1/account/user123 \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "imap": {
      "partial": true,
      "tls": {
        "rejectUnauthorized": false
      }
    }
  }'
```

:::warning Security
Only disable certificate verification for development/testing with self-signed certs. In production, use proper CA-signed certificates.
:::

### IDLE Timeout Issues

**Symptoms:**
- Connection drops after period of inactivity
- Frequent reconnections

**Cause:** Server doesn't support IMAP IDLE or closes IDLE after timeout

**Solution:**
EmailEngine handles this automatically by:
- Detecting IDLE timeout
- Reconnecting automatically
- Falling back to polling if IDLE not supported

No action needed from you. If issues persist, check logs for specific errors.

## Performance Issues

### Slow Initial Sync

**Symptoms:**
- Account stuck in "syncing" for long time
- First sync takes hours

**Cause:** Large mailbox with many messages

**Solution:**

1. **Use path filtering** to sync only needed folders:
   ```bash
   curl -X PUT https://emailengine.example.com/v1/account/user123 \
     -H "Authorization: Bearer YOUR_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{
       "path": ["INBOX", "\\Sent"]
     }'
   ```

2. **Be patient** - Initial sync can take time for large mailboxes
   - 10,000 messages: ~5-10 minutes
   - 50,000 messages: ~30-60 minutes
   - 100,000+ messages: Hours

3. **Consider Gmail API** for very large Gmail accounts:
   - Faster initial sync
   - Better performance
   - [Gmail API guide](./gmail/gmail-api)

### High Memory/CPU Usage

**Symptoms:**
- EmailEngine using excessive resources
- Server becomes slow

**Solutions:**

1. **Reduce number of accounts**
   - Check account count: `curl https://emailengine.example.com/v1/accounts | jq '.total'`
   - Scale vertically (increase server resources)

2. **Reduce sub-connections**
   - Remove unnecessary sub-connections
   - Only monitor critical folders in real-time

3. **Implement path filtering**
   - Don't sync unnecessary folders
   - Use wildcards carefully

4. **Optimize webhook endpoint**
   - Slow webhook responses cause queue backup
   - Implement async processing
   - Return 200 immediately, process in background

5. **Increase Redis memory**
   - EmailEngine stores data in Redis
   - Ensure adequate Redis memory allocation

## OAuth2-Specific Issues

### Token Refresh Failures

**Symptoms:**
- Account enters authenticationError periodically
- "invalid_grant" errors in logs

**Causes:**

1. **Refresh token expired (Microsoft)**
   - Microsoft refresh tokens expire after 90 days of inactivity
   - EmailEngine keeps them active by regular use
   - If expired, user must re-authenticate

2. **OAuth2 app credentials changed**
   - Client secret rotated but not updated in EmailEngine
   - **Solution:** Update OAuth2 app settings in EmailEngine with new credentials

3. **User revoked access**
   - User manually revoked app permission
   - **Solution:** User must re-authenticate

4. **OAuth2 app disabled/deleted**
   - App deleted in Google Cloud Console / Azure AD
   - **Solution:** Recreate app or update settings

:::info Accounts stop retrying after three days
Whatever the cause, an account that keeps failing authentication is [parked](/docs/reference/configuration-options#max-imap-auth-failure-time) once the failures have run for `EENGINE_MAX_IMAP_AUTH_FAILURE_TIME`, so a dead grant is not retried against the provider forever. Check `imap.disabled` on the account before assuming a re-authorization has not been done yet.
:::

### "redirect_uri_mismatch" Error

**Google Error Message:** "The redirect URI in the request does not match..."

**Microsoft Error Code:** AADSTS50011

**Solution:**

1. Check redirect URI in OAuth2 app configuration (Google Cloud Console / Azure AD)
2. Check redirect URL in EmailEngine OAuth2 app settings
3. They must match **exactly**:
   - Case-sensitive
   - http vs https
   - Trailing slashes matter
   - Port numbers must match
   - Domain must match

**Example mismatch:**
- Provider: `https://ee.company.com/oauth`
- EmailEngine: `https://ee.company.com:3000/oauth`
- [NO] Port mismatch!

### Insufficient Permissions

**Error Message:** "insufficient_scope" or "unauthorized_client"

**Cause:** Required scope not configured

**Solution:**

1. **Check required scopes:**
   - Gmail IMAP: `https://mail.google.com/`
   - Gmail API: `gmail.modify`
   - Outlook IMAP: `IMAP.AccessAsUser.All`, `SMTP.Send`, `offline_access`
   - Outlook Graph: `Mail.ReadWrite`, `Mail.Send`, `offline_access`

2. **Update OAuth2 app in provider console:**
   - Google Cloud Console → APIs & Services → OAuth consent screen → Scopes
   - Azure AD → App registrations → API permissions

3. **Update EmailEngine OAuth2 app** if using additional scopes

4. **Have users re-authenticate** to grant new permissions

## Diagnostic Commands

### Check Account Details

```bash
# Full account info
curl https://emailengine.example.com/v1/account/user123 \
  -H "Authorization: Bearer YOUR_TOKEN" \
  | jq .

# Just the state
curl https://emailengine.example.com/v1/account/user123 \
  -H "Authorization: Bearer YOUR_TOKEN" \
  | jq -r .state

# Subconnections info
curl https://emailengine.example.com/v1/account/user123 \
  -H "Authorization: Bearer YOUR_TOKEN" \
  | jq '.subconnections'
```

### Test IMAP Connection

```bash
# From EmailEngine server
openssl s_client -connect imap.gmail.com:993 -crlf
# Type: A LOGIN user@gmail.com password
# Then: B LIST "" "*"
```

### Test SMTP Connection

```bash
# From EmailEngine server
openssl s_client -connect smtp.gmail.com:587 -starttls smtp -crlf
# Type: EHLO example.com
# Then: AUTH LOGIN
```

### Check Network Connectivity

```bash
# Test DNS
dig imap.gmail.com

# Test port connectivity
telnet imap.gmail.com 993
nc -zv imap.gmail.com 993

# Check firewall rules
iptables -L OUTPUT -n

# Trace route
traceroute imap.gmail.com
```

### Verify OAuth2 Token

```bash
# Get current token
curl https://emailengine.example.com/v1/account/user123/oauth-token \
  -H "Authorization: Bearer YOUR_TOKEN"

# Test token with provider API (Gmail example)
TOKEN="ya29.a0AWY7..."
curl https://www.googleapis.com/gmail/v1/users/me/profile \
  -H "Authorization: Bearer $TOKEN"
```

## Getting Help

### Information to Provide

When seeking help, include:

1. **EmailEngine version:** The dashboard footer, or `emailengine --version`
2. **Account state:** From account details API
3. **Error messages:** From logs
4. **Provider:** Gmail, Outlook, Yahoo, etc.
5. **Authentication method:** IMAP/SMTP, OAuth2, service account
6. **Reproduction steps:** What leads to the issue

### EmailEngine Logs

The per-account log is usually more useful than the server log, because it holds the protocol conversation for that one account. Turn it on for the account, reproduce the problem, then read it back:

```bash
curl -X PUT https://emailengine.example.com/v1/account/user123 \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"logs": true}'

curl "https://emailengine.example.com/v1/logs/user123" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

`logs` on an account is a boolean. Retention (`logs.maxLogLines`) and the switch that turns this on for every account (`logs.all`) are server-wide settings. See [Per-account protocol logs](/docs/advanced/logging#per-account-protocol-logs).

See [Logging](/docs/advanced/logging) for the format and what each level records.

For the server log:

```bash
# Logs with account-specific filter
journalctl -u emailengine | grep user123

# Logs with OAuth2 filter
journalctl -u emailengine | grep -i oauth

# Logs with error filter
journalctl -u emailengine | grep -i error

# Follow logs in real-time
journalctl -u emailengine -f
```

### Useful Resources

- [EmailEngine API reference](/docs/api/emailengine-api)
- [EmailEngine GitHub Issues](https://github.com/postalsys/emailengine/issues)
- [Support Contact](/docs/support)
- Provider-specific documentation:
  - [Gmail IMAP Settings](https://support.google.com/mail/answer/7126229)
  - [Outlook IMAP Settings](https://support.microsoft.com/en-us/office/pop-imap-and-smtp-settings-8361e398-8af4-4e97-b147-6c6c4ac95353)

## See Also

- [Managing accounts](/docs/accounts/managing-accounts) - Reconnecting, re-enabling, and rotating credentials
- [Account types](/docs/accounts) - Whether a different backend avoids the problem entirely
- [Logging](/docs/advanced/logging) - Turning up detail on one account or on the whole server
- [Troubleshooting](/docs/troubleshooting) - Problems that are not account-specific
- [Support](/docs/support) - What to send when you ask for help
