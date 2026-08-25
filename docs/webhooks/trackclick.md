---
title: "trackClick"
sidebar_position: 22
description: "Webhook event triggered when a recipient clicks a tracked link in an email"
---

# trackClick

The `trackClick` webhook event is triggered when a recipient clicks a tracked link in an email that has click tracking enabled. This enables you to monitor email engagement and track which links your recipients interact with.

## When This Event is Triggered

The `trackClick` event fires when:

- A recipient clicks a link that has been rewritten for click tracking
- The click redirects through EmailEngine's tracking endpoint before reaching the destination URL
- EmailEngine successfully validates the tracking signature (if service secret is configured)

The tracking works by rewriting all HTTP/HTTPS links in the email's HTML body to redirect through EmailEngine's `/redirect` endpoint. When a link is clicked, EmailEngine records the click event, triggers this webhook, and then redirects the recipient to the original destination URL.

**Important:** Click tracking only works when:
1. The email was sent with HTML content (plain text links are not tracked)
2. Click tracking was enabled when sending the email (via `trackClicks: true` or global tracking settings)
3. The link uses HTTP or HTTPS protocol

## Limitations

Click tracking has some inherent limitations:

- **Plain text emails** - Links in plain text emails cannot be tracked
- **Non-HTTP links** - Links using protocols other than HTTP/HTTPS (e.g., mailto:, tel:) are not tracked
- **Link prefetching** - Some email clients and security scanners prefetch links, potentially triggering false clicks
- **Privacy features** - Link protection features in some email clients may hide the actual clicker or block tracking
- **Link modification detection** - Some security-conscious recipients may notice the modified URLs

EmailEngine attempts to filter out automated requests from known security scanners (such as Google's security scanners) to reduce false positives.

## Common Use Cases

- **Email engagement analytics** - Track click-through rates for marketing campaigns
- **Link performance analysis** - Identify which links in your emails get the most engagement
- **Sales follow-up** - Know when a prospect clicks on a proposal link
- **Content optimization** - Determine which call-to-action buttons perform best
- **A/B testing** - Compare click rates across different email variations
- **User journey tracking** - Understand recipient behavior by tracking link interactions

## Payload Schema

### Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `serviceUrl` | string | No | The configured EmailEngine service URL |
| `account` | string | Yes | Account ID that sent the tracked message |
| `date` | string | Yes | ISO 8601 timestamp when the click was detected |
| `event` | string | Yes | Event type, always "trackClick" for this event |
| `data` | object | Yes | Click tracking data object (see below) |

### Data Object Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `messageId` | string | Yes | Message-ID header of the tracked email |
| `url` | string | Yes | The original destination URL that was clicked |
| `remoteAddress` | string | Yes | IP address of the client that clicked the link |
| `userAgent` | string | No | User-Agent header from the request that clicked the link |

## Example Payload

```json
{
  "serviceUrl": "https://emailengine.example.com",
  "account": "user123",
  "date": "2025-10-17T06:56:27.882Z",
  "event": "trackClick",
  "data": {
    "messageId": "<0ee381d9-581a-2a57-6038-15e64c76f108@example.com>",
    "url": "https://example.com/product-page",
    "remoteAddress": "192.168.1.100",
    "userAgent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/106.0.0.0 Safari/537.36"
  }
}
```

## Example with Minimal Data

When the email client doesn't send a User-Agent header:

```json
{
  "account": "marketing",
  "date": "2025-10-17T14:32:15.000Z",
  "event": "trackClick",
  "data": {
    "messageId": "<campaign-2025-q4-001@marketing.example.com>",
    "url": "https://shop.example.com/promo?code=SAVE20",
    "remoteAddress": "10.0.0.50"
  }
}
```

## Enabling Click Tracking

### Per-Message Tracking

Enable click tracking when sending an email via the submit API:

```bash
curl -X POST "https://emailengine.example.com/v1/account/user123/submit" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from": {
      "name": "Marketing Team",
      "address": "marketing@example.com"
    },
    "to": [
      {
        "address": "customer@company.com"
      }
    ],
    "subject": "Check out our latest products",
    "html": "<p>Hello!</p><p>Visit our <a href=\"https://shop.example.com\">online store</a> for great deals!</p>",
    "trackClicks": true
  }'
```

### Global Tracking Settings

Enable click tracking for all outgoing emails via the Settings API:

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "trackingEnabled": true
  }'
```

Or enable only click tracking (not open tracking):

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "trackOpens": false,
    "trackClicks": true
  }'
```

## Handling the Event

### Basic Handler

