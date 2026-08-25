---
title: Low-Code and No-Code Integrations
sidebar_position: 10
description: Connect EmailEngine with low-code platforms like Zapier, Make.com, n8n, and custom webhook routing
---

# Low-Code and No-Code Integrations

Learn how to integrate EmailEngine with low-code platforms and automation tools using webhooks and custom routing.


## Overview

Anything that can receive a webhook and make an HTTP request can drive EmailEngine, which covers most low-code platforms without writing any code.

### Integration Options

1. **Webhook Routes**: Custom webhook transformations within EmailEngine
2. **Zapier**: Popular automation platform with 5000+ app integrations
3. **Make.com** (Integromat): Visual automation builder
4. **n8n**: Open-source workflow automation
5. **Discord/Slack**: Team notifications
6. **Custom Services**: Any webhook-compatible service

## Webhook Routes Feature

EmailEngine's Webhook Routes feature allows you to set up custom webhook handling in addition to the default webhook handler.

### How It Works

- **Default Handler**: Sends one webhook per event with full EmailEngine payload structure
- **Custom Routes**: Multiple custom routes can be triggered for each event
- **Filtering**: Each route can filter which events to process
- **Transformation**: Convert EmailEngine payloads to match target service requirements

**Example**: A single "new email" event can trigger:
1. Default webhook to your application
2. Discord notification for bounces
3. Slack message for VIP senders
4. Zapier trigger for specific subjects

## Creating a Webhook Route

A webhook route consists of three components:

### 1. Filtering Function

JavaScript function that determines if the route should be processed:

```javascript
// Accept all bounce notifications
if (payload.event === "messageBounce") {
    return true;
}

// Accept emails from specific sender
if (payload.event === "messageNew" &&
    payload.data.from.address === "vip@example.com") {
    return true;
}

// Accept emails with specific subject
if (payload.event === "messageNew" &&
    payload.data.subject.includes("[URGENT]")) {
    return true;
}

// Reject everything else
return false;
```

**Available Features**:
- Top-level `async/await` support
- [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) for external HTTP requests
- All JavaScript standard library functions

**Development Note**: The demo function runner in EmailEngine's web interface runs in browser context, so `fetch()` is limited by CORS. Actual functions run on the server without this limitation.

### 2. Mapping Function

JavaScript function that transforms the webhook payload into the required structure:

```javascript
// Transform to Discord message format
return {
    username: 'EmailEngine',
    content: `Email from [${payload.account}](${payload.serviceUrl}/admin/accounts/${payload.account}) to *${payload.data.recipient}* with Message ID _${payload.data.messageId}_ bounced!`
};
```

**The function should return**:
- Object or string for JSON POST body
- `null` to skip sending the webhook

### 3. Target URL

The webhook endpoint URL of your target service.

**Examples**:
- Discord: `https://discord.com/api/webhooks/123456/abcdef`
- Slack: `https://hooks.slack.com/services/T00/B00/xxxx`
- Zapier: `https://hooks.zapier.com/hooks/catch/123456/abcdef/`
- Custom: `https://yourapp.com/webhook/endpoint`

## Example: Discord Bounce Notifications

### Configure Discord Webhook

1. Open Discord server settings
2. Navigate to Integrations → Webhooks
3. Click "New Webhook"
4. Set name (e.g., "EmailEngine") and channel
5. Copy webhook URL

### Create EmailEngine Route

**Filtering Function**:
```javascript
// Only process bounce events
if (payload.event === "messageBounce") {
    return true;
}
```

**Mapping Function**:
```javascript
return {
    username: 'EmailEngine',
    content: `**Bounce Alert**`,
    embeds: [{
        title: 'Email Bounced',
        color: 16711680, // Red
        fields: [
            {
                name: 'Account',
                value: payload.account,
                inline: true
            },
            {
                name: 'Recipient',
                value: payload.data.recipient,
                inline: true
            },
            {
                name: 'Message ID',
                value: payload.data.messageId || 'N/A',
                inline: false
            },
            {
                name: 'Reason',
                value: payload.data.response?.message || 'Unknown',
                inline: false
            }
        ],
        timestamp: new Date().toISOString()
    }]
};
```

**Target URL**:
```
https://discord.com/api/webhooks/YOUR_WEBHOOK_ID/YOUR_WEBHOOK_TOKEN
```

### Result

When an email bounces, EmailEngine sends a formatted message to Discord with the account, recipient, message ID, and bounce reason displayed in an embedded card format.

## Example: Slack VIP Notifications

Notify team when emails arrive from important contacts:

