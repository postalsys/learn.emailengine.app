---
title: What is EmailEngine? | Email API Introduction
description: EmailEngine is a self-hosted email API gateway. Learn how it provides unified REST API access to IMAP, SMTP, Gmail, and Microsoft 365 email accounts.
sidebar_position: 0
keywords:
  - what is EmailEngine
  - email API gateway
  - unified email API
  - IMAP REST API
  - email integration platform
---

import Price from '@site/src/components/Price';

# What is EmailEngine?

**EmailEngine** is a self-hosted email gateway that allows you to access email accounts over REST API. It provides a unified interface to interact with IMAP and SMTP protocols, as well as native integrations with Gmail API and Microsoft Graph API.

One REST API covers every account type it supports:

- **IMAP** - Standard email protocol
- **SMTP** - Standard email sending protocol
- **Gmail API** - Native Gmail integration
- **Microsoft Graph API** - Native Microsoft 365/Outlook integration

## What EmailEngine is NOT

Understanding what EmailEngine does not do is as important as knowing what it does:

### Not a Mail Server

EmailEngine is **not a mail server** like Postfix, Sendmail, or Microsoft Exchange. You cannot create new email accounts with EmailEngine. It does not host mailboxes, manage email domains, or provide MX records for receiving mail directly.

Instead, EmailEngine connects to **existing email accounts** that you already have with providers like Gmail, Outlook, or any IMAP-compatible email service. Think of it as a bridge between your application and email providers, not as a replacement for those providers.

### Not a Managed Service

EmailEngine is **self-hosted software**, not a cloud service. There is no EmailEngine-hosted API endpoint you can call. You must deploy and run EmailEngine on your own infrastructure, whether that's:

- A cloud server (AWS, DigitalOcean, Render, etc.)
- An on-premise server
- A Docker container
- Your local development machine

This gives you complete control over your data and privacy, but also means you're responsible for hosting, maintenance, and scaling.

### Summary

| EmailEngine IS | EmailEngine is NOT |
|----------------|-------------------|
| An email API gateway | A mail server |
| A bridge to existing accounts | A service that creates email accounts |
| Self-hosted software | A managed cloud service |
| A unified REST API | A replacement for your email provider |

## Key Features

### Unified REST API

Access all email accounts through a single, consistent REST API regardless of the underlying protocol (IMAP, Gmail API, or Microsoft Graph).

### Real-time Webhooks

Receive instant notifications about new emails, email updates, and account changes through webhooks.

### OAuth2 Support

Built-in support for OAuth2 authentication for Gmail, Google Workspace, Microsoft 365, and Outlook.com accounts.

### Email Sending

Send emails through SMTP or native APIs with support for attachments, HTML content, and templates.

### Account Management

Register and manage multiple email accounts with automatic connection handling and reconnection.

### Message Management

- List, search, and filter messages
- Mark messages as read/unread
- Move messages between folders
- Delete messages
- Download attachments

### AI Agent Access (MCP)

Serve the [Model Context Protocol](/docs/mcp) so AI assistants can search, read, organize and send mail through a curated tool set, using narrowed access tokens rather than mailbox credentials.

### Self-hosted

Run EmailEngine on your own infrastructure for complete control over your email data and privacy.

## Use Cases

- **Email Integration** - Add email functionality to your SaaS application
- **Email Automation** - Automate email workflows and responses
- **Customer Support** - Integrate customer email communications into your CRM
- **Email Analytics** - Track and analyze email communications
- **Email Backup** - Create backups of email accounts

## Quick Start

Getting started takes five steps:

1. **[Install EmailEngine](/docs/installation)** - Download and set up EmailEngine on your server
2. **[Configure Redis](/docs/configuration/redis)** - Set up Redis for data storage
3. **[Register an Account](/docs/api/post-v-1-account)** - Add your first email account via API
4. **[Set up Webhooks](/docs/webhooks/overview)** - Configure webhooks to receive notifications
5. **[Start Building](/docs/api-reference)** - Explore the API reference and build your integration

## System Requirements

- **Node.js** 20 or newer, and only when running from source. The packaged builds carry their own runtime
- **Redis** 6.0 or newer, or a Redis-compatible service such as Upstash
- **Memory** 2 GB to evaluate, 4 to 8 GB for production
- **OS** Linux, macOS, or Windows

See [Installation](/docs/installation) for the per-platform figures and what drives them.

## License

EmailEngine includes a **14-day free trial** with full functionality and no limitations. No credit card is required: the trial is activated from the **License** page in the admin interface, as described under [Free Trial](/docs/licensing#free-trial).

For production use, [get a license key](https://postalsys.com/plans) from postalsys.com<Price />.

## See Also

- [Quick Start](/docs/getting-started/quick-start) - Install, connect an account, send a message
- [Installation](/docs/installation) - Every supported way to run EmailEngine
- [API Reference](/docs/api-reference) - Authentication, conventions, and error handling
- [Comparison with Nylas](/docs/comparison/emailengine-vs-nylas) - How a self-hosted gateway differs from a managed service
- [Licensing and privacy](/docs/licensing) - Trial terms, license options, and what leaves your server
