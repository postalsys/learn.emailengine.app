---
title: Integrations Overview
sidebar_position: 1
description: Learn how to integrate EmailEngine with your applications and external services
---

# Integrations Overview

EmailEngine provides flexible integration options for connecting email functionality with your applications, CRM systems, automation platforms, and AI services. This section covers common integration patterns and best practices.

## Integration Patterns

### Direct API Integration

Build custom integrations using EmailEngine's REST API:

- **Account Management**: Register and manage email accounts programmatically
- **Message Operations**: Send, receive, search, and manage emails
- **Real-time Events**: Receive webhooks for email events
- **Webhook Processing**: Process incoming email notifications

Learn more in the [API Reference](/docs/api-reference) section.

### Webhook-Driven Architecture

EmailEngine sends webhooks for various events:

- New incoming emails
- Sent email notifications
- Account status changes
- Bounce notifications
- Message updates

This enables event-driven architectures where your application responds to email events in real-time.

### SDK Integration

There is one published SDK, for PHP: [postalsys/emailengine-php](https://packagist.org/packages/postalsys/emailengine-php).

Everywhere else, the API is plain HTTP and JSON, so an HTTP client is enough. For a typed client, generate one from the [OpenAPI specification](/docs/api-reference/openapi-spec) that every EmailEngine instance publishes.

### AI Agents Over MCP

Let an AI client call EmailEngine directly instead of writing an integration for each one. EmailEngine serves the [Model Context Protocol](/docs/mcp) at `/mcp`, so Claude Code, Cursor, claude.ai connectors and agent frameworks can search, read, organize and send mail through a curated tool set, authenticated with a narrowed access token.

**Read more**: [MCP Overview](/docs/mcp)

### Low-Code Platforms

Connect EmailEngine with no-code and low-code automation tools:

- Zapier
- Make.com (Integromat)
- n8n
- Webhook routing with custom transformations

## Common Use Cases

### CRM Integration

Integrate email functionality directly into CRM systems:

- Bidirectional email sync
- Contact activity tracking
- Send emails as CRM users
- Track email conversations

**Read more**: [CRM Integration Guide](/docs/integrations/crm)

### AI and Email Processing

Enhance email workflows with artificial intelligence:

- Automatic email summarization
- Sentiment analysis
- Event and action extraction
- Smart email routing
- Conversational search

**Read more**: [AI and ChatGPT Integration](/docs/integrations/ai-chatgpt)

For the other direction - an AI agent calling EmailEngine rather than EmailEngine calling a model - see [MCP for AI Agents](/docs/mcp).

### Marketing Automation

Build email marketing and automation features:

- Transactional email sending
- Campaign tracking
- Bounce handling
- List management
- Delivery testing

### Support Systems

Integrate with customer support platforms:

- Shared inbox management
- Ticket creation from emails
- Response tracking
- Team collaboration

### Business Applications

Embed email functionality in business apps:

- Document management systems
- Project management tools
- Collaboration platforms
- Custom business workflows

## Architecture Considerations

### Scalability

**Vertical Scaling** (Recommended): Increase CPU, RAM, and optimize worker threads, webhook processing, and Redis configuration.

**Manual Sharding** (Advanced): For very large deployments exceeding single-instance capacity, manually distribute accounts across separate EmailEngine instances, each with its own Redis database.

**Read more**: [Performance Tuning](/docs/advanced/performance-tuning)

### Security

- **API Token Management**: Use separate tokens for different applications
- **Secret Encryption**: Enable encryption for stored credentials
- **Network Security**: Use HTTPS and secure network configurations
- **Access Control**: Implement proper authentication and authorization

**Read more**: [Security Best Practices](/docs/deployment/security)

### Reliability

- **Webhook Retries**: EmailEngine retries a delivery that does not get a 2xx response, so make the receiver idempotent rather than assuming each event arrives once
- **Queue Management**: Acknowledge a webhook immediately and process it asynchronously
- **Error Handling**: Handle API errors and timeouts; the [error reference](/docs/reference/error-codes) lists what to expect
- **Monitoring**: Track system health and performance metrics

**Read more**: [Monitoring](/docs/advanced/monitoring)

### Data Management

- **Message Storage**: Messages are stored on the IMAP server
- **Attachment Handling**: Consider storage requirements for attachments
- **Data Retention**: Implement appropriate retention policies
- **Privacy Compliance**: Ensure GDPR and privacy regulation compliance

## Getting Started

Choose your integration path:

- **PHP Developers**: Start with [PHP Integration Guide](/docs/integrations/php)
- **CRM Builders**: Follow the [CRM Integration Guide](/docs/integrations/crm)
- **AI Enthusiasts**: Explore [AI and ChatGPT Integration](/docs/integrations/ai-chatgpt)
- **No-Code Users**: Check out [Low-Code Integrations](/docs/integrations/low-code)

## See Also

- [API Reference](/docs/api-reference) - Authentication, conventions, and error handling
- [Webhooks overview](/docs/webhooks/overview) - The event side of every integration here
- [MCP for AI agents](/docs/mcp) - Letting an agent call EmailEngine without an integration of its own
- [GitHub issues](https://github.com/postalsys/emailengine/issues) - Bug reports and feature requests
- [Support](/docs/support) - Support channels and what a subscription covers
