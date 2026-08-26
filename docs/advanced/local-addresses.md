---
title: Local IP Address Binding
sidebar_position: 11
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

If the server EmailEngine runs on has more than one IP address, EmailEngine can bind its outbound IMAP and SMTP connections to specific addresses instead of leaving the choice to the operating system. This spreads connections across addresses, which matters when a provider counts connections per source IP, and lets an account keep a stable source address.

## Overview

Three settings control the behaviour. All three are runtime settings stored in Redis, changed through the admin UI or the [Settings API](/docs/api/post-v-1-settings):

| Setting | Type | Default | Purpose |
| ------- | ---- | ------- | ------- |
| `localAddresses` | array of IP addresses | `[]` | The pool of local addresses EmailEngine may bind to |
| `imapStrategy` | `default`, `dedicated` or `random` | `default` | How an address is picked for IMAP connections |
| `smtpStrategy` | `default`, `dedicated` or `random` | `default` | How an address is picked for SMTP connections |

The strategies:

- `default` - Let the operating system pick the source address
- `dedicated` - Reuse the same local address for an account, so the remote server sees a stable IP. The address is chosen by rendezvous hashing of the account ID over the pool, so adding or removing an address only moves the accounts that hashed to it
- `random` - Pick a random local address for every connection

The pool is applied to:

