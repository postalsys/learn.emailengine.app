---
title: Inbox Placement Testing
sidebar_position: 10
description: Track whether emails land in inbox or spam folder by monitoring seed mailboxes with EmailEngine's sub-connections feature
keywords:
  - inbox placement
  - spam testing
  - deliverability
  - seed lists
  - sub-connections
  - gmail categories
---

# Inbox Placement Testing

Measure deliverability by registering seed mailboxes with EmailEngine, sending to them, and reading back where each message landed: the inbox, the spam folder, and for Gmail which category tab.

:::info Placement is not authentication
This page is about where a message ends up. EmailEngine's delivery test (`POST /v1/delivery-test/account/{account}` and `GET /v1/delivery-test/check/{deliveryTest}`) checks SPF, DKIM and DMARC on a message you send to a test address. That is covered on [Email Authentication Testing](/docs/advanced/email-authentication-testing).
:::

## Overview

An inbox placement test sends a message to a list of seed accounts at the providers you care about and then checks, for each account:

- Did the message arrive in **INBOX** or in the **spam** folder?
- For Gmail: which tab? (Primary, Promotions, Social, Updates, Forums)
- How long did delivery take?

The delay between sending and reading back is the part that needs EmailEngine configuration, because by default a change in a spam folder is only noticed on the next poll.

## The Challenge

### Normal IMAP Behavior

An IMAP connection can watch **one folder at a time** for real-time updates (IMAP `IDLE`):

- EmailEngine keeps the primary connection on INBOX, or on "All Mail" for Gmail
- Every other folder is **polled periodically**
- Gmail's "All Mail" **does not include Spam or Trash**, so those are polled as well

That is fine for a normal mailbox and too slow for a placement test.

### The Solution: Sub-Connections

A sub-connection is an additional IMAP session that EmailEngine opens for one specific folder:

- The sub-connection sits in `IDLE` on that folder, so the server pushes changes immediately
- When the folder changes, EmailEngine syncs it and fires the usual webhooks
- Sub-connections are opened after the primary connection is up, and a failed sub-connection does not affect the primary one

Sub-connections are an IMAP feature. Gmail API and MS Graph accounts receive change notifications for every folder from the provider and neither need nor support them.

