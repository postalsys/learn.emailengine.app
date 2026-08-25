---
title: Webhook Overview
sidebar_position: 1
description: "Complete guide to EmailEngine webhooks - setup, event types, testing, debugging, and best practices"
keywords:
  - webhooks
  - real-time notifications
  - webhook events
  - webhook debugging
  - webhook testing
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Webhooks

Webhooks are the primary mechanism for receiving real-time notifications from EmailEngine about mailbox events, message changes, and delivery status. Instead of repeatedly polling for updates, EmailEngine pushes notifications to your application as events occur.

## Why Use Webhooks?

**Real-Time Updates**
- Instant notifications when events occur
- No polling delays or missed events
- Process messages as they arrive

**Efficient**
- Eliminates the need for constant polling
- Reduces API calls and server load
- Lower latency for time-sensitive operations

**Event Coverage**
- Message lifecycle (new, updated, deleted)
- Delivery status (sent, failed, bounced)
- Account status (connected, disconnected, errors)
- User interactions (opens, clicks, unsubscribes)

**Scalable**
- Absorb bursts without dropping events
- Process events asynchronously
- Built on BullMQ for reliability

## Setting Up Webhooks

### 1. Configure Webhook URL

Set your webhook endpoint URL in EmailEngine:

**Via Web UI:**
1. Navigate to **Configuration → Webhooks**
2. Check **Enable webhooks**
3. Enter your **Webhook URL**: `https://your-app.com/webhooks/emailengine`
4. Select which events to receive. Selecting none means no webhooks are sent
5. Click **Update Settings**

![Webhooks configuration page](/img/screenshots/05-webhooks-config.png)
_The Webhooks settings page with the target URL and event selection_

**Via API:**

Use the [settings API](/docs/api/post-v-1-settings) to configure webhooks:

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "webhooks": "https://your-app.com/webhooks/emailengine",
    "webhooksEnabled": true,
    "webhookEvents": ["*"],
    "notifyHeaders": ["List-ID", "X-Priority"],
    "notifyTextSize": 65536,
    "notifyWebSafeHtml": true,
    "notifyCalendarEvents": true
  }'
```

:::warning `webhookEvents` is an allowlist with no default
Nothing is delivered unless the event is named in `webhookEvents`, and an unset `webhookEvents` names nothing. `["*"]` allows every event; a list of names allows exactly those. This is the first thing to check when a correctly configured URL receives nothing.

The allowlist applies to the default webhook target above. [Webhook routes](/docs/webhooks/webhook-routing) carry their own filters and are not affected by it.
:::

:::tip Advanced: Webhook Routes
For more advanced scenarios, you can configure multiple webhook routes to send different events to different endpoints based on account, event type, or custom filtering logic. Webhook routes also support pre-processing functions to filter or transform payloads before delivery.

See [Webhooks API](/docs/api-reference/webhooks-api) for route management and [Pre-Processing Functions](/docs/advanced/pre-processing) for custom filters.
:::

### 2. Create Webhook Handler

Your webhook endpoint must:
- Accept HTTP POST requests
- Return a 2xx status code before the delivery times out (30 seconds by default)
- Process events asynchronously, so that acknowledging never waits on your own work

<Tabs>
<TabItem value="nodejs" label="Node.js" default>

```javascript
const express = require('express');
const app = express();

app.use(express.json());

app.post('/webhooks/emailengine', async (req, res) => {
  const event = req.body;

  // Acknowledge receipt immediately
  res.status(200).json({ success: true });

  // Process asynchronously
  processEvent(event).catch(err => {
    console.error('Webhook processing error:', err);
  });
});

async function processEvent(event) {
  console.log(`Received ${event.event} event for account ${event.account}`);

  switch (event.event) {
    case 'messageNew':
      await handleNewMessage(event);
      break;
    case 'messageSent':
      await handleMessageSent(event);
      break;
    case 'messageFailed':
      await handleMessageFailed(event);
      break;
    // Handle other events...
  }
}

app.listen(3000, () => {
  console.log('Webhook server running on port 3000');
});
```

</TabItem>
<TabItem value="python" label="Python">

```python
from flask import Flask, request, jsonify
import asyncio