**Filtering Function**:
```javascript
// List of VIP email addresses
const vips = [
    'ceo@clientcompany.com',
    'important@partner.com',
    'urgent@customer.com'
];

if (payload.event === "messageNew") {
    const fromAddress = payload.data.from.address.toLowerCase();
    return vips.includes(fromAddress);
}

return false;
```

**Mapping Function**:
```javascript
return {
    text: `New email from VIP: ${payload.data.from.name || payload.data.from.address}`,
    blocks: [
        {
            type: 'section',
            text: {
                type: 'mrkdwn',
                text: `*New VIP Email* \n*From:* ${payload.data.from.name || payload.data.from.address}\n*Subject:* ${payload.data.subject}\n*Account:* ${payload.account}`
            }
        },
        {
            type: 'actions',
            elements: [
                {
                    type: 'button',
                    text: {
                        type: 'plain_text',
                        text: 'View in EmailEngine'
                    },
                    url: `${payload.serviceUrl}/admin/accounts/${payload.account}/messages/${payload.data.id}`
                }
            ]
        }
    ]
};
```

**Target URL**: Your Slack incoming webhook URL

## Example: Custom API Integration

Send specific email data to your custom API:

**Filtering Function**:
```javascript
// Only process new invoices
if (payload.event === "messageNew" &&
    payload.data.subject.match(/invoice|receipt/i)) {
    return true;
}
return false;
```

**Mapping Function**:
```javascript
// Extract invoice number from subject
const invoiceMatch = payload.data.subject.match(/INV-(\d+)/);
const invoiceNumber = invoiceMatch ? invoiceMatch[1] : null;

return {
    type: 'invoice_received',
    invoice_number: invoiceNumber,
    from: payload.data.from.address,
    received_at: payload.date,
    account_id: payload.account,
    email_id: payload.data.id,
    has_attachments: payload.data.attachments && payload.data.attachments.length > 0
};
```

**Target URL**: `https://yourapp.com/api/invoices/received`

## Advanced Filtering Examples

### Filter by Account

```javascript
// Only process events for specific accounts
const accountsToMonitor = ['sales@company.com', 'support@company.com'];

if (accountsToMonitor.includes(payload.account)) {
    return true;
}
return false;
```

### Filter by Time

```javascript
// Only process during business hours (9am-5pm UTC)
const date = new Date(payload.date);
const hour = date.getUTCHours();

if (hour >= 9 && hour < 17) {
    return true;
}
return false;
```

### Filter by Folder

```javascript
// Only process emails in specific folders
if (payload.event === "messageNew" &&
    payload.data.messageSpecialUse === "\\Inbox") {
    return true;
}
return false;
```

### Filter by Attachment Type

```javascript
// Only process emails with PDF attachments
if (payload.event === "messageNew" && payload.data.attachments) {
    const hasPDF = payload.data.attachments.some(att =>
        att.filename.toLowerCase().endsWith('.pdf')
    );
    return hasPDF;
}
return false;
```

### Complex Multi-Condition Filtering

```javascript
// Process urgent emails from specific domain during business hours
const urgentKeywords = ['urgent', 'asap', 'emergency'];
const trustedDomain = '@company.com';

if (payload.event === "messageNew") {
    const from = payload.data.from.address || '';
    const subject = (payload.data.subject || '').toLowerCase();
    const hour = new Date(payload.date).getUTCHours();

    const isUrgent = urgentKeywords.some(keyword =>
        subject.includes(keyword)
    );
    const isTrustedSender = from.endsWith(trustedDomain);
    const isBusinessHours = hour >= 9 && hour < 17;

    return isUrgent && isTrustedSender && isBusinessHours;
}

return false;
```

## Advanced Mapping Examples

### Enriching with External Data

```javascript
// Fetch additional data from external API
const response = await fetch(
    `https://api.example.com/users?email=${payload.data.from.address}`
);
const userData = await response.json();

return {
    email_event: 'new_message',
    sender: payload.data.from.address,
    subject: payload.data.subject,
    user_segment: userData.segment,
    user_score: userData.score,
    email_id: payload.data.id
};
```

### Conditional Formatting

```javascript
// Different formats based on email type
const isReply = payload.data.inReplyTo ? true : false;
const priority = payload.data.subject.includes('[URGENT]') ? 'high' : 'normal';

return {
    type: isReply ? 'reply' : 'new_email',
    priority: priority,
    from: payload.data.from.address,
    subject: payload.data.subject,
    preview: payload.data.text?.plain ? payload.data.text.plain.substring(0, 200) : '',
    account: payload.account,
    timestamp: payload.date
};
```

### Extracting Structured Data

```javascript
// Extract phone numbers and URLs from email
// payload.data.text is an object - the plain text content is in text.plain
const text = payload.data.text?.plain || '';