- IMAP connections opened by account workers, including sub-connections
- SMTP connections made when [sending a message](/docs/sending/basic-sending) through an account
- The upstream IMAP connections opened by the [IMAP proxy](/docs/configuration/environment-variables#imap-proxy-server)

Gmail API and MS Graph accounts talk HTTPS to the provider and are not affected.

### How an Address Is Chosen

For every connection EmailEngine evaluates the pool in this order:

1. Entries that are not IPv4, or are not configured on a local network interface at that moment, are dropped. IPv6 addresses are accepted by the settings schema but are not used for binding
2. If nothing is left, the operating system default is used
3. If exactly one address is left, it is used regardless of the strategy
4. Otherwise the protocol's strategy decides: `dedicated` or `random` as described above, and `default` hands the choice back to the operating system
5. The chosen address is looked up in the interface list that **Scan for IPs** builds (see [Admin UI](#admin-ui) below). An address that has never been scanned cannot be bound; the connection falls back to the operating system default and the log entry described under [Watch the Selection in the Logs](#watch-the-selection-in-the-logs) records `selector: "error"`

A submitted message can also name an address directly with the `localAddress` field of [`POST /v1/account/{account}/submit`](/docs/api/post-v-1-account-account-submit). That address is used when it is in the scanned interface list and configured on the host, and falls through to the rules above when it is not.

## Prerequisites

The addresses must exist on the host before EmailEngine can bind to them:

```bash
ip addr show
```

Confirm each one can reach the outside world, and that the provider sees the address you expect, by forcing a request out of it:

```bash
curl --interface 192.168.1.101 https://ifconfig.me
```

If a firewall filters outbound traffic by source address, allow ports 993 and 465 (or 143 and 587) from each address in the pool.

## Configuration

### Admin UI

Open **Configuration** > **Network**:

- The **IP Address Strategy** card has a row for each protocol, **IMAP** and **SMTP**, with a **Selection Method** dropdown offering **Dedicated**, **Random** and **Default** (the card's description calls the last one the server default)
- The **Available IP Addresses** card lists the IPv4 addresses found by the last scan. **Scan for IPs** probes each interface on the host, records the public address it reaches the internet from, and stores the result in Redis; tick the addresses EmailEngine may use. Ticked addresses that later disappear from the host are skipped and the server default applies

The scan is the only thing that builds the interface list: EmailEngine does not scan at startup, and neither the Settings API nor `EENGINE_SETTINGS` does it for you. After adding an address to the host, open this page and click **Scan for IPs** once; the list survives restarts.

### Settings API

The same three values through `POST /v1/settings`:

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "localAddresses": ["192.168.1.100", "192.168.1.101", "192.168.1.102"],
    "imapStrategy": "dedicated",
    "smtpStrategy": "dedicated"
  }'
```

The response lists the keys that changed:

```json
{
  "updated": ["localAddresses", "imapStrategy", "smtpStrategy"]
}
```

### Preconfiguring at Startup

These are runtime settings, so they cannot be set in the TOML configuration file. To seed them before the first start, pass them as JSON in the `EENGINE_SETTINGS` environment variable:

```bash
EENGINE_SETTINGS='{"localAddresses":["192.168.1.100","192.168.1.101"],"imapStrategy":"dedicated","smtpStrategy":"dedicated"}' emailengine
```

See [Prepared settings](/docs/configuration/prepared-settings) for how `EENGINE_SETTINGS` is applied. The seeded addresses are still only used once they appear in the scanned interface list, so run **Scan for IPs** on **Configuration** > **Network** after the first start.

## Choosing a Strategy

### Spreading Accounts Across Addresses

Providers that limit concurrent IMAP connections count them per source address as well as per account. With `imapStrategy: "dedicated"` and five addresses, one hundred accounts settle at roughly twenty per address, and each account keeps its address across reconnects:

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "localAddresses": ["192.168.1.100", "192.168.1.101", "192.168.1.102", "192.168.1.103", "192.168.1.104"],
    "imapStrategy": "dedicated",
    "smtpStrategy": "dedicated"
  }'
```

### Rotating Addresses for Sending

For outbound mail where no single address should carry all the volume, keep IMAP on `dedicated` and let SMTP pick a random address per connection:

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "localAddresses": ["192.168.1.100", "192.168.1.101", "192.168.1.102", "192.168.1.103"],
    "imapStrategy": "dedicated",
    "smtpStrategy": "random"
  }'
```

The two strategies are independent, so any combination is valid.

## Verifying Configuration

### Check the Settings

```bash
curl "https://emailengine.example.com/v1/settings?localAddresses=true&imapStrategy=true&smtpStrategy=true" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

```json
{
  "localAddresses": ["192.168.1.100", "192.168.1.101", "192.168.1.102"],
  "imapStrategy": "dedicated",
  "smtpStrategy": "dedicated"
}
```

### Watch the Selection in the Logs

Each IMAP and SMTP connection logs the address it was bound to at the `debug` level, so the process log level must be `debug` or `trace` (the default) for the entry to appear:

```bash
emailengine | jq -c 'select(.msg == "Selected local address")'
```

```text
{"level":20,"time":1697123456789,"pid":12345,"hostname":"server-01","tid":3,"msg":"Selected local address","account":"user123","proto":"IMAP","address":"192.168.1.101","name":"mail.example.com","selector":"dedicated"}
```

`selector` records which rule produced the address: `dedicated`, `random`, `single` (only one usable address in the pool), `hint` (an address named on the submit request), `default` (empty pool), `unknown` (several addresses with the `default` strategy) or `error` (the chosen address is not in the scanned interface list, so the operating system default was used). `name` is the hostname EmailEngine presents in the SMTP `EHLO` greeting, taken from the `smtpEhloName` setting.

### Count Connections per Address

```bash
ss -tn state established '( dport = :993 or dport = :465 )' | awk 'NR > 1 {split($3, a, ":"); print a[1]}' | sort | uniq -c
```

## Updating the Pool

Changing any of the three settings affects connections opened after the change. Existing IMAP sessions keep their address until they reconnect; to move an account immediately, request a reconnect with [`PUT /v1/account/{account}/reconnect`](/docs/api/put-v-1-account-account-reconnect).

## See Also

- [Performance tuning](/docs/advanced/performance-tuning) - Spreading connections across addresses at scale
- [Inbox placement testing](/docs/advanced/inbox-placement-testing) - Checking how a sending address is received
- [Email authentication testing](/docs/advanced/email-authentication-testing) - SPF, DKIM, and DMARC for the addresses you send from
- [Proxying connections](/docs/accounts/proxying-connections) - Routing through a proxy instead of a local address
- [Settings API](/docs/api/post-v-1-settings) - Reading and writing `localAddresses` and the strategies