app = Flask(__name__)

@app.route('/webhooks/emailengine', methods=['POST'])
def webhook_handler():
    event = request.get_json()

    # Acknowledge immediately
    asyncio.create_task(process_event(event))
    return jsonify({'success': True}), 200

async def process_event(event):
    event_type = event.get('event')
    account = event.get('account')

    print(f"Processing {event_type} for {account}")

    if event_type == 'messageNew':
        await handle_new_message(event)
    elif event_type == 'messageSent':
        await handle_message_sent(event)
    # Handle other events...

if __name__ == '__main__':
    app.run(port=3000)
```

</TabItem>
<TabItem value="php" label="PHP">

```php
<?php
// webhook.php

// Read the webhook payload
$payload = file_get_contents('php://input');
$event = json_decode($payload, true);

// Respond immediately
http_response_code(200);
header('Content-Type: application/json');
echo json_encode(['success' => true]);

// Close connection and process asynchronously
fastcgi_finish_request();

// Process the event
processEvent($event);

function processEvent($event) {
    $eventType = $event['event'] ?? '';
    $account = $event['account'] ?? '';

    error_log("Processing $eventType for $account");

    switch ($eventType) {
        case 'messageNew':
            handleNewMessage($event);
            break;
        case 'messageSent':
            handleMessageSent($event);
            break;
        // Handle other events...
    }
}
?>
```

</TabItem>
</Tabs>

## Delivery and Retries

Webhooks are queued, not sent inline with the mail operation that produced them, so a slow or unavailable endpoint never blocks syncing.

| Behavior | Value |
|----------|-------|
| Success | Any `2xx` response |
| Attempts | 10, counting the first |
| Backoff | Exponential, starting at 5 seconds, with 20% jitter |
| Per-attempt timeout | 30 seconds, configurable with `EENGINE_WEBHOOK_TIMEOUT` |
| After the last attempt | The delivery moves to the **Failed** tab of the Webhooks queue and is not retried again |

Two failures are treated as final and consume the whole retry budget at once, because retrying could not change the outcome: a destination refused by the [egress policy](#blocked-destinations-and-redirects) (`EEGRESSBLOCKED`) and an endpoint that answers with a redirect (`EREDIRECTNOTFOLLOWED`).

:::warning Design your handler for repeat deliveries
Because a timed-out or failed attempt is retried, your endpoint will sometimes receive the same event more than once. Respond `2xx` as soon as you have durably accepted the payload and do the real work afterwards, and deduplicate on the `X-EE-Wh-Event-Id` header. This matters most for handlers with side effects, such as creating a ticket or charging for a message.
:::

## Webhook Events

EmailEngine sends different types of events organized into categories:

### Message Events

Events related to emails in monitored mailbox folders. These webhooks notify you when messages arrive, are modified, or are removed from the mailbox.

#### messageNew

Triggered when a new message is detected in a mailbox folder. This is one of the most commonly used webhook events, enabling real-time processing of incoming emails.

[See full messageNew reference →](/docs/webhooks/messagenew)

#### messageDeleted

Triggered when a previously tracked email has been removed from a mailbox folder. Helps keep external systems synchronized with mailbox state changes.

[See full messageDeleted reference →](/docs/webhooks/messagedeleted)

#### messageUpdated

Triggered when EmailEngine detects that the flags or labels on a message have changed, enabling real-time synchronization of message state changes with external systems.

[See full messageUpdated reference →](/docs/webhooks/messageupdated)

#### messageMissing

Triggered when EmailEngine detects that a message it expected to find on the mail server is not available. This event indicates a potential synchronization issue and helps handle edge cases in message processing.

[See full messageMissing reference →](/docs/webhooks/messagemissing)

### Delivery Events

Events related to outgoing email delivery. These webhooks track the lifecycle of messages sent through EmailEngine, from successful delivery to failures, bounces, and spam complaints.

#### messageSent

Triggered when a queued message is successfully accepted by the SMTP server or email API (Gmail API, Microsoft Graph API). This event confirms that the message has been handed off to the mail transfer agent for delivery.

[See full messageSent reference →](/docs/webhooks/messagesent)

#### messageDeliveryError

Triggered when EmailEngine fails to deliver an email to the SMTP server. This event indicates a temporary failure that may be retried automatically based on the configured retry policy.

[See full messageDeliveryError reference](/docs/webhooks/messagedeliveryerror)

#### messageFailed

Triggered when EmailEngine permanently fails to deliver a queued email after all retry attempts have been exhausted. This is a terminal event indicating that the message will not be delivered.

[See full messageFailed reference](/docs/webhooks/messagefailed)

#### messageBounce

Triggered when a bounce notification (Delivery Status Notification) is received in a monitored mailbox. EmailEngine parses the bounce message to extract delivery failure information including the failed recipient, SMTP error codes, and details about the original message.

[See full messageBounce reference](/docs/webhooks/messagebounce)

#### messageComplaint

Triggered when a feedback loop (FBL) complaint is detected. EmailEngine parses ARF (Abuse Reporting Format) complaint messages to extract information about the complainant and the original message that was reported as spam.

[See full messageComplaint reference](/docs/webhooks/messagecomplaint)

### Mailbox Events

Events related to mailbox folder changes. These webhooks notify you when folders are created, deleted, or reset on the mail server.

#### mailboxNew

Triggered when a new folder is discovered on the mail server during synchronization.

[See full mailboxNew reference](/docs/webhooks/mailboxnew)

#### mailboxDeleted

Triggered when a previously tracked folder is no longer found on the mail server.

[See full mailboxDeleted reference](/docs/webhooks/mailboxdeleted)

#### mailboxReset

Triggered when a folder's UIDVALIDITY changes, indicating a mailbox reset. This is a rare but significant event that invalidates all previously tracked message UIDs in the folder.

[See full mailboxReset reference](/docs/webhooks/mailboxreset)

### Account Events

Events related to email account lifecycle and connection status. These webhooks track account registration, initialization, authentication, and connection health.

#### accountAdded

Triggered when a new email account is registered with EmailEngine. This is the first webhook in the account lifecycle, fired before authentication is attempted.

[See full accountAdded reference](/docs/webhooks/accountadded)

#### accountInitialized

Triggered when an email account completes its initial mailbox synchronization. This marks the point at which the account is fully operational and ready for use.

[See full accountInitialized reference](/docs/webhooks/accountinitialized)

#### accountDeleted

Triggered when an email account is removed from EmailEngine. This is the final webhook event in the account lifecycle.

[See full accountDeleted reference](/docs/webhooks/accountdeleted)

#### authenticationSuccess

Triggered when EmailEngine successfully authenticates an email account for the first time or after recovering from an error state.

[See full authenticationSuccess reference](/docs/webhooks/authenticationsuccess)

#### authenticationError

Triggered when EmailEngine fails to authenticate an email account due to invalid credentials, expired OAuth2 tokens, or API authentication errors.

[See full authenticationError reference](/docs/webhooks/authenticationerror)

#### connectError

Triggered when EmailEngine fails to establish a connection to an email server due to network issues, server unavailability, or TLS/SSL problems. This is distinct from authentication errors which occur after connection is established.

[See full connectError reference](/docs/webhooks/connecterror)

### Tracking Events

Events related to email engagement tracking. These webhooks notify you when recipients open emails, click links, or manage their subscription preferences.

#### trackOpen

Triggered when a recipient opens an email that has open tracking enabled. The tracking works by embedding a 1x1 pixel image in the email's HTML body that is loaded when the email is viewed.

[See full trackOpen reference](/docs/webhooks/trackopen)

#### trackClick

Triggered when a recipient clicks a tracked link in an email that has click tracking enabled. EmailEngine rewrites links in outgoing HTML emails to redirect through a tracking endpoint, capturing click events before redirecting recipients to the original destination.

[See full trackClick reference](/docs/webhooks/trackclick)

#### listUnsubscribe

Triggered when a recipient uses the one-click unsubscribe mechanism to remove themselves from a mailing list. EmailEngine adds the recipient to the suppression list and fires this webhook.

[See full listUnsubscribe reference](/docs/webhooks/listunsubscribe)

#### listSubscribe

Triggered when a recipient re-subscribes to a mailing list after previously unsubscribing. This event enables you to restore subscriptions and keep your mailing lists synchronized.

[See full listSubscribe reference](/docs/webhooks/listsubscribe)

### Export Events

Events related to bulk email export jobs. These webhooks notify you when export jobs complete or fail.

#### exportCompleted

Triggered when a bulk email export job finishes successfully. The export file is ready for download.

[See full exportCompleted reference](/docs/webhooks/exportcompleted)

#### exportFailed

Triggered when a bulk email export job fails. Check the `resumable` field to determine if the export can be continued from checkpoint.

[See full exportFailed reference](/docs/webhooks/exportfailed)

## Testing Webhooks

### Using webhook.site

The easiest way to test webhooks is using a temporary webhook inspector:

1. Visit [https://webhook.site/](https://webhook.site/)
2. Copy your unique webhook URL
3. Set it as the webhook URL in EmailEngine
4. Trigger an event (send a test email, add an account, etc.)
5. View the webhook payload in real-time

### Tailing Webhooks to a Log File

For ongoing webhook monitoring, you can log all webhooks to a file and tail them:

**Step 1: Create log file**

```bash
sudo touch /var/log/emailengine-webhooks.log
sudo chown www-data /var/log/emailengine-webhooks.log  # Adjust user as needed
```

**Step 2: Create PHP webhook logger**

```php
<?php
// webhook-logger.php

