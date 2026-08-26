---
title: Gmail Pub/Sub Integration
sidebar_label: Pub/Sub Integration
sidebar_position: 4
description: Configure Google Cloud Pub/Sub for Gmail push notifications in EmailEngine
---

# Gmail Pub/Sub Integration

This guide covers the Google Cloud Pub/Sub integration used by EmailEngine to receive real-time notifications from Gmail accounts using the Gmail REST API.

:::info Prerequisites
This guide assumes you have already configured Gmail API access following the [Gmail API Setup Guide](./gmail-api). Pub/Sub is configured when you create a Gmail API OAuth2 application with a linked service account.
:::

## Overview

When using Gmail REST API (instead of IMAP), EmailEngine relies on Google Cloud Pub/Sub for push notifications:

1. Gmail detects changes in an account (new message, flag change, etc.)
2. Gmail publishes a notification to a Pub/Sub topic
3. EmailEngine pulls the Pub/Sub subscription for messages
4. EmailEngine processes the notification and fires webhooks

This is different from IMAP, where EmailEngine maintains persistent connections and uses IMAP IDLE for notifications.

Pub/Sub is optional. A Gmail API application without a linked service account polls each account for changes about every 10 minutes instead, so webhooks still fire, with up to that much delay.

## Configuration Settings

### Service Account Settings

When creating a Gmail Service Account application in EmailEngine for Pub/Sub, these fields matter:

| Setting | Description |
|---------|-------------|
| `provider` | `gmailService` |
| `baseScopes` | `pubsub` for a service account that manages Pub/Sub |
| `googleProjectId` | Google Cloud project ID containing the Pub/Sub resources (`project_id` in the key file) |
| `serviceClient` | Service account unique ID (`client_id` in the key file) |
| `serviceClientEmail` | Service account email address (`client_email` in the key file) |
| `serviceKey` | Service account private key in PEM format (the `private_key` field of the JSON key file) |
| `googleTopicName`, `googleSubscriptionName` | Optional overrides for the generated resource names, see below |

