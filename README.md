# WordPress Email Delivery Diagnostics

A practical guide to diagnosing WordPress email delivery problems, troubleshooting `wp_mail()`, configuring SMTP, investigating PHPMailer errors, and improving email authentication and deliverability.

## Table of Contents

- [Introduction](#introduction)
- [Understanding WordPress Email Delivery](#understanding-wordpress-email-delivery)
- [How wp_mail() Works](#how-wp_mail-works)
- [Testing WordPress Email](#testing-wordpress-email)
- [Diagnosing Email Failures](#diagnosing-email-failures)
- [Logging Email Errors](#logging-email-errors)
- [SMTP Configuration](#smtp-configuration)
- [PHPMailer Diagnostics](#phpmailer-diagnostics)
- [Email Headers and Sender Identity](#email-headers-and-sender-identity)
- [SPF, DKIM, and DMARC](#spf-dkim-and-dmarc)
- [Email Deliverability Problems](#email-deliverability-problems)
- [Common Troubleshooting Scenarios](#common-troubleshooting-scenarios)
- [Security Best Practices](#security-best-practices)
- [Troubleshooting Checklist](#troubleshooting-checklist)
- [References](#references)

## Introduction

WordPress sends different types of automated emails, including:

- User registration notifications.
- Password reset messages.
- Order confirmation emails.
- Contact form notifications.
- Administrator notifications.
- Comment notifications.
- Plugin-generated transactional emails.

Email problems can affect website functionality and customer communication.

Common symptoms include:

- Emails are not being sent.
- Emails are delivered to spam folders.
- Password reset emails never arrive.
- Contact form notifications are missing.
- SMTP authentication fails.
- Emails are sent from an incorrect address.
- Messages work on one hosting provider but fail on another.

This guide provides a systematic approach to diagnosing these problems.

## Understanding WordPress Email Delivery

WordPress uses the `wp_mail()` function as its email-sending interface.

The general delivery process looks like this:

```text
WordPress
    |
    v
wp_mail()
    |
    v
PHPMailer
    |
    v
PHP mail() or configured SMTP transport
    |
    v
Mail Transfer Agent
    |
    v
Recipient Mail Server
    |
    v
Inbox or Spam Folder
```

WordPress does not operate a complete email delivery server by itself.

The server environment or configured external mail service handles actual message delivery.

A successful `wp_mail()` return value does not guarantee that the recipient received the message. It indicates that the sending process completed without reporting an error.

## How wp_mail() Works

The basic function signature is:

```php
wp_mail(
    $to,
    $subject,
    $message,
    $headers,
    $attachments
);
```

Example:

```php
$to      = 'customer@example.com';
$subject = 'Test Email';
$message = 'This is a WordPress email test.';

$headers = array(
    'Content-Type: text/plain; charset=UTF-8',
);

$result = wp_mail(
    $to,
    $subject,
    $message,
    $headers
);

if ( $result ) {
    error_log( 'WordPress email sending process completed.' );
} else {
    error_log( 'WordPress email sending failed.' );
}
```

Do not interpret a `true` result as proof of inbox delivery. Check the receiving mail server and delivery logs separately.

## Testing WordPress Email

Create a temporary test function in a development environment:

```php
function example_send_test_email() {

    $to      = 'your-test@example.com';
    $subject = 'WordPress Email Diagnostic';
    $message = 'Testing WordPress email delivery.';

    $result = wp_mail(
        $to,
        $subject,
        $message
    );

    if ( $result ) {
        error_log( 'Email submission completed.' );
    } else {
        error_log( 'Email submission failed.' );
    }
}
```

Run the function through a controlled development workflow.

Do not expose a public URL that allows visitors to trigger unlimited email tests.

## Diagnosing Email Failures

Check the following areas in order.

### 1. WordPress application

- Verify that the email-sending function executes.
- Check the recipient address.
- Inspect the subject and message.
- Review custom email filters.
- Check whether another plugin intercepts email sending.

### 2. PHP environment

- Check PHP error logs.
- Verify mail-related configuration.
- Confirm that the hosting environment supports outgoing mail.
- Inspect restrictions imposed by the hosting provider.

### 3. SMTP server

- Verify the SMTP hostname.
- Check the SMTP port.
- Confirm the encryption method.
- Verify username and password.
- Check outbound network restrictions.

### 4. Receiving server

- Inspect delivery logs.
- Check rejection and bounce messages.
- Review spam filtering.
- Verify sender authentication.
- Inspect the original email headers.

## Logging Email Errors

WordPress provides the `wp_mail_failed` action for handling email failures.

Example:

```php
function example_log_mail_failure( $error ) {

    if ( ! is_wp_error( $error ) ) {
        return;
    }

    error_log(
        'WordPress email error: ' . $error->get_error_message()
    );
}

add_action(
    'wp_mail_failed',
    'example_log_mail_failure'
);
```

You can also monitor successful email submission:

```php
function example_log_mail_success( $mail_data ) {

    error_log(
        'WordPress email submission completed.'
    );
}

add_action(
    'wp_mail_succeeded',
    'example_log_mail_success'
);
```

Important:

- A successful submission does not prove inbox delivery.
- Avoid logging passwords, SMTP credentials, or message contents containing personal information.
- Store diagnostic logs securely.
- Disable verbose debugging after troubleshooting.

## SMTP Configuration

SMTP allows WordPress to submit messages through an external mail server instead of relying on the hosting server's default PHP mail transport.

Typical SMTP configuration requires:

| Setting | Example |
|---|---|
| SMTP Host | mail.example.com |
| SMTP Port | 587 |
| Encryption | STARTTLS |
| Authentication | Enabled |
| Username | sender@example.com |
| Password | Secure application credential |

These are illustrative values. Use the actual configuration supplied by your email provider.

### SMTP Troubleshooting

If SMTP authentication fails:

1. Verify the SMTP hostname.
2. Confirm the correct port.
3. Check SSL or TLS settings.
4. Verify the account credentials.
5. Check whether application passwords are required.
6. Confirm that SMTP access is enabled.
7. Inspect server firewall restrictions.
8. Review SMTP provider logs.

Never publish SMTP credentials in a public repository or commit them to version control.

## PHPMailer Diagnostics

WordPress uses PHPMailer internally to process outgoing messages.

The `phpmailer_init` action allows developers to inspect or configure the PHPMailer instance.

Example:

```php
add_action(
    'phpmailer_init',
    function ( $mailer ) {

        error_log(
            'PHPMailer transport: ' . $mailer->Mailer
        );

        error_log(
            'PHPMailer host: ' . $mailer->Host
        );
    }
);
```

This is useful when investigating whether another plugin or configuration is changing the mail transport.

### Inspect failed email errors

```php
add_action(
    'wp_mail_failed',
    function ( $error ) {

        if ( is_wp_error( $error ) ) {

            error_log(
                $error->get_error_message()
            );
        }
    }
);
```

Do not enable public SMTP debug output on a production website.

## Email Headers and Sender Identity

Incorrect sender identity can cause delivery and authentication problems.

WordPress provides filters for changing the default sender address and sender name.

### Change the sender address

```php
add_filter(
    'wp_mail_from',
    function ( $email ) {
        return 'notifications@example.com';
    }
);
```

### Change the sender name

```php
add_filter(
    'wp_mail_from_name',
    function ( $name ) {
        return 'Example Website';
    }
);
```

Use an address belonging to a domain that you control and have configured with your mail provider.

### Add Reply-To

```php
$headers = array(
    'Reply-To: Support Team <support@example.com>',
);

wp_mail(
    'customer@example.com',
    'Support Request',
    'Thank you for contacting us.',
    $headers
);
```

Avoid using a visitor's email address as the sender address in contact form notifications. Use your authenticated domain address as the sender and the visitor's address as Reply-To.

## SPF, DKIM, and DMARC

Email authentication helps receiving servers verify that messages are authorized to use your domain.

### SPF

Sender Policy Framework identifies which mail servers are authorized to send email for a domain.

An illustrative SPF record:

```text
v=spf1 include:mail-provider.example ~all
```

Replace the example provider with the actual provider's SPF instructions.

Do not create multiple SPF records for the same domain. Update the existing record when adding another authorized sender.

### DKIM

DomainKeys Identified Mail uses a cryptographic signature to help receiving servers verify message authenticity and integrity.

The mail provider normally supplies:

- DKIM selector.
- Public key.
- DNS record name.
- DNS TXT record value.

Publish the supplied record through your DNS provider and verify that outgoing messages contain a valid DKIM signature.

### DMARC

Domain-based Message Authentication, Reporting, and Conformance tells receiving servers how to handle messages that fail authentication checks.

An illustrative monitoring record:

```text
v=DMARC1; p=none; rua=mailto:dmarc@example.com
```

Start with a monitoring policy when appropriate. Review reports and verify all legitimate sending services before moving to quarantine or reject policies.

### Authentication comparison

| Method | Main purpose |
|---|---|
| SPF | Identifies authorized sending servers |
| DKIM | Verifies a cryptographic message signature |
| DMARC | Defines policy and checks domain alignment |

For reliable delivery, configure SPF and DKIM and establish an appropriate DMARC policy.

## Email Deliverability Problems

Email submission and email deliverability are different stages.

A message can leave WordPress successfully and still be rejected, delayed, or placed in spam.

Investigate:

- Sender domain reputation.
- SPF authentication results.
- DKIM signature validation.
- DMARC alignment.
- SMTP server reputation.
- Reverse DNS and PTR records.
- Message content and formatting.
- Recipient server rejection responses.
- Spam complaints and bounce rates.

Inspect the original message headers to identify authentication results and the delivery path.

For Gmail recipients, review the current Google email sender requirements.

## Common Troubleshooting Scenarios

### Scenario 1: WordPress sends no emails

Possible causes:

- PHP mail transport is unavailable.
- SMTP is incorrectly configured.
- Outgoing connections are blocked.
- A plugin is intercepting email.
- The server rejects the message.

Recommended investigation:

1. Trigger a controlled test email.
2. Check `wp_mail()` return status.
3. Inspect `wp_mail_failed` logs.
4. Review hosting mail logs.
5. Test SMTP connectivity.
6. Check provider rejection messages.

### Scenario 2: Emails arrive in spam

Possible causes:

- Missing SPF or DKIM.
- DMARC alignment failures.
- Poor sender reputation.
- Misleading sender identity.
- Spam-like message content.

Recommended investigation:

1. Inspect the original message headers.
2. Review SPF authentication results.
3. Verify the DKIM signature.
4. Check DMARC alignment.
5. Review sending domain reputation.
6. Inspect provider delivery reports.

### Scenario 3: Password reset emails are missing

Investigate:

- Whether the password reset process completes.
- Whether the recipient address is correct.
- Whether `wp_mail()` is called.
- Whether the mail provider rejects the message.
- Whether the message is delayed or filtered.

Avoid changing WordPress authentication code before confirming where the email delivery process fails.

### Scenario 4: SMTP authentication fails

Investigate:

- Incorrect credentials.
- Incorrect SMTP hostname.
- Wrong encryption settings.
- Disabled SMTP authentication.
- Provider security restrictions.
- Application password requirements.

### Scenario 5: Emails have the wrong sender address

Investigate:

- `wp_mail_from` filters.
- SMTP plugin settings.
- Hosting mail configuration.
- Sender restrictions imposed by the mail provider.
- Domain authentication and alignment.

## Security Best Practices

1. Never hardcode SMTP passwords in plugin or theme files.
2. Use environment variables or secure configuration storage.
3. Use TLS for SMTP connections.
4. Validate recipient addresses.
5. Avoid exposing mail server debug information.
6. Protect public email-testing endpoints.
7. Avoid sending sensitive information through diagnostic logs.
8. Use authenticated sender addresses.
9. Keep SPF, DKIM, and DMARC records maintained.
10. Monitor bounces, spam complaints, and delivery failures.

## Troubleshooting Checklist

- [ ] Confirm that the email function executes.
- [ ] Verify the recipient address.
- [ ] Check the `wp_mail()` return value.
- [ ] Inspect `wp_mail_failed` errors.
- [ ] Review PHP and hosting mail logs.
- [ ] Verify SMTP hostname and port.
- [ ] Confirm TLS and authentication settings.
- [ ] Inspect PHPMailer configuration.
- [ ] Check sender address and Reply-To headers.
- [ ] Verify SPF records.
- [ ] Verify DKIM signatures.
- [ ] Review DMARC policy and alignment.
- [ ] Inspect original message headers.
- [ ] Check bounce and rejection messages.
- [ ] Test delivery to multiple email providers.

## References

- [WordPress wp_mail() Reference](https://developer.wordpress.org/reference/functions/wp_mail/)
- [WordPress Mail Administration Handbook](https://developer.wordpress.org/advanced-administration/server/mail/)
- [PHPMailer Documentation](https://github.com/PHPMailer/PHPMailer)
- [Google Email Sender Guidelines](https://support.google.com/mail/answer/81126)
- [Google SPF Setup Guide](https://knowledge.workspace.google.com/admin/security/set-up-spf)
- [Google DMARC Setup Guide](https://knowledge.workspace.google.com/admin/security/set-up-dmarc)

## License

This repository is intended for educational and technical reference purposes.