$logFile = '/var/log/emailengine-webhooks.log';

// Read webhook payload
$payload = file_get_contents('php://input');
$headers = getallheaders();

// Create log entry
$logEntry = [
    'timestamp' => date('c'),
    'method' => $_SERVER['REQUEST_METHOD'],
    'headers' => $headers,
    'request' => json_decode($payload, true)
];

// Append to log file
file_put_contents($logFile, json_encode($logEntry) . "\n", FILE_APPEND);

// Respond
http_response_code(200);
header('Content-Type: application/json');
echo json_encode(['success' => true]);
?>
```

**Step 3: Tail the log with jq**

```bash
# Install jq if needed
sudo apt update && sudo apt install -y jq

# Tail and pretty-print webhooks
tail -f /var/log/emailengine-webhooks.log | jq
```

This gives you a real-time, pretty-printed view of all incoming webhooks.

### Send Test Webhook

EmailEngine allows you to send a test webhook from the UI:

1. Go to **Configuration → Webhooks**
2. Click **Send test webhook**
3. Check your webhook endpoint receives the test

## Debugging Webhooks

If webhooks aren't working as expected, follow this diagnostic process:

### 1. Verify External Connectivity

**Test with webhook.site:**
1. Set webhook URL to https://webhook.site/your-unique-id
2. Trigger an event in EmailEngine
3. Check if webhook.site receives the request

If no request appears:
- Check firewall rules
- Verify DNS resolution
- Ensure EmailEngine can make outbound HTTPS requests
- Check for typos in webhook URL
- Check whether EmailEngine refused the destination itself, see below

#### Blocked destinations and redirects

EmailEngine will not deliver to every address. Two refusals come from EmailEngine rather than from the network, and both report themselves in the webhook error flag on the account or configuration page:

| Error code | Meaning | Fix |
|------------|---------|-----|
| `EEGRESSBLOCKED` | The destination resolves to an address the egress policy blocks. By default that is the link-local range where cloud instance metadata services live | Point the webhook at a routable address, or widen `EENGINE_WEBHOOK_EGRESS_POLICY` |
| `EREDIRECTNOTFOLLOWED` | The endpoint answered with a redirect (301, 302, 307, ...). Redirects are not followed, because a permitted host could redirect to a blocked one | Configure the webhook with the endpoint's final URL |

Both fail immediately without consuming the retry budget, since the same address would be refused, and the same endpoint would redirect, on every attempt.

Both checks apply to the **Send test webhook** button as well, so the button reports the same refusal you would see on a real delivery. Set `EENGINE_WEBHOOK_EGRESS_POLICY=off` to restore the previous behavior, including following redirects.

See [Webhook Delivery settings](/docs/configuration/environment-variables#webhook-delivery) for the available policies.

### 2. Monitor Webhook Queue

EmailEngine uses BullMQ to manage webhook delivery. To inspect webhook jobs:

1. Go to **System → Queues**
2. Select **Webhooks** queue
3. Check these tabs:
   - **Active**: Currently processing
   - **Delayed**: Failed but will retry
   - **Failed**: Exceeded retry limit
   - **Completed**: Successfully delivered (if retention enabled)

Failed webhooks are retained with full error details by default, so there is nothing to enable before inspecting them. To also keep successful deliveries, go to **Configuration → General** and set **Job History Limit** to, for example, 100. See [Queue Management](/docs/advanced/queue-management#enable-job-retention).

### 3. Inspect Failed Jobs

Click on a failed job in Bull Board to see:
- Complete error stack trace
- Request headers sent
- Payload data
- Response from your server
- Retry attempts

Common errors:
- **Connection timeout**: Your server is unreachable
- **SSL/TLS error**: Certificate issues
- **4xx status**: Your server rejected the webhook
- **5xx status**: Your server had an internal error
- **JSON parse error**: Your server returned invalid JSON

### 4. Verify Event Generation

Test if events are being generated at all:

**Add a new account:**
- Should trigger `accountAdded` and `accountInitialized` events

**Send test email to an account:**
- Should trigger `messageNew` within 10-60 seconds
- If not, verify message arrived (check via webmail)
- Check message is visible via API:

```bash
curl "https://emailengine.example.com/v1/account/ACCOUNT_ID/messages?path=INBOX" \
  -H "Authorization: Bearer TOKEN"