**Connection limits:** each sub-connection is one more concurrent IMAP session against the account. Google documents a limit of 15 simultaneous IMAP connections per Gmail account, and other providers have limits of their own, so open sub-connections only for the folders the test needs. The rules for which paths EmailEngine accepts as sub-connections, and how to read their state, are on [Performance tuning](/docs/advanced/performance-tuning#sub-connections-for-selected-folders).

## Setting Up Inbox Placement Testing

### 1. Create a Test Account

Register a seed mailbox with EmailEngine and add the spam folder as a sub-connection in the same request:

```bash
curl -X POST "https://emailengine.example.com/v1/account" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -d '{
    "account": "seed-gmail-1",
    "name": "Delivery Test Account",
    "email": "deliverytest@gmail.com",
    "imap": {
      "host": "imap.gmail.com",
      "port": 993,
      "secure": true,
      "auth": {
        "user": "deliverytest@gmail.com",
        "pass": "app-password-here"
      }
    },
    "smtp": {
      "host": "smtp.gmail.com",
      "port": 465,
      "secure": true,
      "auth": {
        "user": "deliverytest@gmail.com",
        "pass": "app-password-here"
      }
    },
    "subconnections": ["\\Junk"]
  }'
```

`subconnections` is an array of folder paths. It can also be set later with `PUT /v1/account/{account}`.

### 2. Using Special-Use Flags

Instead of a folder path, use a special-use flag and let EmailEngine resolve it to whatever the server calls that folder:

```json
{
  "subconnections": ["\\Junk"]
}
```

Special-use flags EmailEngine matches against the folder listing:

- `\\Junk` - the spam folder
- `\\Trash` - deleted items
- `\\Sent` - sent messages
- `\\Drafts` - drafts

### 3. Using Folder Paths

Or name the folder exactly as the server lists it:

```json
{
  "subconnections": ["[Gmail]/Spam"]
}
```

**Gmail folder names:**
- `[Gmail]/Spam` - spam folder
- `[Gmail]/Trash` - trash
- `[Gmail]/All Mail` - the folder the primary connection already watches

**Outlook.com and Exchange folder names:**
- `Junk Email` - spam folder
- `Deleted Items` - trash

On a Gmail account only `\\Trash` and `\\Junk` (or their paths) are accepted, because "All Mail" already covers every other folder. Requesting `[Gmail]/Sent Mail` as a sub-connection leaves a disabled entry with the reason `Covered by the "All Mail" folder`.

### 4. Multiple Sub-Connections

Monitor more than one folder by listing each path:

```bash
curl -X PUT "https://emailengine.example.com/v1/account/seed-gmail-1" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -d '{"subconnections": ["\\Junk", "\\Trash"]}'
```

## Verifying Sub-Connections

The account page in the admin UI lists them:

1. Open **Accounts** and click the seed account
2. Scroll to the **Subconnections** card
3. Each configured path is listed with a state badge. A path EmailEngine did not open shows **Disabled**; hovering over the badge shows why: `Mailbox folder not found`, `Covered by the primary connection`, `Covered by the "All Mail" folder`, or `Can not use the default folder`

![Sub-connections UI](/img/external/Screenshot-2023-02-27-at-11.48.41.png)

The API returns the configured list, not the live state:

```bash
curl "https://emailengine.example.com/v1/account/seed-gmail-1" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  | jq '.subconnections'
```

```json
["\\Junk", "\\Trash"]
```

## Gmail Category Detection

Gmail sorts inbox mail into category tabs:

- **Primary** - personal mail and direct correspondence
- **Social** - social network notifications
- **Promotions** - marketing mail and offers
- **Updates** - confirmations, receipts, statements
- **Forums** - mailing lists and discussion groups

![Gmail Category Tabs](/img/external/Screenshot-2023-02-27-at-11.52.29.png)

### Gmail API vs IMAP

How the category reaches you depends on how the Gmail account is connected:

**Gmail API accounts:** the category comes with the message's labels, so it is always present. Message listings, message details and the `messageNew`, `messageUpdated` and `messageDeleted` webhooks carry a `category` field with one of `primary`, `social`, `promotions`, `updates` or `forums`. Nothing needs to be configured.

**IMAP accounts:** the category is not part of the IMAP data. When enabled, EmailEngine runs a Gmail search (`category:primary`, `category:social`, and so on, in that order) for each new INBOX message on a Gmail account, which is one extra IMAP command per category tried. It is off by default for that reason, and the result appears only in the `messageNew` webhook, not in message listings.

If the seed accounts exist for placement testing only, connecting Gmail through the API avoids the extra IMAP traffic.

### Enable Category Detection (IMAP only)

#### Enable in UI

1. Go to **Configuration** > **Email Processing**
2. Scroll to the **Gmail Features** card
3. Tick **Detect Gmail Categories (IMAP)**
4. Save

![Enable Category Detection](/img/external/Screenshot-2023-02-27-at-11.50.10.png)

#### Enable via Settings API

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"resolveGmailCategories": true}'
```

Or preconfigure it at startup with the `EENGINE_SETTINGS` environment variable:

```bash
EENGINE_SETTINGS='{"resolveGmailCategories":true}' emailengine
```

### Category in Webhooks

With detection enabled, the `messageNew` webhook for a Gmail INBOX message includes `category` inside `data`:

```json
{
  "event": "messageNew",
  "account": "seed-gmail-1",
  "path": "INBOX",
  "data": {
    "id": "AAAABgAAAdk",
    "path": "INBOX",
    "subject": "Special Offer - 50% Off!",
    "category": "promotions",
    "from": {
      "name": "Marketing",
      "address": "marketing@example.com"
    }
  }
}
```

Values on an IMAP account are `primary`, `social`, `promotions`, `updates`, `forums`, `reservations` or `purchases`, the categories EmailEngine tries in turn. Gmail API accounts report the five tab categories only.

## Implementing Inbox Placement Tests

### How the Test Works

An inbox placement test is a loop over a seed list:

1. **Send** the message to each seed address, with a unique marker in the subject so you can find it again.
2. **Wait** for delivery. Give it 30 seconds or more, since providers do not deliver instantly.
3. **Match** the `messageNew` webhook for each seed mailbox against the marker, and read its `path` and `category`.
4. **Score** the result: which folder it landed in, per provider.

The marker matters. Matching on subject alone collides with earlier runs of the same campaign, and a stale hit reports yesterday's placement as today's.

The example below implements all four steps. It sends through an EmailEngine account called `sender` and expects EmailEngine to deliver webhooks for the seed accounts to your application, which calls `process_webhook()` for each one.

### Python Example

```python
import re
import time
from datetime import datetime

import requests


