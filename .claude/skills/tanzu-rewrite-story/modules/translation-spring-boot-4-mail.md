# Spring Boot 4 Translation: MailRequirement

Map **MailRequirement** to `spring.mail.*` and `JavaMailSender`. For the core translation guide, see
[../SKILL.md](../SKILL.md).

## Patterns

- **Remove the lookup.** Delete the `InitialContext` lookup of `mail/...` and the `<resource-ref>` for
  `javax.mail.Session`. Inject `JavaMailSender` through the constructor.
- **Dependency:** `spring-boot-starter-mail`.
- **Configuration:** map `properties` to `spring.mail.*`.

| `MailRequirement.properties` | Spring Boot property |
|------------------------------|----------------------|
| `smtp_host` | `spring.mail.host` |
| `smtp_port` | `spring.mail.port` |
| `smtp_auth` | `spring.mail.properties.mail.smtp.auth` |
| `starttls_enabled` | `spring.mail.properties.mail.smtp.starttls.enable` |
| `smtp_timeout_ms` | `spring.mail.properties.mail.smtp.timeout` |

```yaml
spring:
  mail:
    host: smtp.example.com
    port: 587
    username: ${SMTP_USERNAME}
    password: ${SMTP_PASSWORD}
    properties:
      mail.smtp.auth: true
      mail.smtp.starttls.enable: true
```

```java
@Service
class NotificationService {

    private final JavaMailSender mailSender;

    NotificationService(JavaMailSender mailSender) {
        this.mailSender = mailSender;
    }

    void sendOrderConfirmation(String to, String orderId) {
        SimpleMailMessage message = new SimpleMailMessage();
        message.setTo(to);
        message.setSubject("Order " + orderId + " confirmed");
        message.setText("Your order is confirmed.");
        mailSender.send(message);
    }
}
```

Use `MimeMessageHelper` when the message needs HTML or an attachment.

## Cloud Foundry notes

- Credentials never go in `application.yml`. Read them from the environment, or from a bound user-provided
  service. Never carry over a password found in a vendor descriptor.
- Sending mail is a blocking network call. On a request thread it holds the thread for the duration of the
  SMTP exchange. For anything not needed in the response, send it from a `@Async` method or a queue consumer.
- An outbound SMTP port can be closed by the platform's egress policy. Name this as a deployment check.

## Reading mail

`JavaMailSender` sends only. When `MailRequirement` records a `store-protocol` (IMAP or POP3), the legacy
application also reads mail. There is no Spring Boot starter for that:

- Use the JavaMail `Store` API directly, or Spring Integration's mail inbound channel adapter.
- A polling reader must not run on every instance, or every instance processes the same message. Run it as a
  separate single-instance application, or take a distributed lock.
- Say this in the summary to the user. Do not silently drop the read path.

## Checklist

- [ ] No `javax.mail.Session` / `jakarta.mail.Session` lookup remains
- [ ] No credential is hardcoded in `application.yml`
- [ ] A recorded store protocol is either implemented or reported as out of scope