```

[List messages API →](/docs/api/get-v-1-account-account-messages)

If message is missing:
- Wrong account credentials
- OAuth token lacks required scopes
- Message filtered to different folder

### 5. Special Requirements for API Backends

#### Gmail API + Cloud Pub/Sub

If using Gmail API (not IMAP):

1. Go to **Integrations → OAuth2 Apps**
2. Select your Gmail OAuth app
3. Scroll to **Cloud Pub/Sub configuration**
4. Verify all show **Created** (in green):
   - Topic
   - Subscription
   - Gmail bindings

If not created:
- Google Cloud service account missing IAM roles
- Pub/Sub API not enabled
- Invalid credentials

#### Microsoft Graph API

If using MS Graph (not IMAP):

1. Go to **Accounts**
2. Select the account
3. Scroll to **Change subscription**
4. Verify status is **Created** and expiration is in future

If not created:
- EmailEngine not reachable from Microsoft servers
- TLS certificate invalid
- Service URL not configured correctly
- OAuth app missing required scopes

Microsoft Graph requires these endpoints to be publicly accessible:
```
https://YOUR-EMAILENGINE-HOST/oauth/msg/lifecycle
https://YOUR-EMAILENGINE-HOST/oauth/msg/notification
```

## Webhook Security

### Verify Webhook Authenticity

Every webhook is signed with HMAC-SHA256 over the raw request body, sent as `X-EE-Wh-Signature`. The key is the `serviceSecret` setting, which EmailEngine generates automatically the first time it needs one, so the header is present whether or not you configured anything. Set your own value so that you know the secret your handler should verify against.

**1. Set a service secret using the [settings API](/docs/api/post-v-1-settings):**

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"serviceSecret": "your-secret-key-here"}'
```