const phoneRegex = /\b\d{3}[-.]?\d{3}[-.]?\d{4}\b/g;
const urlRegex = /https?:\/\/[^\s]+/g;

const phones = text.match(phoneRegex) || [];
const urls = text.match(urlRegex) || [];

return {
    from: payload.data.from.address,
    subject: payload.data.subject,
    extracted_phones: phones,
    extracted_urls: urls,
    full_text: text
};
```

## Integration with Zapier

### Setup Zapier Webhook Trigger

1. Create new Zap in Zapier
2. Select "Webhooks by Zapier" as trigger
3. Choose "Catch Hook"
4. Copy the webhook URL provided
5. Use this URL as target in EmailEngine webhook route

### Create EmailEngine Route for Zapier

**Filtering Function**:
```javascript
// Send all new emails to Zapier
return payload.event === "messageNew";
```

**Mapping Function**:
```javascript
// Zapier-friendly format
return {
    trigger_type: 'new_email',
    account: payload.account,
    from_name: payload.data.from.name || '',
    from_email: payload.data.from.address,
    subject: payload.data.subject || '',
    text_content: (payload.data.text && payload.data.text.plain) || '',
    html_content: (payload.data.text && payload.data.text.html) || '',
    received_date: payload.date,
    has_attachments: !!(payload.data.attachments && payload.data.attachments.length > 0),
    attachment_count: payload.data.attachments ? payload.data.attachments.length : 0,
    email_id: payload.data.id,
    thread_id: payload.data.threadId
};
```

### Zapier Action Examples

After receiving the trigger, you can:

1. **Create Google Sheet Row**: Log emails to spreadsheet
2. **Send Slack Message**: Notify team
3. **Create Trello Card**: Convert emails to tasks
4. **Add to Mailchimp**: Sync email addresses to mailing list
5. **Create Calendar Event**: Extract meetings from emails
6. **Update CRM**: Add contact activity

## Integration with Make.com (Integromat)

### Setup Make Webhook

1. Create new scenario in Make
2. Add "Webhooks" module as first step
3. Select "Custom webhook"
4. Create new webhook and copy URL
5. Use this URL in EmailEngine

### Example Make Scenario

**EmailEngine → Make → Google Sheets + Slack**

1. **Webhook**: Receive email event from EmailEngine
2. **Filter**: Only process emails with attachments
3. **Google Sheets**: Add row with email details
4. **Slack**: Send notification to channel

EmailEngine webhook sends the data, Make handles the rest visually.

## Integration with n8n

n8n is an open-source workflow automation tool.

### Setup n8n Webhook

1. Add "Webhook" node to workflow
2. Set HTTP Method to "POST"
3. Copy webhook URL
4. Use in EmailEngine route

### Example n8n Workflow

```
Webhook → IF (check conditions) → Multiple outputs:
  ├─→ Send Telegram notification
  ├─→ Save to PostgreSQL
  └─→ Call API endpoint
```

n8n builds workflows visually and can branch on arbitrary expressions.

## Testing Webhook Routes

### Using the Test Feature

EmailEngine provides a test interface for webhook routes:

1. Navigate to Webhook Routes in the sidebar
2. Create or edit a webhook route
3. Use the "Test" button
4. Provide sample payload
5. View results (filtered, transformed, sent)

### Testing with Real Events

1. Set up route
2. Trigger actual event (send test email)
3. Check target service received webhook
4. Review EmailEngine logs for errors

### Debug Logging

Check EmailEngine logs for webhook routing:

```bash
# Docker
docker logs emailengine

# SystemD
journalctl -u emailengine
```

EmailEngine logs to stdout using the pino logger. There is no log file - logs are available through your process manager or container runtime.


## Common Use Cases

### 1. Team Notifications

- Bounce alerts to Discord
- VIP email alerts to Slack
- Support email notifications to Teams

### 2. Task Management

- Create Trello cards from emails
- Add tasks to Asana
- Create Jira tickets from support emails

### 3. Data Logging

- Log emails to Google Sheets
- Store in Airtable
- Archive to PostgreSQL

### 4. CRM Integration

- Add contacts to HubSpot
- Update Salesforce records
- Log activities in Pipedrive

### 5. Marketing Automation

- Add to Mailchimp lists
- Trigger ActiveCampaign workflows
- Update ConvertKit subscribers

### 6. Document Management

- Save attachments to Google Drive
- Upload PDFs to Dropbox
- Archive to Box

