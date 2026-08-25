---
title: Inbox Placement Testing
sidebar_position: 8
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

Measure email deliverability by monitoring test mailboxes with EmailEngine. Track whether your emails land in the inbox or spam folder, and for Gmail, which category tab they appear in.

## Overview

Common use case: Send test emails to seed list accounts and immediately check:

- Did the email arrive in **INBOX** or **Spam**?
- For Gmail: Which tab? (Primary, Promotions, Social, Updates, Forums)
- How long did delivery take?
- Were there any delivery issues?

EmailEngine's **sub-connections** feature enables real-time monitoring of multiple folders simultaneously, eliminating polling delays for instant delivery results.

## The Challenge

### Normal IMAP Behavior

IMAP clients can only subscribe to **one folder at a time** for real-time updates:

- EmailEngine subscribes to INBOX (or Gmail's "All Mail") for live updates
- Other folders (Spam, Junk, etc.) are **polled periodically**
- Polling introduces delays (typically 1-5 minutes)
- Gmail's "All Mail" **doesn't include Spam or Trash folders**, requiring polling

This is fine for normal operations but problematic for delivery testing where you need **immediate results**.

### The Solution: Sub-Connections

Sub-connections allow EmailEngine to establish **additional IMAP sessions** for specific folders:

- Each sub-connection monitors a specific folder in real-time
- IMAP servers notify EmailEngine about changes immediately
- No polling delay - instant delivery detection
- Perfect for spam folder monitoring

**Important Limitation:** Email providers heavily limit parallel IMAP connections:
- **Gmail:** 15 concurrent connections max
- **Outlook:** 10-15 concurrent connections
- **Yahoo:** Similar limits

Use sub-connections sparingly. EmailEngine prioritizes the primary connection and only establishes sub-connections after the main session is stable.

## Setting Up Inbox Placement Testing

### 1. Create Test Account

Register a test email account (Gmail, Outlook, etc.) with EmailEngine:

```bash
curl -XPOST "http://localhost:3000/v1/account" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "account": "test-account",
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
    "subconnections": ["\\Junk"]
  }'
```

**Key field:** `subconnections` array specifies additional folders to monitor.

### 2. Using Special-Use Flags

Instead of folder paths, use **special-use flags** (automatically resolved):

```json
{
  "subconnections": ["\\Junk"]
}
```

Common special-use flags:
- `\\Junk` - Spam folder
- `\\Trash` - Deleted items
- `\\Sent` - Sent messages
- `\\Drafts` - Draft messages

EmailEngine resolves these to actual folder paths automatically.

### 3. Using Folder Paths

Or specify exact folder paths:

```json
{
  "subconnections": ["[Gmail]/Spam", "Promotions"]
}
```

**Gmail folder names:**
- `[Gmail]/Spam` - Spam folder
- `[Gmail]/All Mail` - All mail (default primary connection)
- `[Gmail]/Sent Mail` - Sent items
- `[Gmail]/Trash` - Trash

**Outlook folder names:**
- `Junk Email` - Spam folder
- `Deleted Items` - Trash
- `Sent Items` - Sent mail

### 4. Multiple Sub-Connections

Monitor multiple folders simultaneously:

```bash
curl -XPOST "http://localhost:3000/v1/account" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "account": "test-account",
    "email": "test@ethereal.email",
    "imap": {
      "host": "imap.ethereal.email",
      "port": 993,
      "secure": true,
      "auth": {
        "user": "test@ethereal.email",
        "pass": "password123"
      }
    },
    "subconnections": [
      "\\Junk",
      "\\Trash"
    ]
  }'
```

## Verifying Sub-Connections

Check which sub-connections are active in the EmailEngine UI:

1. Navigate to **Accounts**
2. Click on test account
3. Scroll to **IMAP** section
4. View **Sub-connections** list

![Sub-connections UI](/img/external/Screenshot-2023-02-27-at-11.48.41.png)

**Note:** EmailEngine skips duplicate connections. If you request INBOX as a sub-connection and EmailEngine already monitors it as the primary connection, it won't appear in the sub-connections list.

You can also check via API:

```bash
curl "http://localhost:3000/v1/account/test-account" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  | jq '.subconnections'
```

Response (returns the configured folder paths as an array):

```json
["[Gmail]/Spam", "\\Junk"]
```

## Gmail Category Detection

Gmail sorts inbox emails into category tabs:

- **Primary** - Personal emails, direct correspondence
- **Social** - Social media notifications
- **Promotions** - Marketing emails, offers
- **Updates** - Confirmations, receipts, statements
- **Forums** - Mailing lists, discussion groups

![Gmail Category Tabs](/img/external/Screenshot-2023-02-27-at-11.52.29.png)

### Gmail API vs IMAP

Category detection behavior depends on how the Gmail account is connected:

**Gmail API accounts:** Category information is **always available** automatically. The Gmail API provides category labels directly as part of the message metadata, so no additional configuration is needed.

**IMAP accounts:** Category detection must be **explicitly enabled** (see below). EmailEngine uses advanced IMAP commands to detect categories, which adds overhead. This is why it's disabled by default for IMAP connections.

If you're setting up accounts specifically for inbox placement testing and need category detection, consider using Gmail API authentication instead of IMAP for automatic category support.

### Enable Category Detection (IMAP only)

For IMAP-connected Gmail accounts, category detection requires **additional IMAP commands per email**, which can slow processing for high-volume accounts. Enable only when needed.

#### Enable in UI

1. Go to **Configuration** > **Email Processing**
2. Scroll to the **Gmail Features** section
3. Enable **Detect Gmail Categories (IMAP)**
4. Save changes

![Enable Category Detection](/img/external/Screenshot-2023-02-27-at-11.50.10.png)

#### Enable via Settings API

```bash
curl -X POST "http://localhost:3000/v1/settings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"resolveGmailCategories": true}'
```

Or pre-configure at startup using the `EENGINE_SETTINGS` environment variable:

```bash
EENGINE_SETTINGS='{"resolveGmailCategories":true}' node server.js
```

### Category in Webhooks

Once enabled, `messageNew` webhooks for Gmail INBOX emails include a `category` field:

```json
{
  "event": "messageNew",
  "account": "test@gmail.com",
  "data": {
    "id": "AAAABgAAAdk",
    "path": "INBOX",
    "subject": "Special Offer - 50% Off!",
    "category": "promotions",
    "from": {
      "address": "marketing@example.com"
    }
  }
}
```

Possible `category` values:
- `primary`
- `social`
- `promotions`
- `updates`
- `forums`
- `reservations`
- `purchases`

## Implementing Inbox Placement Tests

### How the Test Works

An inbox placement test is a loop over a seed list:

1. **Send** the message to each seed address, with a unique marker in the subject so you can find it again.
2. **Wait** for delivery. Give it 30 seconds or more, since providers do not deliver instantly.
3. **Search** each seed mailbox for that marker, across INBOX and Junk.
4. **Score** the result: which folder it landed in, per provider.

The marker matters. Searching by subject alone collides with earlier runs of the same campaign, and a stale hit reports yesterday's placement as today's.

The example below implements all four steps.

### Python Example

```python
import requests
import time
from datetime import datetime

class DeliveryTester:
    def __init__(self, api_url, token):
        self.api_url = api_url
        self.token = token
        self.test_emails = {}

    def send_test_email(self, test_account, subject, content):
        test_id = f"test-{int(time.time() * 1000)}"

        # Send email
        response = requests.post(
            f"{self.api_url}/v1/account/sender@example.com/submit",
            headers={
                'Authorization': f'Bearer {self.token}',
                'Content-Type': 'application/json'
            },
            json={
                'to': {'address': test_account},
                'subject': f"{subject} [Test:{test_id}]",
                'text': content
            }
        )

        data = response.json()

        # Store test metadata
        self.test_emails[test_id] = {
            'message_id': data['messageId'],
            'test_account': test_account,
            'subject': subject,
            'sent_at': datetime.now(),
            'result': None
        }

        return test_id

    def process_webhook(self, webhook):
        """Process incoming webhook"""
        if webhook['event'] == 'messageNew':
            data = webhook['data']
            subject = data.get('subject', '')

            # Extract test ID
            import re
            match = re.search(r'\[Test:([^\]]+)\]', subject)

            if match:
                test_id = match.group(1)
                test = self.test_emails.get(test_id)

                if test:
                    delivery_time = (datetime.now() - test['sent_at']).total_seconds()

                    # Determine status
                    path = data['path']
                    if path == 'INBOX':
                        status = 'inbox'
                    elif 'Junk' in path or 'Spam' in path:
                        status = 'spam'
                    else:
                        status = 'other'

                    test['result'] = {
                        'folder': path,
                        'category': data.get('category'),
                        'delivered_at': datetime.now(),
                        'delivery_time_sec': delivery_time,
                        'status': status
                    }

                    print(f"Test {test_id} result: {status} in {delivery_time:.2f}s")

    def run_seed_list_test(self, seed_list, subject, content):
        """Test multiple accounts"""
        results = {
            'test_id': f"seed-{int(time.time())}",
            'sent_at': datetime.now(),
            'accounts': []
        }

        for email in seed_list:
            test_id = self.send_test_email(email, subject, content)
            results['accounts'].append({
                'email': email,
                'test_id': test_id
            })

        return results

    def check_results(self, test_results, timeout=30):
        """Check test results after timeout"""
        time.sleep(timeout)

        completed = []
        pending = []

        for account in test_results['accounts']:
            test = self.test_emails[account['test_id']]

            if test['result']:
                completed.append({
                    'email': account['email'],
                    **test['result']
                })
            else:
                pending.append(account['email'])

        # Calculate statistics
        inbox_count = sum(1 for r in completed if r['status'] == 'inbox')
        spam_count = sum(1 for r in completed if r['status'] == 'spam')

        return {
            'total': len(test_results['accounts']),
            'completed': len(completed),
            'pending': len(pending),
            'inbox_rate': f"{inbox_count / len(completed) * 100:.1f}%" if completed else "N/A",
            'spam_rate': f"{spam_count / len(completed) * 100:.1f}%" if completed else "N/A",
            'results': completed
        }

# Usage
tester = DeliveryTester('http://localhost:3000', 'YOUR_TOKEN')

seed_list = [
    'test1@gmail.com',
    'test2@outlook.com',
    'test3@yahoo.com'
]

test_results = tester.run_seed_list_test(
    seed_list,
    'Marketing Email - Special Offer',
    'Check out our exclusive deals!'
)

# Wait and check results
results = tester.check_results(test_results, timeout=30)
print(f"Inbox rate: {results['inbox_rate']}")
print(f"Spam rate: {results['spam_rate']}")
```

## Analyzing Results

### Track Deliverability Metrics

```javascript
async function analyzePlacementResults(testResults) {
  const stats = {
    total: testResults.length,
    inbox: 0,
    spam: 0,
    other: 0,
    byProvider: {},
    byCategory: {
      primary: 0,
      promotions: 0,
      social: 0,
      updates: 0,
      forums: 0
    },
    avgDeliveryTime: 0
  };

  let totalDeliveryTime = 0;

  for (const result of testResults) {
    // Count by status
    stats[result.status]++;

    // Count by provider
    const domain = result.email.split('@')[1];
    stats.byProvider[domain] = stats.byProvider[domain] || { inbox: 0, spam: 0 };
    stats.byProvider[domain][result.status]++;

    // Count by category (Gmail)
    if (result.category) {
      stats.byCategory[result.category]++;
    }

    // Sum delivery times
    totalDeliveryTime += parseFloat(result.deliveryTime);
  }

  stats.avgDeliveryTime = (totalDeliveryTime / stats.total).toFixed(2) + 'ms';
  stats.inboxRate = ((stats.inbox / stats.total) * 100).toFixed(1) + '%';
  stats.spamRate = ((stats.spam / stats.total) * 100).toFixed(1) + '%';

  return stats;
}
```

### Generate Report

```javascript
function generateDeliveryReport(stats) {
  console.log('\n=== Delivery Test Report ===\n');
  console.log(`Total Emails: ${stats.total}`);
  console.log(`Inbox: ${stats.inbox} (${stats.inboxRate})`);
  console.log(`Spam: ${stats.spam} (${stats.spamRate})`);
  console.log(`Other: ${stats.other}`);
  console.log(`Avg Delivery Time: ${stats.avgDeliveryTime}`);

  console.log('\n--- By Provider ---');
  for (const [provider, counts] of Object.entries(stats.byProvider)) {
    const total = counts.inbox + counts.spam;
    const rate = ((counts.inbox / total) * 100).toFixed(1);
    console.log(`${provider}: ${counts.inbox}/${total} inbox (${rate}%)`);
  }

  console.log('\n--- Gmail Categories ---');
  for (const [category, count] of Object.entries(stats.byCategory)) {
    if (count > 0) {
      console.log(`${category}: ${count}`);
    }
  }
}
```

## See Also

- [Email Authentication Testing](/docs/advanced/email-authentication-testing) - Verify DKIM, SPF, DMARC configuration
- [Webhooks](/docs/webhooks/overview) - Configure webhook notifications
- [Gmail OAuth2](/docs/accounts/gmail/gmail-imap) - Set up Gmail accounts for testing