**2. Verify signature in your handler:**

The signature is computed on the raw request body using HMAC-SHA256 and encoded as base64url.

```javascript
const crypto = require('crypto');

function verifyWebhookSignature(rawBody, signature, secret) {
  const expected = crypto
    .createHmac('sha256', secret)
    .update(rawBody)
    .digest('base64url');

  // Compare in constant time so the comparison cannot be used as an oracle
  const a = Buffer.from(expected);
  const b = Buffer.from(signature || '', 'utf8');

  return a.length === b.length && crypto.timingSafeEqual(a, b);
}

// Important: Use raw body parser to get the exact bytes for signature verification
app.post('/webhooks/emailengine', express.raw({ type: 'application/json' }), (req, res) => {
  const signature = req.headers['x-ee-wh-signature'];
  const secret = process.env.SERVICE_SECRET;

  if (!verifyWebhookSignature(req.body, signature, secret)) {
    return res.status(401).json({ error: 'Invalid signature' });
  }

  // Parse body after verification
  const event = JSON.parse(req.body.toString());

  // Process webhook...
  res.json({ success: true });
});
```

## Advanced Webhook Settings

### Inbox-Only Webhooks (inboxNewOnly)

By default, EmailEngine triggers `messageNew` webhooks for new messages in all monitored folders. If you only care about incoming mail, enable the `inboxNewOnly` setting to limit `messageNew` webhooks to messages arriving in the Inbox folder only.

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"inboxNewOnly": true}'
```

When enabled:
- `messageNew` webhooks are only triggered for messages in the Inbox
- Messages arriving in Sent, Drafts, Junk, Trash, and other folders are silently ignored
- Other webhook events (`messageDeleted`, `messageUpdated`, etc.) are not affected

This is useful for reducing webhook volume when you only need to process incoming emails.

### Webhook Error Tracking (webhookErrorFlag)

EmailEngine automatically tracks webhook delivery errors. When a webhook delivery fails, the error details are stored and displayed in the admin panel on the account details page. When a subsequent webhook delivery succeeds, the error flag is automatically cleared.

The error flag includes:
- Event type that failed
- Error message
- Webhook URL
- Error code and HTTP status code
- Timestamp

This is an internal tracking mechanism - there is no configuration needed. Check the admin panel or account details API response for the current webhook error status.

### Use HTTPS

Always use HTTPS for webhook URLs to prevent:
- Man-in-the-middle attacks
- Credential exposure
- Data tampering

### Webhook HTTP Headers

EmailEngine includes the following HTTP headers with each webhook request:

| Header | Description |
|--------|-------------|
| `X-EE-Wh-Event-Id` | Unique identifier for the event. Use for deduplication. Present whenever the payload carries an event ID, and moved out of the JSON body into this header. |
| `X-EE-Wh-Signature` | HMAC-SHA256 signature of the request body, base64url encoded. Always sent, see [Verify Webhook Authenticity](#verify-webhook-authenticity). |
| `X-EE-Wh-Id` | Queue job ID for this delivery. Identifies the job in **System > Queues**. |
| `X-EE-Wh-Attempts-Made` | How many attempts have already been made for this delivery. `0` on the first attempt. |
| `X-EE-Wh-Queued-Time` | How long the event waited in the queue before this attempt, in seconds (for example `3s`). |
| `X-EE-Wh-Custom-Route` | ID of the [custom route](/docs/webhooks/webhook-routing) that produced this delivery. Only sent for route deliveries. |
| `User-Agent` | `emailengine/<version>` plus the project URL |
| `Content-Type` | Always `application/json` |
| `Content-Length` | Size of the request body in bytes |

Any [custom headers](/docs/webhooks/webhook-routing) configured globally, on a route, or on an account are added to this set.

**Using the Event ID for Deduplication:**

```javascript
const processedEvents = new Set();

app.post('/webhooks/emailengine', (req, res) => {
  const eventId = req.headers['x-ee-wh-event-id'];

  // Skip if already processed (idempotency)
  if (processedEvents.has(eventId)) {
    return res.json({ success: true, skipped: true });
  }

  processedEvents.add(eventId);

  // Process the event...
  res.json({ success: true });
});
```

## See Also

- [Webhook events reference](/docs/reference/webhook-events) - Every event and its payload in one table
- [Webhook routing](/docs/webhooks/webhook-routing) - Sending different events to different endpoints
- [Pre-processing functions](/docs/advanced/pre-processing) - Filtering or reshaping a payload before delivery
- [Queue management](/docs/advanced/queue-management) - Watching and draining the notify queue
- [Webhooks API](/docs/api-reference/webhooks-api) - Managing routes programmatically
