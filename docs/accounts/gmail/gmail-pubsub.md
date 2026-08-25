---
title: Gmail Pub/Sub Integration
sidebar_label: Pub/Sub Integration
sidebar_position: 4
description: Configure Google Cloud Pub/Sub for Gmail push notifications in EmailEngine
---

# Gmail Pub/Sub Integration

This guide covers the Google Cloud Pub/Sub integration used by EmailEngine to receive real-time notifications from Gmail accounts using the Gmail REST API.

:::info Prerequisites
This guide assumes you have already configured Gmail API access following the [Gmail API Setup Guide](./gmail-api). Pub/Sub is automatically configured when you create a Gmail API OAuth2 application with a linked service account.
:::

## Overview

When using Gmail REST API (instead of IMAP), EmailEngine relies on Google Cloud Pub/Sub for push notifications:

1. Gmail detects changes in an account (new message, flag change, etc.)
2. Gmail pushes a notification to a Pub/Sub topic
3. EmailEngine polls the Pub/Sub subscription for messages
4. EmailEngine processes the notification and fires webhooks

This is different from IMAP, where EmailEngine maintains persistent connections and uses IMAP IDLE for notifications.

## Configuration Settings

### Service Account Settings

When creating a Gmail Service Account application in EmailEngine, these settings are configured:

| Setting | Description |
|---------|-------------|
| `googleProjectId` | Google Cloud project ID containing the Pub/Sub resources |
| `serviceKey` | Service account private key in PEM format (the `private_key` field of the JSON key file) |
| `baseScopes` | Set to `pubsub` for service accounts managing Pub/Sub |

### OAuth2 Application Settings

When creating a Gmail OAuth2 application, link it to a service account:

| Setting | Description |
|---------|-------------|
| `pubSubApp` | ID of the service account application to use for Pub/Sub management |
| `baseScopes` | Set to `api` for Gmail REST API access |

### Subscription Expiration (TTL)

You can configure how long a Pub/Sub subscription persists without activity before Google automatically deletes it.

| Setting | Description |
|---------|-------------|
| `gmailSubscriptionTtl` | Subscription TTL in days. Empty for Google's default (31 days), `0` for indefinite (never expires), `1`-`365` for a custom TTL |

Configure via the API:

```bash
curl -X POST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "gmailSubscriptionTtl": 0
  }'
```

Or through the admin UI at **Integrations > OAuth2 Apps > Gmail Subscriptions**.

:::warning When do expiration changes take effect?
Changing this setting does **not** immediately update existing subscriptions. The new TTL is applied only when EmailEngine next calls Google's Pub/Sub API for that subscription, which happens when:

- An OAuth2 application is **created** (new subscription gets the current TTL)
- An OAuth2 application is **updated** (existing subscription is patched to match the current TTL)
- A subscription is **re-created after expiring** (e.g., Google deleted it due to inactivity, and EmailEngine's recovery process creates a new one with the current TTL)

If you need the change to apply immediately, edit and save each Pub/Sub-enabled OAuth2 application to trigger a subscription update.
:::

### Generated Pub/Sub Resources

EmailEngine automatically creates and manages these Pub/Sub resources:

| Resource | Naming Pattern | Description |
|----------|----------------|-------------|
| Topic | `projects/{projectId}/topics/ee-pub-{appId}` | Receives notifications from Gmail |
| Subscription | `projects/{projectId}/subscriptions/ee-sub-{appId}` | EmailEngine polls this for messages |

The default names can be overridden with the `googleTopicName` and `googleSubscriptionName` fields when creating or updating the service account application.

## API Configuration

### Creating a Service Account Application

```bash
curl -X POST "https://emailengine.example.com/v1/oauth2" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Gmail Pub/Sub Manager",
    "provider": "gmailService",
    "enabled": true,
    "baseScopes": "pubsub",
    "googleProjectId": "your-project-id",
    "serviceClient": "113457912345678901234",
    "serviceClientEmail": "emailengine@your-project-id.iam.gserviceaccount.com",
    "serviceKey": "-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n"
  }'
```

The values come from the service account JSON key file downloaded from Google Cloud: `serviceClient` is the numeric `client_id`, `serviceClientEmail` is the `client_email`, `googleProjectId` is the `project_id`, and `serviceKey` is the PEM-formatted `private_key` string - not the base64-encoded JSON file.

The response includes the application ID:

```json
{
  "id": "AAABkQ3c5eQ",
  "name": "Gmail Pub/Sub Manager",
  "provider": "gmailService",
  "baseScopes": "pubsub"
}
```

### Creating a Gmail OAuth2 Application with Pub/Sub

```bash
curl -X POST "https://emailengine.example.com/v1/oauth2" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Gmail API OAuth2",
    "provider": "gmail",
    "enabled": true,
    "baseScopes": "api",
    "clientId": "YOUR_CLIENT_ID",
    "clientSecret": "YOUR_CLIENT_SECRET",
    "redirectUrl": "https://emailengine.example.com/oauth",
    "pubSubApp": "AAABkQ3c5eQ"
  }'
```

The `pubSubApp` field links this OAuth2 application to the service account for Pub/Sub management.

### Checking Pub/Sub Status

Retrieve application details to check Pub/Sub status:

```bash
curl "https://emailengine.example.com/v1/oauth2/AAABkQ3c5eQ" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

Response includes Pub/Sub resource names:

```json
{
  "id": "AAABkQ3c5eQ",
  "name": "Gmail Pub/Sub Manager",
  "provider": "gmailService",
  "pubSubTopic": "projects/my-project/topics/ee-pub-AAABkQ3c5eQ",
  "pubSubSubscription": "projects/my-project/subscriptions/ee-sub-AAABkQ3c5eQ",
  "pubSubIamPolicy": true
}
```

### Listing All Pub/Sub Applications

To list all Pub/Sub-enabled OAuth2 applications and their subscription errors:

```bash
curl "https://emailengine.example.com/v1/pubsub/status" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

```json
{
  "total": 2,
  "page": 0,
  "pages": 1,
  "apps": [
    {
      "id": "AAABkQ3c5eQ",
      "name": "Gmail Pub/Sub Manager",
      "pubSubError": null
    },
    {
      "id": "AAABkR7d2fR",
      "name": "Another App",
      "pubSubError": {
        "message": "Failed to process subscription loop",
        "description": "Subscription not found"
      }
    }
  ]
}
```

A `pubSubError` of `null` means the subscription is healthy. If present, it contains the latest error from the Pub/Sub subscription loop.

:::info Subscription expiration is not shown in the status list
The `/v1/pubsub/status` endpoint does not include subscription expiration dates. Expiration is managed by Google Cloud based on the TTL configured in EmailEngine. To check the actual expiration policy on a subscription, use the Google Cloud Console or the `gcloud` CLI:

```bash
gcloud pubsub subscriptions describe ee-sub-AAABkQ3c5eQ \
  --project=your-project-id \
  --format="value(expirationPolicy.ttl)"
```
:::

## Required Google Cloud Permissions

The service account requires these permissions:

### Pub/Sub Admin Role

The **Pub/Sub Admin** role (`roles/pubsub.admin`) provides all necessary permissions:

- `pubsub.topics.create` - Create Pub/Sub topics
- `pubsub.topics.delete` - Delete Pub/Sub topics
- `pubsub.topics.setIamPolicy` - Set IAM policies on topics
- `pubsub.subscriptions.create` - Create subscriptions
- `pubsub.subscriptions.delete` - Delete subscriptions
- `pubsub.subscriptions.consume` - Pull messages from subscriptions

### Alternative: Custom Role

For least-privilege access, create a custom role with only:

```yaml
title: "EmailEngine Pub/Sub Manager"
description: "Manages Pub/Sub for EmailEngine Gmail integration"
includedPermissions:
  - pubsub.topics.create
  - pubsub.topics.delete
  - pubsub.topics.get
  - pubsub.topics.setIamPolicy
  - pubsub.topics.getIamPolicy
  - pubsub.topics.attachSubscription
  - pubsub.subscriptions.create
  - pubsub.subscriptions.update
  - pubsub.subscriptions.delete
  - pubsub.subscriptions.get
  - pubsub.subscriptions.consume
```

## Gmail Watch Renewal

EmailEngine automatically renews Gmail watch subscriptions to maintain push notifications:

- **Watch duration**: Gmail watches expire after approximately 7 days
- **Renewal frequency**: EmailEngine renews watches before expiration
- **Account storage**: Last watch time is stored per account (`lastWatch` field)

:::info Topic region policies
Renewal only succeeds if Google's Gmail push service can publish to the topic. A message storage policy that restricts the topic to specific regions with in-transit enforcement blocks renewals - see [Watch renewal fails with a message storage policy error](#watch-renewal-fails-with-a-message-storage-policy-error).
:::

### Force Watch Renewal

To force a watch renewal for an account:

```bash
curl -X POST "https://emailengine.example.com/v1/account/ACCOUNT_ID/reconnect" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

This triggers a full account reconnection, including watch renewal.

## Troubleshooting

### Pub/Sub Topic Creation Failed

**Error:** `Failed to create Pub/Sub topic`

**Causes:**
- Service account lacks `pubsub.topics.create` permission
- Pub/Sub API not enabled in the Google Cloud project
- Project quota exceeded

**Solution:**
1. Verify the service account has **Pub/Sub Admin** role
2. Enable Cloud Pub/Sub API in Google Cloud Console
3. Check project quotas in Google Cloud Console

### IAM Policy Update Failed

**Error:** `Failed to set IAM policy on topic`

**Causes:**
- Service account lacks `pubsub.topics.setIamPolicy` permission
- Gmail service account email not granted publisher permission

**Solution:**
1. Ensure service account has **Pub/Sub Admin** role
2. EmailEngine automatically grants `gmail-api-push@system.gserviceaccount.com` the publisher role

### Subscription Messages Not Received

**Symptoms:**
- Webhooks not firing for Gmail accounts
- Account shows as connected but no events

**Causes:**
- Gmail watch expired and not renewed
- Subscription not receiving messages
- EmailEngine webhook worker not processing

**Solution:**
1. Check EmailEngine logs for Pub/Sub errors:
   ```bash
   journalctl -u emailengine | grep "pub/sub"
   ```
2. Force account reconnection to renew watch
3. Verify webhook worker is running (`threads{type="webhooks"}` metric)

### Watch Renewal Fails with a Message Storage Policy Error

**Symptoms:**

- An account's **Change subscription** status shows **Failed** in the admin UI, with the error **OAuth2 request failed**
- The **Show details** view contains a `FAILED_PRECONDITION` region error, for example: _"The topic's message storage policy requires enforcement in transit, but the Publish request was received by a Pub/Sub server in a non-allowed region."_

**Cause:**

The topic has a message storage policy with in-transit enforcement (`enforce_in_transit`) that limits it to specific regions - often applied through an organization-level "Enforce in-transit regions for Pub/Sub messages" policy. Notifications are published to the topic by Google's Gmail push service (`gmail-api-push@system.gserviceaccount.com`), not by EmailEngine. That publisher routes through Google's global infrastructure and cannot be pinned to a region, so the publish is rejected. This is a general limitation of Gmail push - no client controls which region Gmail publishes from - and it cannot be avoided by pointing EmailEngine at a regional Pub/Sub endpoint.

**Solution:**

Choose one of the following:

1. **Keep real-time push - relax the topic policy.** Remove in-transit enforcement on the EmailEngine topic, or ask whoever manages your Google Cloud organization policies to exclude this topic or project so the Gmail publisher is allowed. In-transit enforcement also rejects EmailEngine's own subscription pulls on the global endpoint, so lifting it is required for Pub/Sub notifications to work end to end.
2. **Drop Pub/Sub - use polling instead.** Create the Gmail OAuth2 application without a linked service account (leave `pubSubApp` unset). EmailEngine then skips the watch entirely and falls back to polling each account about every 10 minutes for changes. Real-time push is lost, but the accounts sync normally and the policy is never triggered.

### Debugging Pub/Sub in Logs

EmailEngine logs Pub/Sub activity at various levels:

```bash
# View Pub/Sub processing logs
journalctl -u emailengine | grep -i pubsub

# View Google API errors
journalctl -u emailengine | grep "googleapis"

# View watch renewal logs
journalctl -u emailengine | grep "watch"
```

## Monitoring

### Prometheus Metrics

Monitor Pub/Sub-related activity through these metrics:

| Metric | Description |
|--------|-------------|
| `oauth2_api_request{provider="gmail"}` | Gmail API requests including watch calls |
| `oauth2_token_refresh{provider="gmailService"}` | Service account token refreshes |
| `events{event="messageNew"}` | New message events (triggered by Pub/Sub notifications) |

### Health Indicators

A healthy Pub/Sub integration shows:

- Regular `messageNew` events for active Gmail accounts
- Successful `oauth2_api_request` metrics
- No Pub/Sub errors in logs

## Architecture

```
                     +-----------------+
                     |   Gmail API     |
                     +--------+--------+
                              |
                              | Push notification
                              v
+----------------+   +--------+--------+   +------------------+
|  EmailEngine   |<--|  Cloud Pub/Sub  |<--|  Gmail Accounts  |
|  Webhook       |   |                 |   |  (via watches)   |
|  Worker        |   +-----------------+   +------------------+
+-------+--------+
        |
        | Poll subscription
        v
+-------+--------+
|  Your App      |
|  (webhooks)    |
+----------------+
```

**Flow:**

1. EmailEngine creates a Pub/Sub topic and subscription
2. EmailEngine registers a Gmail watch pointing to the topic
3. Gmail pushes notifications when changes occur
4. EmailEngine webhook worker polls the subscription
5. Notifications are converted to webhooks and delivered to your app

## Comparison: Pub/Sub vs IMAP IDLE

| Feature | Gmail API + Pub/Sub | IMAP + IDLE |
|---------|---------------------|-------------|
| Connection type | HTTP polling | Persistent TCP |
| Notification latency | 1-10 seconds | Near real-time |
| Resource usage | Lower (no persistent connections) | Higher (per-account connections) |
| Reliability | Very high (managed by Google) | Connection-dependent |
| Setup complexity | Higher (service account + Pub/Sub) | Lower (OAuth2 only) |
| Scope requirements | Granular (`gmail.modify`, etc.) | Full scope (`mail.google.com`) |

## See Also

- [Gmail API Setup](./gmail-api) - Complete setup guide for Gmail API access
- [Gmail IMAP OAuth2](./gmail-imap) - Alternative setup using IMAP/SMTP
- [Monitoring](/docs/advanced/monitoring) - Prometheus metrics for monitoring
- [Webhooks Overview](/docs/webhooks/overview) - Understanding EmailEngine webhooks