A service account can also authenticate without a stored key through Workload Identity Federation (`authMethod: "externalAccount"` with `externalAccount`); see [Google Service Accounts](./google-service-accounts#alternative-workload-identity-federation-keyless).

### OAuth2 Application Settings

When creating a Gmail OAuth2 application, link it to a service account:

| Setting | Description |
|---------|-------------|
| `pubSubApp` | ID of the service account application to use for Pub/Sub management |
| `baseScopes` | `api` for Gmail REST API access |

The service account must belong to the same Google Cloud project as the OAuth2 client.

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

Or through the admin UI at **Integrations** > **OAuth2 Apps** > **Gmail Subscriptions**.

:::warning When do expiration changes take effect?
Changing this setting does **not** immediately update existing subscriptions. The new TTL is applied only when EmailEngine next calls Google's Pub/Sub API for that subscription, which happens when:

- A Pub/Sub service account application is **created** (the new subscription gets the current TTL)
- A Pub/Sub service account application is **updated** (the existing subscription is patched to match the current TTL, unless EmailEngine adopted it rather than creating it; an adopted subscription is never modified)
- A subscription is **re-created after expiring** (Google deleted it due to inactivity, and EmailEngine's recovery process creates a new one with the current TTL)

If you need the change to apply immediately, edit and save each Pub/Sub service account application to trigger a subscription update.
:::

### Generated Pub/Sub Resources

EmailEngine creates and manages these Pub/Sub resources for each service account application with the `pubsub` base scope:

| Resource | Naming Pattern | Description |
|----------|----------------|-------------|
| Topic | `projects/{projectId}/topics/ee-pub-{appId}` | Receives notifications from Gmail |
| Subscription | `projects/{projectId}/subscriptions/ee-sub-{appId}` | EmailEngine pulls this for messages |

The default names can be overridden with the `googleTopicName` and `googleSubscriptionName` fields when creating or updating the service account application. If a topic or subscription with the configured name already exists, EmailEngine adopts it and records that it did not create it; such adopted resources are left in place when the application is deleted.

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
    "serviceKey": "-----BEGIN PRIVATE KEY-----\nMIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQC7\n-----END PRIVATE KEY-----\n"
  }'
```

The values come from the service account JSON key file downloaded from Google Cloud: `serviceClient` is the numeric `client_id`, `serviceClientEmail` is the `client_email`, `googleProjectId` is the `project_id`, and `serviceKey` is the PEM-formatted `private_key` string - not the base64-encoded JSON file.

The response carries the application ID:

```json
{
  "id": "AAABkQ3c5eQ",
  "created": true
}
```

Registering the application also creates the topic and subscription and grants Gmail's push service publish access to the topic. A failure in that setup does not fail the request; it is recorded on the application as `lastError` and shown in the Gmail Subscriptions list.

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

The response includes the Pub/Sub resource names alongside the other application fields:

```json
{
  "id": "AAABkQ3c5eQ",
  "name": "Gmail Pub/Sub Manager",
  "provider": "gmailService",
  "enabled": true,
  "baseScopes": "pubsub",
  "googleProjectId": "my-project",
  "pubSubTopic": "projects/my-project/topics/ee-pub-AAABkQ3c5eQ",
  "pubSubSubscription": "projects/my-project/subscriptions/ee-sub-AAABkQ3c5eQ",
  "pubSubIamPolicy": true
}
```

`pubSubIamPolicy` is `true` once Gmail's push service has been granted publish access to the topic. A `lastError` or `pubSubError` object appears in this response only while there is an error to report; the status listing below returns `null` for them instead.

### Listing All Pub/Sub Applications

To list all Pub/Sub service account applications and their errors, use [`GET /v1/pubsub/status`](/docs/api/get-v-1-pubsub-status):

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
      "lastError": null,
      "pubSubError": null
    },
    {
      "id": "AAABkR7d2fR",
      "name": "Another App",
      "lastError": null,
      "pubSubError": {
        "message": "Failed to process subscription loop",
        "description": "Subscription not found"
      }
    }
  ]
}
```

- `lastError` is the last setup error for the application (creating the topic or subscription, setting the IAM policy) or a failed token renewal for the service account; its `response` field carries the message, for example `Enable the Cloud Pub/Sub API`
- `pubSubError` is the latest error from the subscription pull loop, with `message` and `description`

`null` for both means the subscription is healthy.

:::info Subscription expiration is not shown in the status list
The `/v1/pubsub/status` endpoint does not include subscription expiration dates. Expiration is managed by Google Cloud based on the TTL configured in EmailEngine. To check the actual expiration policy on a subscription, use the Google Cloud Console or the `gcloud` CLI:

```bash
gcloud pubsub subscriptions describe ee-sub-AAABkQ3c5eQ \
  --project=your-project-id \
  --format="value(expirationPolicy.ttl)"
```
:::

## Required Google Cloud Permissions

EmailEngine uses the service account to get, create and delete the topic, read and set the topic's IAM policy, get, create, patch and delete the subscription, and pull and acknowledge messages.

### Pub/Sub Admin Role

The **Pub/Sub Admin** role (`roles/pubsub.admin`) covers all of these.

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

A Gmail watch (`users.watch`) tells Gmail to publish an account's changes to the topic, and it expires after about 7 days. EmailEngine renews it:

- On account initialization and on every reconnect, if the last watch is more than a day old
- On a timer that fires hourly, or an hour before the expiration Gmail reported, whichever is later

The time of the last successful watch is stored with the account internally; it is not part of the account API response.

:::info Topic region policies
Renewal only succeeds if Google's Gmail push service can publish to the topic. A message storage policy that restricts the topic to specific regions with in-transit enforcement blocks renewals - see [Watch renewal fails with a message storage policy error](#watch-renewal-fails-with-a-message-storage-policy-error).
:::

### Force Watch Renewal

To force a watch renewal for an account, request a [reconnect](/docs/api/put-v-1-account-account-reconnect):

```bash
curl -X PUT "https://emailengine.example.com/v1/account/user123/reconnect" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"reconnect": true}'
```

A reconnect re-initializes the account and renews the watch regardless of its age. The response is `{"reconnect": true}` when the reconnect was requested. Since v2.79.4 it is `{"reconnect": false}` for an account that EmailEngine [switched off after repeated authentication failures](/docs/accounts/managing-accounts#accounts-switched-off-after-authentication-failures); re-authorize the account instead.

## Troubleshooting

### Pub/Sub Setup Failed When Registering the App

**Symptoms:**
- The application appears in **Integrations** > **OAuth2 Apps** > **Gmail Subscriptions** with a setup error
- `lastError.response` in `GET /v1/pubsub/status` carries a message such as `Enable the Cloud Pub/Sub API`

**Causes:**
- Cloud Pub/Sub API not enabled in the Google Cloud project
- Service account lacks the permissions listed above
- Project quota exceeded

**Solution:**
1. Enable the Cloud Pub/Sub API in Google Cloud Console
2. Verify the service account has the **Pub/Sub Admin** role
3. Edit and save the application in EmailEngine to retry the setup

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
   journalctl -u emailengine | grep -i "pub/sub\|pubsub"
   ```
2. Force account reconnection to renew the watch
3. Verify the webhook worker is running (`threads{type="webhooks"}` metric)

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
journalctl -u emailengine | grep -i "watch"
```

## Monitoring

### Prometheus Metrics

Monitor Pub/Sub-related activity through these metrics:

| Metric | Description |
|--------|-------------|
| `oauth2_api_request{provider="gmail"}` | Gmail API requests, including watch calls; labeled by `status` and `statusCode` |
| `events{event="messageNew"}` | New message events (triggered by Pub/Sub notifications or fallback polling) |
| `threads{type="webhooks"}` | The webhook worker threads, which run the subscription pull loop |

### Health Indicators

A healthy Pub/Sub integration shows:

- Regular `messageNew` events for active Gmail accounts
- Successful `oauth2_api_request` metrics
- `pubSubError: null` for every application in `GET /v1/pubsub/status`

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
        | Pull subscription
        v
+-------+--------+
|  Your App      |
|  (webhooks)    |
+----------------+
```

**Flow:**

1. EmailEngine creates a Pub/Sub topic and subscription when the service account application is registered
2. EmailEngine registers a Gmail watch for each account, pointing to the topic
3. Gmail publishes a notification when changes occur
4. The EmailEngine webhook worker pulls the subscription (up to 100 messages per pull) and acknowledges what it processed
5. Notifications are converted to webhooks and delivered to your app

## Comparison: Pub/Sub vs IMAP IDLE

| Feature | Gmail API + Pub/Sub | IMAP + IDLE |
|---------|---------------------|-------------|
| Connection type | HTTPS pull requests against the subscription | Persistent TCP connection per account |
| Change detection | Gmail history, triggered by notifications and a 10-minute fallback poll | IMAP IDLE plus periodic resync |
| Setup | OAuth2 app, service account, Pub/Sub API | OAuth2 app only |
| Scope requirements | Granular (`gmail.modify` and narrower) | Full scope (`https://mail.google.com/`) |

## See Also

- [Gmail API Setup](./gmail-api) - Complete setup guide for Gmail API access
- [Gmail IMAP OAuth2](./gmail-imap) - Alternative setup using IMAP/SMTP
- [Google Service Accounts](./google-service-accounts) - The service account that manages the Pub/Sub resources
- [Monitoring](/docs/advanced/monitoring) - Prometheus metrics for monitoring
- [Webhooks Overview](/docs/webhooks/overview) - Understanding EmailEngine webhooks
