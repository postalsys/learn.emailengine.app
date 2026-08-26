---
title: PHP Integration
sidebar_position: 7
description: Using the official EmailEngine PHP SDK from Composer to register accounts, send and read mail, and verify webhooks
---

# PHP Integration Guide

How to call EmailEngine from PHP with the official SDK, [postalsys/emailengine-php](https://packagist.org/packages/postalsys/emailengine-php). The SDK is a thin client over the REST API: every call maps to one endpoint documented in the [API reference](/docs/api-reference), and the request and response bodies are the ones the [OpenAPI specification](/docs/api-reference/openapi-spec) describes.

This page matches SDK version 1.3.0 (checked 2026-08-26). It requires PHP 8.1 or newer and uses Guzzle 7 for HTTP.

## Installation

```bash
composer require postalsys/emailengine-php
```

## Quick Start

### 1. Create a Client

```php
<?php

require 'vendor/autoload.php';

use Postalsys\EmailEnginePhp\EmailEngine;

$ee = new EmailEngine(
    accessToken: '3eb50ef80efb67885af...',
    baseUrl: 'https://emailengine.example.com',
);
```

The constructor takes named (or positional) arguments. `EmailEngine::fromOptions()` accepts the same values as an array with snake_case keys:

```php
<?php

use Postalsys\EmailEnginePhp\EmailEngine;

$ee = EmailEngine::fromOptions([
    'access_token' => '3eb50ef80efb67885af...',
    'ee_base_url' => 'https://emailengine.example.com',
    'service_secret' => 'the-service-secret-from-configuration-security',
    'redirect_url' => 'https://app.example.com/email/connected',
]);
```

### Client Options

| Constructor argument | `fromOptions()` key | Required | Description |
|----------------------|---------------------|----------|-------------|
| `accessToken` | `access_token` | Yes | An EmailEngine access token |
| `baseUrl` | `ee_base_url` | No | Base URL of the instance. Defaults to `http://localhost:3000` |
| `serviceSecret` | `service_secret` | No | The instance's **Service Secret** (Configuration > Security). Needed for `verifyWebhookSignature()` |
| `redirectUrl` | `redirect_url` | No | Default `redirectUrl` for `getAuthenticationUrl()` |
| `timeout` | `timeout` | No | HTTP timeout in seconds. Defaults to 30 |

**Getting an Access Token**:
1. Open the EmailEngine admin interface and sign in
2. Navigate to **Integrations** > **Access Tokens**
3. Click **Create access token**, pick the scope and, if the token is for one mailbox only, the account
4. Copy the token; it is shown once

### Two Ways to Call the API

The client exposes one resource object per API area: `$ee->accounts`, `$ee->messages`, `$ee->mailboxes`, `$ee->outbox`, `$ee->settings`, `$ee->tokens`, `$ee->templates`, `$ee->gateways`, `$ee->oauth2`, `$ee->webhooks`, `$ee->stats`, and `$ee->blocklists`. Each method calls one endpoint and returns the decoded JSON response as an array.

For anything the resource classes do not cover, `request()` calls any path directly:

```php
<?php

// request(string $method, string $path, ?array $data = null, array $query = [], array $headers = [])
$stats = $ee->request('GET', '/v1/stats');
$results = $ee->request('POST', '/v1/account/example/search', ['search' => ['unseen' => true]], ['path' => 'INBOX']);
```

`$data` is sent as the JSON body, `$query` as the query string. The examples below use the resource classes where one exists.

## Registering an Email Account

`$ee->accounts->create()` posts to [`POST /v1/account`](/docs/api/post-v-1-account) with the same body the endpoint takes:

```php
<?php

$account = $ee->accounts->create([
    'account' => 'example',
    'name' => 'John Doe',
    'email' => 'john@example.com',
    'imap' => [
        'auth' => [
            'user' => 'john@example.com',
            'pass' => 'your-password',
        ],
        'host' => 'imap.example.com',
        'port' => 993,
        'secure' => true,
    ],
    'smtp' => [
        'auth' => [
            'user' => 'john@example.com',
            'pass' => 'your-password',
        ],
        'host' => 'smtp.example.com',
        'port' => 465,
        'secure' => true,
    ],
]);

echo "Account {$account['account']} is {$account['state']}\n";
```

The response carries `account` and `state`, where `state` is `new` when the ID did not exist and `existing` when it did and was updated. Registering an ID that already exists is therefore not an error.

### Account Configuration

**IMAP Settings**:
- `auth.user`: IMAP username (usually the email address)
- `auth.pass`: IMAP password or app-specific password
- `host`: IMAP server hostname
- `port`: IMAP port (993 for TLS, 143 for STARTTLS)
- `secure`: `true` for TLS on connect, `false` for plaintext with STARTTLS

**SMTP Settings**:
- `auth.user`: SMTP username (usually the email address)
- `auth.pass`: SMTP password or app-specific password
- `host`: SMTP server hostname
- `port`: SMTP port (465 for TLS, 587 for STARTTLS)
- `secure`: `true` for TLS on connect, `false` for plaintext with STARTTLS

### Hosted Authentication Instead of Credentials

If you would rather not collect passwords in your own form, `getAuthenticationUrl()` calls [`POST /v1/authentication/form`](/docs/api/post-v-1-authentication-form) and returns the URL of EmailEngine's hosted form. Redirect the user there; EmailEngine sends them back to `redirectUrl` once the account is connected.

```php
<?php

$url = $ee->getAuthenticationUrl([
    'account' => 'example',
    'name' => 'John Doe',
    'email' => 'john@example.com',
    'redirectUrl' => 'https://app.example.com/email/connected',
]);

header('Location: ' . $url);
```

`redirectUrl` is required, either here or as the client's `redirectUrl` option. See [Hosted Authentication](/docs/accounts/hosted-authentication) for the form's options.

## Waiting for Account Connection

After registering an account, EmailEngine connects and indexes it in the background. Poll [`GET /v1/account/{account}`](/docs/api/get-v-1-account-account) until it reports a usable state:

```php
<?php

$deadline = time() + 120;

while (time() < $deadline) {
    $info = $ee->accounts->get('example');

    if (in_array($info['state'], ['connected', 'syncing'], true)) {
        echo "Account is {$info['state']}\n";
        break;
    }

    if ($info['state'] === 'authenticationError') {
        throw new RuntimeException('Credentials were rejected: ' . ($info['lastError']['response'] ?? ''));
    }

    echo "Account state is {$info['state']}\n";
    sleep(1);
}
```

### Account States

Poll until the state reaches `connected` or `syncing`, and stop on `authenticationError`, which does not resolve on its own. `unset` means the account is not syncing at all: either no IMAP or OAuth2 configuration is set, or syncing was switched off, by an operator or automatically after repeated authentication failures. Since v2.79.4 the account object carries `authFailureDisabledAt` (a timestamp, or `null`), which tells the automatic case from a deliberate one. Supplying working credentials lifts it: a new `imap` object through `update()`, or re-authorizing through the hosted form. In v2.79.4 a `partial` update that only changes the password keeps the flag unless it also sets `disabled` to `false`; v2.79.5 lift it on the changed credentials alone. See [Account States](/docs/api-reference/accounts-api#account-states) for the complete list.

The loop above has a deadline. Without one, an account that can never connect keeps the loop running forever.

## Sending Emails

Once the account is connected, `$ee->messages->submit()` posts to [`POST /v1/account/{account}/submit`](/docs/api/post-v-1-account-account-submit):

```php
<?php

$result = $ee->messages->submit('example', [
    'from' => [
        'name' => 'John Doe',
        'address' => 'john@example.com',
    ],
    'to' => [
        [
            'name' => 'Jane Smith',
            'address' => 'jane@example.com',
        ],
    ],
    'subject' => 'Test message',
    'text' => 'Hello from PHP!',
    'html' => '<p>Hello from <strong>PHP</strong>!</p>',
]);

echo "Queued as {$result['queueId']} with Message-ID {$result['messageId']}\n";
```

The message is queued, not sent inline: the response carries `messageId` (the Message-ID header EmailEngine assigned), `queueId` (the queue entry), and `sendAt`. Delivery is reported later through the `messageSent` or `messageDeliveryError` webhooks. A third argument takes request options: `idempotencyKey` is sent as the `Idempotency-Key` header, so a retried call does not queue the message twice, and `timeout` (seconds) as `X-EE-Timeout`.

### Message Options

**Main Fields**:
- `from`: Sender address (object with `name` and `address`). Optional; defaults to the account's configured identity
- `to`: Array of recipient addresses

**Other Fields**:
- `cc`: Carbon copy recipients (array)
- `bcc`: Blind carbon copy recipients (array)
- `subject`: Email subject line
- `text`: Plain text content
- `html`: HTML content. A submission with `html` only gets no generated plain-text part
- `attachments`: Array of attachment objects
- `headers`: Custom email headers
- `sendAt`: Future delivery time, as an ISO 8601 timestamp

See [Sending](/docs/sending) for the full field list, templates, and mail merge.

### Sending with Attachments

```php
<?php

$result = $ee->messages->submit('example', [
    'from' => [
        'name' => 'John Doe',
        'address' => 'john@example.com',
    ],
    'to' => [
        [
            'address' => 'jane@example.com',
        ],
    ],
    'subject' => 'Invoice Attached',
    'text' => 'Please find the invoice attached.',
    'attachments' => [
        [
            'filename' => 'invoice.pdf',
            'contentType' => 'application/pdf',
            'content' => base64_encode(file_get_contents('/path/to/invoice.pdf')),
            'encoding' => 'base64',
        ],
    ],
]);
```

`base64` is the only accepted `encoding`, and the attachment counts against the instance's `EENGINE_MAX_SIZE` limit (5 MB by default).

## Common Operations

### Listing Messages

`list()` calls [`GET /v1/account/{account}/messages`](/docs/api/get-v-1-account-account-messages). `path` is a required query parameter, and the second argument is sent as the query string:

```php
<?php

$page = $ee->messages->list('example', ['path' => 'INBOX', 'pageSize' => 20]);

foreach ($page['messages'] as $message) {
    echo "From: {$message['from']['address']}\n";
    echo "Subject: {$message['subject']}\n";
    echo "---\n";
}
```

### Getting Message Details

`get()` calls [`GET /v1/account/{account}/message/{message}`](/docs/api/get-v-1-account-account-message-message). Text content is not included unless `textType` asks for it:

```php
<?php

// textType=* returns both the plain-text and the HTML part
$message = $ee->messages->get('example', $messageId, ['textType' => '*']);

echo "Subject: {$message['subject']}\n";
echo "Text: " . ($message['text']['plain'] ?? '') . "\n";

// Attachments are listed as metadata; fetch the content by attachment ID
foreach ($message['attachments'] as $attachment) {
    echo "Attachment: {$attachment['filename']} ({$attachment['id']})\n";
}
```

### Searching Messages

[`POST /v1/account/{account}/search`](/docs/api/post-v-1-account-account-search) takes the criteria in the body and the folder as a query parameter. The `search()` resource method sends only a body, so use `request()` when you need `path`:

```php
<?php

$results = $ee->request('POST', '/v1/account/example/search', [
    'search' => [
        'from' => 'jane@example.com',
    ],
], ['path' => 'INBOX']);

echo "Found {$results['total']} messages\n";
```

### Updating Account Settings

`update()` calls [`PUT /v1/account/{account}`](/docs/api/put-v-1-account-account). The `imap` and `smtp` objects replace the stored configuration unless `partial` is set, so include it when changing one field:

```php
<?php

$ee->accounts->update('example', [
    'name' => 'John Doe Updated',
    'smtp' => [
        'partial' => true,
        'host' => 'new-smtp.example.com',
    ],
]);
```

### Deleting an Account

```php
<?php

$ee->accounts->delete('example');
echo "Account deleted\n";
```

## Error Handling

Every method throws on a non-2xx response. The SDK maps the status code to a typed exception, all of which extend `EmailEngineException`:

| Exception | HTTP status |
|-----------|-------------|
| `ValidationException` | 400 |
| `AuthenticationException` | 401 |
| `AuthorizationException` | 403 |
| `NotFoundException` | 404 |
| `RateLimitException` | 429; `getRetryAfter()` returns the `Retry-After` value |
| `ServerException` | 500 and above |
| `EmailEngineException` | Anything else, including 413, 422 and transport failures |

`getMessage()` carries the `error` field of the API's error body, which is the HTTP status phrase (`Bad Request`, `Not Found`), not the human-readable `message` field; SDK 1.3.0 does not expose that one. `getErrorCode()` returns the body's `code` and `getDetails()` its `details`, which for a validation error names the offending fields. See the [error reference](/docs/reference/error-codes) for the error body itself.

```php
<?php

use Postalsys\EmailEnginePhp\Exceptions\AuthenticationException;
use Postalsys\EmailEnginePhp\Exceptions\NotFoundException;
use Postalsys\EmailEnginePhp\Exceptions\ValidationException;
use Postalsys\EmailEnginePhp\Exceptions\EmailEngineException;

try {
    $info = $ee->accounts->get('example');
} catch (AuthenticationException $e) {
    // The access token was rejected
    error_log('Check the EmailEngine access token: ' . $e->getMessage());
} catch (NotFoundException $e) {
    // No account with this ID
    echo "Account not registered\n";
} catch (ValidationException $e) {
    // The request body did not pass validation; getDetails() names the fields
    error_log('Invalid request: ' . json_encode($e->getDetails()));
} catch (EmailEngineException $e) {
    error_log("EmailEngine error {$e->getErrorCode()}: {$e->getMessage()}");
}
```

Registering an account ID that already exists is not an error: the request succeeds with `"state": "existing"`. A `ValidationException` with the code `AccountAlreadyExists` is raised only when another account for the same OAuth2 user already exists.

## Processing Webhooks

A webhook handler is a plain HTTP endpoint. Verify the signature first, then act on the event:

```php
<?php

// webhook.php

require 'vendor/autoload.php';

use Postalsys\EmailEnginePhp\EmailEngine;

$ee = new EmailEngine(
    accessToken: getenv('EMAILENGINE_TOKEN'),
    baseUrl: getenv('EMAILENGINE_URL'),
    serviceSecret: getenv('EMAILENGINE_SERVICE_SECRET'),
);

$body = file_get_contents('php://input');
$signature = $_SERVER['HTTP_X_EE_WH_SIGNATURE'] ?? '';

if (!$ee->verifyWebhookSignature($body, $signature)) {
    http_response_code(403);
    exit('Invalid signature');
}

// Acknowledge first; EmailEngine retries a delivery that does not get a 2xx in time
http_response_code(200);

$payload = json_decode($body, true);

switch ($payload['event']) {
    case 'messageNew':
        handleNewMessage($payload);
        break;
    case 'messageBounce':
        handleBounce($payload);
        break;
    default:
        error_log("Unhandled event: {$payload['event']}");
}

function handleNewMessage(array $payload): void
{
    $account = $payload['account'];
    $from = $payload['data']['from']['address'] ?? '';
    $subject = $payload['data']['subject'] ?? '';

    error_log("New email for $account from $from: $subject");
}

function handleBounce(array $payload): void
{
    $recipient = $payload['data']['recipient'];
    $reason = $payload['data']['response']['message'] ?? '';

    error_log("Email to $recipient bounced: $reason");
}
```

`verifyWebhookSignature()` recomputes the HMAC-SHA256 of the raw body with the instance's Service Secret and compares it with the `X-EE-Wh-Signature` header, so the client needs the `serviceSecret` option and the body must be read before anything decodes it. Signature verification and retry behavior are described in the [Webhooks overview](/docs/webhooks/overview).

### Webhook Configuration

The target URL and the event allowlist are instance settings. `setWebhooks()` writes them through [`POST /v1/settings`](/docs/api/post-v-1-settings), using its own short keys: `url` becomes the `webhooks` setting, `events` becomes `webhookEvents`, `enabled` becomes `webhooksEnabled`, `headers` becomes `notifyHeaders`, and `text` sets `notifyText` (a number also sets `notifyTextSize`, `false` turns text off). Keys it does not know are dropped:

```php
<?php

$ee->settings->setWebhooks([
    'url' => 'https://app.example.com/webhook.php',
    'events' => ['messageNew', 'messageBounce'],
    'enabled' => true,
]);
```

It returns `true` when the API reported the update. To write any setting under its API name, use `$ee->settings->update(['webhookEvents' => ['*']])`, which posts the array as given.

`webhookEvents` is an allowlist with no default: an instance where it was never set delivers nothing, and `['*']` selects every event.

## See Also

- [SDK on GitHub](https://github.com/postalsys/emailengine-php) - Source, issues, and the full method list of every resource class
- [API Reference Overview](/docs/api-reference) - Authentication, error shape, and pagination conventions
- [Webhook Overview](/docs/webhooks/overview) - Signature verification and retry behavior for the handler above
- [CRM Integration Guide](/docs/integrations/crm) - A worked PHP integration built on these same calls
- [Hosted Authentication](/docs/accounts/hosted-authentication) - Onboarding accounts without handling credentials yourself
