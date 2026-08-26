---
title: Email API - REST API for Email Integration
sidebar_label: REST API for Email Integration
description: Add email functionality to your app with EmailEngine's REST API. Send emails, receive webhooks, manage mailboxes. Self-hosted with flat pricing.
sidebar_position: 98
slug: /email-api
keywords:
  - email API
  - REST API for email
  - email integration API
  - send email API
  - receive email API
  - email webhook API
  - email API for developers
---

import Price from '@site/src/components/Price';

# Email API for Application Integration

EmailEngine exposes a **REST API for email integration**. Add sending, receiving, and mailbox management to an application without implementing IMAP or SMTP.

## Email API Features

### Send Emails via API

Send emails through any provider with one REST endpoint. EmailEngine handles the SMTP connection, OAuth2 authentication, retries, and delivery tracking.

```bash
curl -X POST "https://emailengine.example.com/v1/account/user123/submit" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "to": [{ "address": "user@example.com", "name": "John Doe" }],
    "subject": "Hello from EmailEngine",
    "html": "<p>Your message content</p>",
    "attachments": [
      { "filename": "hello.txt", "contentType": "text/plain", "content": "SGVsbG8gZnJvbSBFbWFpbEVuZ2luZQo=" }
    ]
  }'
```

`content` is base64 by default; the [Sending API](/docs/api-reference/sending-api#attachments) covers the other encodings and referencing an attachment from an existing message.

### Receive Emails in Real-Time

Webhooks fire when emails arrive. No polling is required: EmailEngine keeps a connection open to each mailbox and pushes events to your application.

```json
{
  "account": "support-inbox",
  "path": "INBOX",
  "event": "messageNew",
  "data": {
    "id": "AAAAAQAACnA",
    "uid": 1838,
    "unseen": true,
    "subject": "Re: Your inquiry",
    "from": { "name": "Jane Smith", "address": "client@example.com" },
    "to": [{ "name": "Support", "address": "support@yourapp.com" }]
  }
}
```

See [messageNew](/docs/webhooks/messagenew) for every field the event carries.

### Manage Email Accounts

Register and manage multiple email accounts through the API. Gmail, Microsoft 365, and any IMAP/SMTP provider are supported, with reconnection handled by EmailEngine.

```bash
curl -X POST "https://emailengine.example.com/v1/account" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "support-inbox",
    "name": "Support Inbox",
    "email": "support@yourcompany.com",
    "imap": {
      "host": "imap.yourprovider.com",
      "port": 993,
      "secure": true,
      "auth": { "user": "support@yourcompany.com", "pass": "app-password" }
    },
    "smtp": {
      "host": "smtp.yourprovider.com",
      "port": 465,
      "secure": true,
      "auth": { "user": "support@yourcompany.com", "pass": "app-password" }
    }
  }'
```

### Search and Organize

Search messages, manage folders, update flags, and download attachments through the same REST API.

```bash
# Search a folder. The terms go in the request body, not the query string
curl -X POST "https://emailengine.example.com/v1/account/user123/search?path=INBOX" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "search": { "subject": "invoice", "from": "billing@" } }'

# List messages in a folder
curl "https://emailengine.example.com/v1/account/user123/messages?path=INBOX&page=0&pageSize=20" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"

# Download an attachment
curl "https://emailengine.example.com/v1/account/user123/attachment/AAAAAQAACnAy" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -o invoice.pdf
```

## Why Choose EmailEngine's Email API?

| Feature | EmailEngine | Per-Mailbox APIs (Nylas, etc.) |
|---------|-------------|-------------------------------|
| **Pricing** | Flat annual fee<Price /> | Per mailbox/month |
| **Data Location** | Your servers | Third-party cloud |
| **Webhooks** | Real-time | Real-time |
| **Account Limits** | Unlimited | Based on plan |
| **Setup** | Self-hosted | Managed service |

The difference that matters at scale is the shape of the bill rather than its size: a per-mailbox service charges for each mailbox you connect, and EmailEngine charges one annual fee<Price /> whatever the count. Where the crossover falls depends on the current price of both, so check each vendor's own page.

[Compare EmailEngine vs Nylas →](/docs/comparison/emailengine-vs-nylas)

## Supported Email Providers

EmailEngine works with any email service:

- **Gmail & Google Workspace** - OAuth2 or Gmail API
- **Microsoft 365 & Outlook.com** - OAuth2 or Microsoft Graph API
- **Yahoo Mail** - IMAP/SMTP with an app password
- **FastMail** - IMAP/SMTP with an app password
- **Proton Mail** - Via the Proton Mail Bridge
- **Any IMAP/SMTP server** - Standard protocol support

## API Capabilities

### Core Operations
- **Send emails** - Single emails, bulk sending, mail merge
- **Receive emails** - Real-time webhooks, message listing
- **Search** - Full-text and header-based search
- **Attachments** - Upload, download, inline images
- **Threading** - Conversation tracking where the provider exposes it (Gmail, Microsoft Graph, Yahoo and AOL); see [provider support](/docs/sending/threading/provider-support)

### Account Management
- **OAuth2 flows** - Built-in authorization for Gmail and Microsoft
- **Connection handling** - Reconnection with backoff after a dropped connection
- **Multi-account** - One instance serves many accounts; see [performance tuning](/docs/advanced/performance-tuning) for sizing

### Advanced Features
- **Bounce detection** - Automatic bounce and complaint handling
- **Delivery tracking** - Open and click tracking
- **Templates** - Mail merge with variable substitution
- **Scheduling** - Delayed sending

## Get Started

The examples above address a deployed instance at `emailengine.example.com`. The walkthrough below runs one locally, so it calls `http://localhost:3000`.

### 1. Install EmailEngine

```bash
# Docker (quickest)
docker run -p 3000:3000 \
  --env EENGINE_REDIS="redis://host.docker.internal:6379/8" \
  postalsys/emailengine:v2

# Or download binary
wget https://go.emailengine.app/emailengine.tar.gz
tar xzf emailengine.tar.gz
./emailengine
```

[Full installation guide →](/docs/installation)

### 2. Register an Email Account

```bash
curl -X POST http://localhost:3000/v1/account \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "my-account",
    "name": "My Gmail Account",
    "email": "user@gmail.com",
    "oauth2": {
      "provider": "AAABlf_0iLgAAAAQ",
      "refreshToken": "1//0gExampleRefreshTokenFromGoogle",
      "auth": { "user": "user@gmail.com" }
    }
  }'
```

`provider` is the ID of an OAuth2 application you registered in EmailEngine, not the name of the provider, and `refreshToken` is what that application's authorization flow handed back. The [account setup guide](/docs/accounts/managing-accounts) covers where both come from, and [hosted authentication](/docs/accounts/hosted-authentication) covers letting EmailEngine collect them for you.

### 3. Send Your First Email

```bash
curl -X POST http://localhost:3000/v1/account/my-account/submit \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "to": [{"address": "recipient@example.com"}],
    "subject": "Hello from EmailEngine",
    "text": "This is my first email via the API!"
  }'
```

[Sending guide →](/docs/sending/transactional-service)

### 4. Set Up Webhooks

Configure webhooks to receive real-time notifications. `webhookEvents` is an allowlist with no default, so name the events you want, or `["*"]` for all of them:

```bash
curl -X POST http://localhost:3000/v1/settings \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "webhooks": "https://yourapp.com/webhooks/email",
    "webhooksEnabled": true,
    "webhookEvents": ["messageNew", "messageSent", "messageDeliveryError"]
  }'
```

[Webhooks guide →](/docs/webhooks/overview)

## Email API Documentation

- [API Reference](/docs/api-reference) - Complete endpoint documentation
- [Sending API](/docs/api-reference/sending-api) - Email submission endpoints
- [Messages API](/docs/api-reference/messages-api) - Read and manage emails
- [Accounts API](/docs/api-reference/accounts-api) - Account management
- [Webhooks Reference](/docs/api-reference/webhooks-api) - Event notifications

## Use Cases

### CRM Email Integration
Integrate customer email communications directly into your CRM. Track conversations, send follow-ups, and manage relationships.
[CRM integration guide →](/docs/integrations/crm)

### Transactional Email
Send receipts, notifications, and automated emails from user accounts rather than a shared sending domain.
[Transactional email guide →](/docs/sending/transactional-service)

### Customer Support
Build email into your help desk. Manage support inboxes, track threads, and send templated responses.
[Support integration examples →](/docs/integrations/crm)

### AI Email Processing
Connect email to AI systems for summarization, classification, and automated responses.
[AI integration guide →](/docs/integrations/ai-chatgpt)

## See Also

- [Introduction](/docs/getting-started/introduction) - What EmailEngine is and how it fits an application
- [Quick Start](/docs/getting-started/quick-start) - The same four steps with the responses shown
- [API Reference](/docs/api-reference) - Authentication, conventions, and error handling
- [IMAP API](/docs/imap-api) - The same API described from the IMAP side
- [Licensing](/docs/licensing) - Trial terms and what a production license covers
