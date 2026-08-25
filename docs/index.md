---
title: EmailEngine - Self-Hosted Email API for Developers
sidebar_label: Self-Hosted Email API for Developers
sidebar_position: 0
description: Build email features into your app with EmailEngine's unified REST API. Self-hosted email gateway supporting IMAP, SMTP, Gmail API, and Microsoft Graph. Flat pricing, no per-mailbox fees.
slug: /
keywords:
  - email API
  - IMAP API
  - SMTP API
  - self-hosted email
  - email integration
  - email gateway
  - REST API email
  - Gmail API alternative
  - Nylas alternative
---

import Price from '@site/src/components/Price';

# Self-Hosted Email API for Developers

**EmailEngine** is a self-hosted email API that lets you add email functionality to any application. Access Gmail, Outlook, and any IMAP mailbox through a single REST API. Unlike per-mailbox services like Nylas, EmailEngine uses flat annual pricing<Price /> - connect unlimited accounts for one predictable cost.

## What Can You Do With EmailEngine?

### Send Emails

Send emails through any email provider with a single API endpoint. EmailEngine handles SMTP connections, OAuth2 authentication, retries, and delivery tracking.

```bash
curl -X POST "https://emailengine.example.com/v1/account/user123/submit" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "to": [{ "address": "user@example.com" }],
    "subject": "Hello from EmailEngine",
    "html": "<p>Your message here</p>"
  }'
```

### Receive Emails in Real-Time

Get instant webhook notifications when new emails arrive, no polling required.

```json
{
  "account": "user123",
  "path": "INBOX",
  "event": "messageNew",
  "data": {
    "id": "AAAAAQAACnA",
    "uid": 1838,
    "unseen": true,
    "subject": "Re: Meeting tomorrow",
    "from": { "name": "Ann Client", "address": "client@example.com" },
    "messageId": "<abc123@mail.example.com>"
  }
}
```

[The full payload](/docs/webhooks/messagenew) carries the envelope, the headers, the attachment list, and a reference for fetching the body.

### Manage Email Accounts

Register and manage multiple email accounts with automatic connection handling.

```bash
curl -X POST "https://emailengine.example.com/v1/account" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "user123",
    "email": "user@gmail.com",
    "oauth2": {
      "provider": "AAABlf_0iLgAAAAQ",
      "refreshToken": "1//0gF...",
      "auth": { "user": "user@gmail.com" }
    }
  }'
```

`provider` is the ID of an [OAuth2 application](/docs/accounts/oauth2-setup) registered in EmailEngine, and `refreshToken` comes from that application's authorization flow. EmailEngine can also [run the flow for you](/docs/accounts/hosted-authentication), in which case neither value is yours to supply.

### Search and Organize

Search messages, organize mailboxes, manage flags, and download attachments.

### Connect AI Agents

Expose the connected mailboxes to AI agents over the [Model Context Protocol](/docs/mcp). Point an MCP client at `/mcp`, hand it a narrowed access token, and it can search, read, file, draft and send mail without ever holding a mailbox credential.