class DeliveryTester:
    def __init__(self, api_url, token):
        self.api_url = api_url
        self.token = token
        self.test_emails = {}

    def send_test_email(self, test_account, subject, content):
        test_id = f"test-{int(time.time() * 1000)}"

        response = requests.post(
            f"{self.api_url}/v1/account/sender/submit",
            headers={
                "Authorization": f"Bearer {self.token}",
                "Content-Type": "application/json",
            },
            json={
                "to": [{"address": test_account}],
                "subject": f"{subject} [Test:{test_id}]",
                "text": content,
            },
        )
        response.raise_for_status()
        data = response.json()

        self.test_emails[test_id] = {
            "message_id": data["messageId"],
            "test_account": test_account,
            "subject": subject,
            "sent_at": datetime.now(),
            "result": None,
        }

        return test_id

    def process_webhook(self, webhook):
        """Call this for every webhook EmailEngine delivers to your application"""
        if webhook["event"] != "messageNew":
            return

        data = webhook["data"]
        match = re.search(r"\[Test:([^\]]+)\]", data.get("subject", ""))
        if not match:
            return

        test = self.test_emails.get(match.group(1))
        if not test:
            return

        path = webhook["path"]
        if path == "INBOX":
            status = "inbox"
        elif "Junk" in path or "Spam" in path:
            status = "spam"
        else:
            status = "other"

        test["result"] = {
            "folder": path,
            "category": data.get("category"),
            "delivered_at": datetime.now(),
            "delivery_time_sec": (datetime.now() - test["sent_at"]).total_seconds(),
            "status": status,
        }

    def run_seed_list_test(self, seed_list, subject, content):
        return [
            {"email": email, "test_id": self.send_test_email(email, subject, content)}
            for email in seed_list
        ]

    def check_results(self, accounts, timeout=30):
        """Wait, then summarise what arrived"""
        time.sleep(timeout)

        completed = []
        pending = []

        for account in accounts:
            test = self.test_emails[account["test_id"]]
            if test["result"]:
                completed.append({"email": account["email"], **test["result"]})
            else:
                pending.append(account["email"])

        inbox_count = sum(1 for r in completed if r["status"] == "inbox")
        spam_count = sum(1 for r in completed if r["status"] == "spam")

        return {
            "total": len(accounts),
            "completed": len(completed),
            "pending": len(pending),
            "inbox_rate": f"{inbox_count / len(completed) * 100:.1f}%" if completed else "N/A",
            "spam_rate": f"{spam_count / len(completed) * 100:.1f}%" if completed else "N/A",
            "results": completed,
        }


tester = DeliveryTester("https://emailengine.example.com", "YOUR_ACCESS_TOKEN")

seed_list = ["test1@gmail.com", "test2@outlook.com", "test3@yahoo.com"]

accounts = tester.run_seed_list_test(
    seed_list, "Marketing Email - Special Offer", "Check out our exclusive deals!"
)

results = tester.check_results(accounts, timeout=30)
print(f"Inbox rate: {results['inbox_rate']}")
print(f"Spam rate: {results['spam_rate']}")
```

## Analyzing Results

### Track Deliverability Metrics

The function below takes the `results` array produced above and breaks it down by provider and, for Gmail, by category:

```javascript
function analyzePlacementResults(results) {
  const stats = {
    total: results.length,
    inbox: 0,
    spam: 0,
    other: 0,
    byProvider: {},
    byCategory: {},
    avgDeliveryTimeSec: 0
  };

  let totalDeliveryTime = 0;

  for (const result of results) {
    stats[result.status]++;

    const domain = result.email.split("@")[1];
    stats.byProvider[domain] = stats.byProvider[domain] || { inbox: 0, spam: 0, other: 0 };
    stats.byProvider[domain][result.status]++;

    if (result.category) {
      stats.byCategory[result.category] = (stats.byCategory[result.category] || 0) + 1;
    }

    totalDeliveryTime += result.delivery_time_sec;
  }

  stats.avgDeliveryTimeSec = results.length ? (totalDeliveryTime / results.length).toFixed(1) : 0;
  stats.inboxRate = results.length ? ((stats.inbox / results.length) * 100).toFixed(1) + "%" : "N/A";
  stats.spamRate = results.length ? ((stats.spam / results.length) * 100).toFixed(1) + "%" : "N/A";

  return stats;
}
```

### Generate Report

```javascript
function generateDeliveryReport(stats) {
  console.log("=== Delivery Test Report ===");
  console.log(`Total: ${stats.total}`);
  console.log(`Inbox: ${stats.inbox} (${stats.inboxRate})`);
  console.log(`Spam: ${stats.spam} (${stats.spamRate})`);
  console.log(`Other: ${stats.other}`);
  console.log(`Average delivery time: ${stats.avgDeliveryTimeSec}s`);

  console.log("--- By provider ---");
  for (const [provider, counts] of Object.entries(stats.byProvider)) {
    const total = counts.inbox + counts.spam + counts.other;
    console.log(`${provider}: ${counts.inbox}/${total} inbox`);
  }

  console.log("--- Gmail categories ---");
  for (const [category, count] of Object.entries(stats.byCategory)) {
    console.log(`${category}: ${count}`);
  }
}
```

## See Also

- [Performance tuning](/docs/advanced/performance-tuning#sub-connections-for-selected-folders) - Which sub-connection paths EmailEngine accepts and what a disabled entry means
- [Email authentication testing](/docs/advanced/email-authentication-testing) - The delivery test endpoints for SPF, DKIM and DMARC
- [messageNew webhook](/docs/webhooks/messagenew) - The payload the test reads `path` and `category` from
- [Gmail via IMAP](/docs/accounts/gmail/gmail-imap) - Connecting a Gmail seed account over IMAP
- [Managing accounts](/docs/accounts/managing-accounts#enable-sub-connections) - Changing `subconnections` on an existing account