```javascript
async function handleTrackClick(event) {
  const { account, date, data } = event;

  console.log(`Link clicked for account ${account}`);
  console.log(`  Message-ID: ${data.messageId}`);
  console.log(`  URL: ${data.url}`);
  console.log(`  Clicked at: ${date}`);
  console.log(`  From IP: ${data.remoteAddress}`);

  // Update your database or analytics system
  await recordLinkClick({
    messageId: data.messageId,
    clickedUrl: data.url,
    clickedAt: new Date(date),
    ipAddress: data.remoteAddress,
    userAgent: data.userAgent
  });
}
```

### Tracking Click Patterns

Analyze which links perform best across your campaigns:

```javascript
async function handleTrackClick(event) {
  const { data, date } = event;

  // Parse the URL to categorize the click
  const clickedUrl = new URL(data.url);

  // Track clicks by domain and path
  await analytics.trackEvent('email_link_click', {
    messageId: data.messageId,
    url: data.url,
    domain: clickedUrl.hostname,
    path: clickedUrl.pathname,
    timestamp: date
  });

  // Track specific call-to-action buttons
  if (clickedUrl.pathname.includes('/buy') || clickedUrl.pathname.includes('/purchase')) {
    await analytics.trackConversion('purchase_click', {
      messageId: data.messageId,
      timestamp: date
    });
  }
}
```

### Correlating Clicks with Sent Messages

Use the `messageId` to link clicks back to your original sent messages:

```javascript
async function handleTrackClick(event) {
  const { data } = event;

  // Find the original message in your database
  const sentMessage = await db.sentMessages.findOne({
    messageId: data.messageId
  });

  if (sentMessage) {
    // Update click stats
    await db.sentMessages.updateOne(
      { messageId: data.messageId },
      {
        $set: { hasClicks: true },
        $inc: { clickCount: 1 },
        $push: {
          clicks: {
            url: data.url,
            clickedAt: new Date(event.date),
            ipAddress: data.remoteAddress
          }
        }
      }
    );

    // Notify sales team if this is a high-value link
    if (data.url.includes('/pricing') || data.url.includes('/demo')) {
      await notifySalesTeam(sentMessage, event);
    }
  }
}
```

## Technical Details

### How Link Tracking Works

When click tracking is enabled, EmailEngine:

1. Parses the HTML content of the outgoing email
2. Finds all anchor tags (`<a>`) with HTTP/HTTPS href attributes
3. Encodes the original URL along with account ID and Message-ID into a signed data payload
4. Rewrites each link to point to EmailEngine's `/redirect` endpoint

A tracked link is transformed like this:

**Original:**
```html
<a href="https://example.com/product">View Product</a>
```

**Tracked:**
```html
<a href="https://emailengine.example.com/redirect?data=...&sig=...">View Product</a>
```

### Excluded Links

EmailEngine does not rewrite certain links to prevent interference with special functionality:

- Links pointing to EmailEngine's own `/unsubscribe` endpoint
- Links already pointing to the `/redirect` endpoint (prevents double-tracking)

### Redirect Flow

When a recipient clicks a tracked link:

1. The browser sends a request to EmailEngine's `/redirect` endpoint
2. EmailEngine decodes and validates the tracking data
3. If a service secret is configured, the signature is verified
4. EmailEngine checks if the request appears to be automated (security scanner)
5. If not automated, the `trackClick` webhook is triggered
6. The recipient is immediately redirected (HTTP 302) to the original destination URL

### Automated Request Filtering

EmailEngine attempts to filter out clicks from automated security scanners to provide more accurate tracking data. Known scanner IP ranges (such as Google's security crawlers) are detected and excluded from triggering webhook events.

When an automated request is detected, the event is logged but no webhook is sent.

## Best Practices

1. **Use with open tracking** - Opens and clicks answer different questions; enable both to tell a read from an act
2. **Handle multiple clicks** - The same link may be clicked multiple times; decide how to count them
3. **Respect privacy** - Be transparent with recipients about tracking and comply with privacy regulations (GDPR, CAN-SPAM)
4. **Secure your tracking** - Configure a service secret to prevent tracking URL tampering
5. **Monitor for anomalies** - Watch for unusual click patterns that might indicate security scanner activity
6. **Test tracked links** - Verify that tracked links redirect correctly to the intended destinations
7. **Consider unsubscribe links** - Ensure unsubscribe functionality works correctly with tracking enabled

## Related Events

- [trackOpen](/docs/webhooks/trackopen) - Triggered when a tracked email is opened
- [messageSent](/docs/webhooks/messagesent) - Triggered when the tracked email was sent

## See Also

- [Webhooks Overview](/docs/webhooks/overview) - Complete webhook setup guide
- [Sending Emails](/docs/sending/basic-sending) - How to send emails with tracking enabled
- [Settings API](/docs/api/post-v-1-settings) - Configure global tracking settings
