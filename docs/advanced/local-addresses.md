---
title: Local IP Address Binding
sidebar_position: 9
description: Configure EmailEngine to use multiple local IP addresses for outbound IMAP and SMTP connections
keywords:
  - local addresses
  - IP binding
  - multiple IPs
  - IP rotation
  - rate limiting
  - connection distribution
---

# Local IP Address Binding

If your server has multiple IP addresses or network interfaces, configure EmailEngine to use specific local addresses for outbound IMAP and SMTP connections. This helps distribute connections across IPs and avoid rate limiting.

## Overview

**Use cases:**

- **Rate limiting avoidance** - Distribute connections across multiple IPs to avoid per-IP rate limits
- **Dedicated IPs per account** - Assign specific IPs to specific accounts
- **IP reputation management** - Use clean IPs for important accounts
- **Load distribution** - Spread connection load across network interfaces
- **Compliance requirements** - Use specific IPs for regulatory compliance

EmailEngine can bind to specific local IP addresses when making outbound connections to email servers.

### 1. Multiple IP Addresses

Your server must have multiple IP addresses configured:

```bash
# Check available IP addresses
ip addr show

# Example output:
# eth0: inet 192.168.1.100/24
# eth0:0: inet 192.168.1.101/24
# eth0:1: inet 192.168.1.102/24
```

### 2. Network Configuration

Ensure IPs are properly configured and routable:

```bash
# Test connectivity from specific IP
curl --interface 192.168.1.101 https://www.google.com
```

### 3. Firewall Rules

Allow outbound connections from all configured IPs:

```bash
# Example iptables rules
iptables -A OUTPUT -s 192.168.1.100 -j ACCEPT
iptables -A OUTPUT -s 192.168.1.101 -j ACCEPT
iptables -A OUTPUT -s 192.168.1.102 -j ACCEPT
```

## Configuration Methods

### Method 1: Global Settings via API

Configure local addresses via the Settings API or web interface (**Configuration** > **Network**). Local addresses are configured globally and EmailEngine selects IPs from this pool based on the configured strategy:

```bash
curl -X POST "http://localhost:3000/v1/settings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "localAddresses": ["192.168.1.100", "192.168.1.101", "192.168.1.102"],
    "imapStrategy": "dedicated",
    "smtpStrategy": "dedicated"
  }'
```

**Configuration options:**

- `localAddresses` - Array of local IP addresses to use for outbound connections
- `imapStrategy` - IP selection strategy for IMAP connections (`default`, `dedicated`, or `random`)
- `smtpStrategy` - IP selection strategy for SMTP connections (`default`, `dedicated`, or `random`)

**Strategy options:**

- `default` - Use the server's default network configuration
- `dedicated` - Each account consistently uses the same IP from the pool (based on account ID hashing)
- `random` - Each connection uses a randomly selected IP from the pool

Note: These settings are runtime settings stored in Redis - they cannot be set via the TOML configuration file. To preconfigure them at startup, use the `EENGINE_SETTINGS` environment variable with a JSON value, for example `EENGINE_SETTINGS='{"localAddresses":["192.168.1.100","192.168.1.101"],"smtpStrategy":"dedicated"}'`.

### Method 2: Dedicated Strategy (Recommended)

Use the `dedicated` strategy for consistent IP assignment per account. EmailEngine automatically assigns each account to a specific IP from your pool using consistent hashing based on the account ID:

```bash
curl -X POST "http://localhost:3000/v1/settings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "localAddresses": ["192.168.1.100", "192.168.1.101", "192.168.1.102"],
    "imapStrategy": "dedicated",
    "smtpStrategy": "dedicated"
  }'
```

With this configuration:
- Each account consistently uses the same IP from the pool
- IP assignment is automatic based on account ID hashing
- Load is distributed across all configured IPs
- No manual IP assignment needed per account

### Method 3: Random Strategy

Use the `random` strategy for connections that don't require consistency:

```bash
curl -X POST "http://localhost:3000/v1/settings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "localAddresses": ["192.168.1.100", "192.168.1.101", "192.168.1.102"],
    "imapStrategy": "random",
    "smtpStrategy": "random"
  }'
```

With this configuration:
- Each new connection randomly selects an IP from the pool
- Good for distributing load when consistency isn't required

## Use Cases

### 1. Avoiding Gmail Rate Limits

Gmail limits IMAP connections per IP. Use the `dedicated` strategy to automatically distribute accounts across multiple IPs:

```bash
# Configure multiple IPs with dedicated strategy
curl -X POST "http://localhost:3000/v1/settings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "localAddresses": [
      "192.168.1.100",
      "192.168.1.101",
      "192.168.1.102",
      "192.168.1.103",
      "192.168.1.104"
    ],
    "imapStrategy": "dedicated",
    "smtpStrategy": "dedicated"
  }'
```

With 5 IPs and 100 Gmail accounts:
- Accounts are automatically distributed across all 5 IPs
- Each account consistently uses the same IP
- Approximately 20 accounts per IP

### 2. IP Rotation for High Volume

Use the `random` strategy to distribute load across IPs for high-volume sending:

```bash
curl -X POST "http://localhost:3000/v1/settings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "localAddresses": [
      "192.168.1.100",
      "192.168.1.101",
      "192.168.1.102",
      "192.168.1.103"
    ],
    "imapStrategy": "dedicated",
    "smtpStrategy": "random"
  }'
```

This configuration:
- Uses consistent IPs for IMAP (receiving) connections
- Randomly distributes SMTP (sending) connections across all IPs
- Helps avoid per-IP sending rate limits

### 3. Separate Strategies for IMAP and SMTP

Configure different strategies for receiving vs sending:

```bash
curl -X POST "http://localhost:3000/v1/settings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "localAddresses": ["192.168.1.100", "192.168.1.101", "192.168.1.102"],
    "imapStrategy": "dedicated",
    "smtpStrategy": "random"
  }'
```

This is useful when:
- You need consistent IMAP connections (to maintain session state)
- You want to distribute SMTP load across multiple IPs

## Verifying Configuration

### Check Global Settings

Verify local addresses and strategies are configured:

```bash
curl "http://localhost:3000/v1/settings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  | jq '{localAddresses, imapStrategy, smtpStrategy}'
```

Expected output:

```json
{
  "localAddresses": ["192.168.1.100", "192.168.1.101", "192.168.1.102"],
  "imapStrategy": "dedicated",
  "smtpStrategy": "dedicated"
}
```

### Test Connectivity

Verify your server can connect using the configured IPs:

```bash
# Check IMAP connection from specific IP
telnet --bind-address 192.168.1.101 imap.gmail.com 993

# Check SMTP connection from specific IP
telnet --bind-address 192.168.1.101 smtp.gmail.com 587
```

### Monitor Active Connections

Check which IPs are being used:

```bash
# View active connections
netstat -an | grep ESTABLISHED | grep 993  # IMAP
netstat -an | grep ESTABLISHED | grep 587  # SMTP

# Count connections per local IP
netstat -an | grep ESTABLISHED | grep 993 | awk '{print $4}' | cut -d: -f1 | sort | uniq -c
```

### EmailEngine Logs

Check logs for IP selection info:

```bash
# View connection logs showing selected local address
# Look for "Selected local address" log entries
tail -f /var/log/emailengine.log | grep "Selected local address"
```

The logs show which IP was selected for each connection, including the selection strategy used.

## Updating Configuration

Update the global IP pool or strategy:

```bash
curl -X POST "http://localhost:3000/v1/settings" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "localAddresses": ["192.168.1.100", "192.168.1.101", "192.168.1.102", "192.168.1.103"],
    "imapStrategy": "random",
    "smtpStrategy": "dedicated"
  }'
```

Changes take effect for new connections. Existing connections continue using their current IP until they reconnect.

## Performance Considerations

### Connection Overhead

Each IP adds slight overhead:

- DNS lookups
- TCP handshakes
- TLS negotiations

**Impact:** Minimal (less than 100ms per connection)

### Memory Usage

Multiple IPs don't significantly increase memory:

- ~1MB per account regardless of IP
- IP binding is just a socket option

### Throughput

Properly distributed IPs can improve throughput:

- Avoid per-IP rate limits
- Parallel connections across IPs
- Better load distribution

## See Also

- [Performance tuning](/docs/advanced/performance-tuning) - Spreading connections across addresses at scale
- [Inbox placement testing](/docs/advanced/inbox-placement-testing) - Checking how a sending address is received
- [Email authentication testing](/docs/advanced/email-authentication-testing) - SPF, DKIM, and DMARC for the addresses you send from
- [Proxying connections](/docs/accounts/proxying-connections) - Routing through a proxy instead of a local address