```bash
# Register EmailEngine as an MCP server in Claude Code
claude mcp add --transport http emailengine https://emailengine.example.com/mcp \
  --header "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

## Why EmailEngine?

### One API for All Providers

- **IMAP/SMTP** - Works with any email provider
- **Gmail API** - Native Gmail integration with Cloud Pub/Sub
- **Microsoft Graph API** - Native Microsoft 365 and Outlook integration
- **Consistent interface** across all provider types

### Built for SaaS Applications

- **Multi-account** - Manage thousands of email accounts
- **OAuth2 support** - Built-in OAuth2 for Gmail, Google Workspace, Microsoft 365
- **Webhooks** - Real-time notifications for all email events
- **Queue management** - Automatic retries and delivery tracking

### Running It Yourself

- **Self-hosted** - The mail and the credentials stay on your infrastructure
- **Tunable** - Worker counts, connection limits, and Redis sit under your control
- **Self-healing** - Dropped connections are re-established and failed jobs retried
- **Pooled** - Connections are reused across requests rather than opened per call

## Quick Start

Four steps from an empty install to a message in your webhook endpoint:

1. **[Install EmailEngine](/docs/installation)** - Set up with Docker, npm, or on platforms like Render.com
2. **[Add Your First Account](/docs/getting-started/quick-start)** - Register an email account via API
3. **[Send an Email](/docs/sending/basic-sending)** - Submit your first message
4. **[Receive Webhooks](/docs/webhooks/overview)** - Get notified of new emails

## Common Use Cases

### Email-Integrated SaaS

Build email functionality into your SaaS application:

- Send transactional emails from user accounts
- Receive and process incoming emails
- Integrate customer email into your CRM
- [Read the CRM integration guide →](/docs/integrations/crm)

### Email Automation

Automate email workflows:

- Auto-respond to incoming emails
- Forward emails based on rules
- Track email threads and replies
- [Learn about email threading →](/docs/sending/threading)

### Customer Support

Integrate email into your support system:

- Manage multiple support email accounts
- Track email conversations
- Send templated responses
- [Explore mail merge →](/docs/sending/mail-merge)

### Email Analytics

Analyze email communications:

- Track email delivery and opens
- Generate AI-powered email summaries
- Monitor email activity across accounts
- [See AI integration →](/docs/integrations/ai-chatgpt)

## Architecture Overview

EmailEngine works as a middleware between your application and email providers:

![EmailEngine Architecture](/img/diagrams/architecture.svg)

**How it works:**

1. **Your Application** - Makes REST API calls to EmailEngine and receives webhook notifications
2. **EmailEngine** - Maintains persistent connections to email providers and manages data synchronization
3. **Redis** - Stores email metadata, message queues, and account data for fast access
4. **Email Providers** - Gmail, Outlook, Microsoft 365, and any IMAP/SMTP server

**Data flows:**

- **API requests**: Your app calls EmailEngine REST API → EmailEngine connects to email providers or retrieves from Redis
- **Webhooks**: Email providers send updates → EmailEngine processes → Your app receives webhook notifications
- **Data storage**: EmailEngine stores metadata and queues in Redis (email content is not stored, only fetched on demand)

## API Reference

The REST API is documented in four parts, plus the generated endpoint reference:

- **[API Overview](/docs/api-reference)** - Authentication, conventions, error handling
- **[Account Management](/docs/api-reference/accounts-api)** - Register and manage accounts
- **[Sending Emails](/docs/api-reference/sending-api)** - Submit endpoint and options
- **[Message Operations](/docs/api-reference/messages-api)** - List, search, and manage emails
- **[Full endpoint reference](/docs/api/emailengine-api)** - Every endpoint with its request and response schema

## Get Help

- **[Troubleshooting Guide](/docs/troubleshooting)** - Common issues and solutions
- **[GitHub Issues](https://github.com/postalsys/emailengine/issues)** - Report bugs and request features
- **[Support](/docs/support)** - Support channels and what a subscription covers

## System Requirements

- **Node.js** 20 or newer, and only when running from source. The packaged builds carry their own runtime
- **Redis** 6.0 or newer, or a Redis-compatible service such as Upstash
- **Memory** 2 GB to evaluate, 4 to 8 GB for production
- **OS** Linux, macOS, or Windows

## License

EmailEngine requires a license key for production use. Get a license:

- **[14-Day Free Trial](https://postalsys.com/plans)** - Full features, no credit card required
- **[Production License](https://postalsys.com/plans)** - For commercial use<Price />

---

## Next Steps

**New to EmailEngine?**

1. [Read the introduction](/docs/getting-started/introduction) to understand what EmailEngine can do
2. [Follow the quick start guide](/docs/getting-started/quick-start) to get your first email working
3. [Set up OAuth2 for Gmail](/docs/accounts/gmail/gmail-imap) or [Outlook](/docs/accounts/microsoft-365/outlook-365)

**Ready to build?**

- [Explore the API reference](/docs/api-reference) to see all available endpoints
- [Connect an AI agent over MCP](/docs/mcp) to let assistants work with the mailboxes directly
- [Check out integration examples](/docs/integrations) for PHP, CRM, AI, and more
- [Read about performance tuning](/docs/advanced/performance-tuning) for production deployments

**Need inspiration?**

- [See the CRM integration guide](/docs/integrations/crm) for a complete architecture example
- [Explore AI integration](/docs/integrations/ai-chatgpt) for email summarization and automation
- [Compare EmailEngine vs Nylas](/docs/comparison/emailengine-vs-nylas) to understand the differences

---

## Frequently Asked Questions

### What is EmailEngine?

EmailEngine is a self-hosted email gateway that provides a unified REST API for accessing email accounts. It supports IMAP, SMTP, Gmail API, and Microsoft Graph API, letting you build email features into your application without dealing with protocol complexity.

### How is EmailEngine different from Nylas?

EmailEngine is self-hosted with flat annual pricing, while Nylas is a managed service charging per connected mailbox. EmailEngine gives you full data control and becomes more cost-effective at 50+ mailboxes. [See detailed comparison →](/docs/comparison/emailengine-vs-nylas)

### What email providers does EmailEngine support?

EmailEngine works with any email provider: Gmail, Google Workspace, Microsoft 365, Outlook.com, Yahoo, FastMail, and any IMAP/SMTP compatible email service.

### Is EmailEngine free?

EmailEngine offers a 14-day free trial with full functionality. Production use requires an annual license from [postalsys.com/plans](https://postalsys.com/plans)<Price />.

### Can I use EmailEngine to send emails?

Yes. EmailEngine supports sending emails via SMTP or native APIs (Gmail API, Microsoft Graph) with features like attachments, HTML content, templates, mail merge, and delivery tracking.

### Can AI agents use EmailEngine?

Yes. EmailEngine includes an [MCP server](/docs/mcp), so any Model Context Protocol client - Claude Code, Cursor, a claude.ai connector, or your own agent - can work with the connected mailboxes through a curated tool set. Agents authenticate with narrowed EmailEngine access tokens, so you decide which account they reach and whether they may only read or also send.

### Where is my email data stored?

EmailEngine stores only metadata (message IDs, flags, folder structure) in Redis. Email content is fetched on-demand from the original mailbox and is not copied to third-party servers. Your data stays on your infrastructure.
