# Spring Boot 4 Translation: MessagingRequirement

Map **MessagingRequirement** to JMS with Spring Boot 4 (JmsClient, `@JmsListener`). For the core translation guide, see [../SKILL.md](../SKILL.md).

## Patterns

- **Consumer:** `@JmsListener(destination = "...")` on a method; use `JmsClient` for send. Map destination, acknowledgment, error handling from spec.
- **Producer:** `JmsClient.send(destination, payload)` or equivalent. Map retry and error handling.
- **Configuration:** `spring.artemis.*` or broker URL from ExternalResourceRequirement; for Cloud Foundry use VCAP bindings.

**Consumer error handling:** read `JMSXDeliveryCount`; when it reaches `legacy_config.max_redeliveries`, send the message to `error_handling.dead_letter_queue` instead of rethrowing. Rethrow below that count so the broker redelivers.
