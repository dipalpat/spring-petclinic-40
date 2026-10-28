# Spring Boot 4 Translation: ServiceTask

Map **ServiceTask** to `@Scheduled` and `@EnableScheduling`. For the core translation guide, see [../SKILL.md](../SKILL.md).

## Patterns

- **Cron:** `@Scheduled(cron = "${property}", zone = "...")` with optional concurrency guard (e.g. AtomicBoolean).
- **Fixed rate/delay:** `@Scheduled(fixedRate = ...)` or `fixedDelay`.
- **Error handling:** Map retry_count, notification_on_failure to try/catch and alert service.

**Cloud Foundry:** every instance runs the same `@Scheduled` method, so a job runs N times on N instances. Guard it with a distributed lock (ShedLock), or deploy the job as a separate single-instance application. State which one was chosen.
