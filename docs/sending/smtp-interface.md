---
title: SMTP Server
sidebar_position: 8
description: Using EmailEngine's SMTP interface for sending emails with standard SMTP clients
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# SMTP Server

EmailEngine provides an SMTP interface that allows you to send emails using standard SMTP protocol instead of the REST API. EmailEngine acts as an SMTP proxy - it accepts messages via SMTP and routes them through the appropriate email account. This is useful for legacy applications or when you need to integrate with tools that only support SMTP.

## Why Use the SMTP Server

The SMTP interface is beneficial when:

- **Legacy applications**: Integration with older systems that only support SMTP
- **Email clients**: Using desktop or mobile email clients
- **Third-party tools**: Tools that require SMTP configuration
- **Standard libraries**: Using standard SMTP libraries in your code
- **Drop-in replacement**: Replacing an existing SMTP server without code changes

## How It Works

When the SMTP interface is enabled:

1. EmailEngine listens on an SMTP port (default: 2525)
2. Clients connect using SMTP protocol
3. Authentication determines which account to use
4. EmailEngine routes the message to the appropriate account's SMTP server
5. Messages are queued just like with the Submit API
6. Delivery status tracked via webhooks

## Enabling the SMTP Server

### Via Settings API (Recommended)

Configure the SMTP interface using the Settings API:

```bash
curl -XPOST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "smtpServerEnabled": true,
    "smtpServerPort": 2525,
    "smtpServerHost": "0.0.0.0",
    "smtpServerAuthEnabled": true,
    "smtpServerPassword": "optional-global-password"
  }'
```

### Via Web Interface

Navigate to **Configuration > SMTP Server** in the EmailEngine admin panel to configure the settings.

![SMTP Server configuration page](/img/screenshots/smtp-interface-config.png)
_Enable the SMTP server and set the listen address, port and authentication options_

### Configuration Options

| Setting | Description | Default |
|---------|-------------|---------|
| `smtpServerEnabled` | Enable/disable SMTP interface | `false` |
| `smtpServerPort` | Port to listen on | `2525` |
| `smtpServerHost` | Host/interface to bind | `0.0.0.0` |
| `smtpServerAuthEnabled` | Require authentication | `false` |
| `smtpServerPassword` | Optional global password | - |
| `smtpServerTLSEnabled` | Enable TLS (implicit) | `false` |

### Restart EmailEngine

After changing configuration, restart EmailEngine for changes to take effect.

## Authentication

### Using Account ID and API Token

Authenticate with the account ID as the username and an [API token](/docs/api-reference/access-tokens) as the password. The message is then sent through that account, using whatever transport it is configured with.

| SMTP setting | Value |
|--------------|-------|
| Host | Your EmailEngine host |
| Port | `2525` by default, set by `smtpServerPort` |
| Encryption | None, unless you [enable TLS](#enabling-tls) |
| Username | The account ID |
| Password | An EmailEngine API token |

:::warning The SMTP username is always the account ID
EmailEngine matches the SMTP `AUTH` username against the **account ID** - the identifier you assigned when registering the account, not the mailbox email address. An email address only works as the username if the account ID happens to be that exact string. Always authenticate with the account ID.
:::

See [Programming Languages](#programming-languages) below for a worked example in Node.js, Python, PHP, or Ruby.

## Configuration Examples

### Desktop Email Clients

<Tabs groupId="mail-client">
<TabItem value="thunderbird" label="Thunderbird" default>

1. Go to **Account Settings → Outgoing Server (SMTP)**
2. Click **Add**
3. Configure:
   - **Server Name**: emailengine.example.com
   - **Port**: 2525
   - **Connection security**: None (or SSL/TLS if TLS is enabled on the server)
   - **Authentication method**: Normal password
   - **Username**: Account ID (the identifier assigned when registering the account)
   - **Password**: API token

</TabItem>
<TabItem value="apple-mail" label="Apple Mail">

1. Go to **Mail → Preferences → Accounts**
2. Select your account
3. Go to **Server Settings**
4. Configure **Outgoing Mail Server**:
   - **Host Name**: emailengine.example.com
   - **Port**: 2525
   - **User Name**: Account ID (the identifier assigned when registering the account)
   - **Password**: API token

</TabItem>
</Tabs>

### Programming Languages

Point any SMTP client at the server. The credentials are the same in every case: the account ID as the user name and an API token as the password.

<Tabs groupId="language">
<TabItem value="nodejs" label="Node.js" default>

Using [Nodemailer](https://nodemailer.com/):

```javascript
const nodemailer = require('nodemailer');

const transporter = nodemailer.createTransport({
  host: 'emailengine.example.com',
  port: 2525,
  secure: false,
  auth: {
    user: 'example-account',
    pass: process.env.EMAILENGINE_TOKEN
  }
});

async function sendEmail() {
  const info = await transporter.sendMail({
    from: '"Sender Name" <sender@example.com>',
    to: 'recipient@example.com',
    subject: 'Test Email',
    text: 'Plain text content',
    html: '<p>HTML content</p>',
    attachments: [
      {
        filename: 'document.pdf',
        path: './files/document.pdf'
      }
    ]
  });

  console.log('Message ID:', info.messageId);
}

sendEmail().catch(console.error);
```

</TabItem>
<TabItem value="python" label="Python">

Using the standard library `smtplib`:

```python
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from email.mime.base import MIMEBase
from email import encoders
import os

def send_email():
    # Create message
    msg = MIMEMultipart()
    msg['From'] = 'sender@example.com'
    msg['To'] = 'recipient@example.com'
    msg['Subject'] = 'Test Email'

    # Add body
    body = 'This is the email body'
    msg.attach(MIMEText(body, 'plain'))

    # Add attachment
    filename = 'document.pdf'
    with open(filename, 'rb') as attachment:
        part = MIMEBase('application', 'octet-stream')
        part.set_payload(attachment.read())

    encoders.encode_base64(part)
    part.add_header(
        'Content-Disposition',
        f'attachment; filename= {filename}'
    )
    msg.attach(part)

    # Send email
    server = smtplib.SMTP('emailengine.example.com', 2525)
    server.login('example-account', os.environ['EMAILENGINE_TOKEN'])
    server.send_message(msg)
    server.quit()

send_email()
```

</TabItem>
<TabItem value="php" label="PHP">

Using [PHPMailer](https://github.com/PHPMailer/PHPMailer):

```php
<?php
use PHPMailer\PHPMailer\PHPMailer;
use PHPMailer\PHPMailer\Exception;

require 'vendor/autoload.php';

$mail = new PHPMailer(true);

try {
    // Server settings
    $mail->isSMTP();
    $mail->Host       = 'emailengine.example.com';
    $mail->SMTPAuth   = true;
    $mail->Username   = 'example-account';
    $mail->Password   = getenv('EMAILENGINE_TOKEN');
    $mail->SMTPSecure = false;  // No encryption by default; use PHPMailer::ENCRYPTION_SMTPS if TLS is enabled
    $mail->Port       = 2525;

    // Recipients
    $mail->setFrom('sender@example.com', 'Sender Name');
    $mail->addAddress('recipient@example.com', 'Recipient Name');

    // Content
    $mail->isHTML(true);
    $mail->Subject = 'Test Email';
    $mail->Body    = '<p>HTML content</p>';
    $mail->AltBody = 'Plain text content';

    // Attachments
    $mail->addAttachment('/path/to/document.pdf');

    $mail->send();
    echo 'Message has been sent';
} catch (Exception $e) {
    echo "Message could not be sent. Error: {$mail->ErrorInfo}";
}
?>
```

</TabItem>
<TabItem value="ruby" label="Ruby">

Using the [Mail](https://github.com/mikel/mail) gem:

```ruby
require 'mail'

Mail.defaults do
  delivery_method :smtp, {
    address: 'emailengine.example.com',
    port: 2525,
    user_name: 'example-account',
    password: ENV['EMAILENGINE_TOKEN'],
    authentication: 'plain',
    enable_starttls_auto: false
  }
end

mail = Mail.new do
  from     'sender@example.com'
  to       'recipient@example.com'
  subject  'Test Email'
  body     'Plain text content'

  add_file '/path/to/document.pdf'
end

mail.deliver!
```

</TabItem>
</Tabs>

## TLS/SSL Support

### Enabling TLS

Enable TLS for encrypted connections using the Settings API or web interface:

```bash
curl -XPOST "https://emailengine.example.com/v1/settings" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "smtpServerTLSEnabled": true
  }'
```

When TLS is enabled, the SMTP server uses implicit TLS (clients must connect with SSL from the start).

### Certificate Configuration

Provide custom TLS certificate using environment variables:

```bash
EENGINE_SMTP_TLS_KEY=/path/to/private.key
EENGINE_SMTP_TLS_CERT=/path/to/certificate.crt
```

Additional TLS options available with the `EENGINE_SMTP_TLS_` prefix:
- `EENGINE_SMTP_TLS_CA` - CA certificate
- `EENGINE_SMTP_TLS_CIPHERS` - TLS ciphers
- `EENGINE_SMTP_TLS_MIN_VERSION` - Minimum TLS version
- `EENGINE_SMTP_TLS_MAX_VERSION` - Maximum TLS version

## Features and Limitations

### Supported Features

- [YES] Standard SMTP protocol
- [YES] Authentication (PLAIN, LOGIN)
- [YES] TLS encryption (implicit TLS when enabled)
- [YES] Multiple recipients (TO, CC, BCC)
- [YES] Attachments
- [YES] Custom headers
- [YES] HTML and plain text
- [YES] Automatic queuing and retries
- [YES] Webhook notifications

### Limitations

- [NO] Cannot specify custom `sendAt` (scheduled sending)
- [NO] Cannot use mail merge via SMTP
- [NO] Cannot reference templates by ID
- [NO] Cannot use reply/forward reference mode
- [NO] Limited access to EmailEngine-specific features

For advanced features, use the [REST API](./basic-sending.md) instead.

## Monitoring and Webhooks

Messages sent via the SMTP interface are treated the same as messages sent via REST API:

- Queued in the outbox queue
- Automatic retry logic
- Webhook notifications (`messageSent`, `messageDeliveryError`, `messageFailed`)
- Visible in Bull Board queue UI

Query queue status (the outbox is a single global queue, not scoped per account):

```bash
curl "https://emailengine.example.com/v1/outbox" \
  -H "Authorization: Bearer <token>"
```

## When to Use the SMTP Server vs REST API

### Use the SMTP Server When:

- Integrating with legacy systems
- Using desktop email clients
- Tools only support SMTP
- Minimal code changes desired
- Standard SMTP features sufficient

### Use REST API When:

- Building new applications
- Need advanced features (mail merge, templates, scheduled sending)
- Need programmatic control
- Want detailed delivery tracking
- Performance is critical (REST is faster)
