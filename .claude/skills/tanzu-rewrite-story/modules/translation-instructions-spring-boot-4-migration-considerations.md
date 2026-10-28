# Spring Boot 4 Translation: migration_considerations

`ModernizationRequirements.migration_considerations` is a list of cross-cutting concerns that the extraction
phase found. Each value names a topic that changes the generated application beyond one entity. Load this file
when the array is not empty, then follow the row for each value present. For the core translation guide, see
[../SKILL.md](../SKILL.md).

## Topics

| Value | What it means | What to do |
|-------|---------------|------------|
| `Session State` | The application keeps state in the HTTP session, a `@Stateful` bean, or a JSF `@SessionScoped` bean. | Load [translation-spring-boot-4-session.md](translation-spring-boot-4-session.md) — it holds the full guidance: Spring Session with Redis, the session-ID resolution flow, caches on the same Redis, the Cloud Foundry service binding, and the serialization checks. Externalize the state; do not rely on one instance's heap or on sticky sessions. |
| `Caching` | The application uses in-process caching (Caffeine/Simple/Ehcache), embedded cluster grids (Infinispan), or app-server cache managers. | Load [translation-spring-boot-4-cache.md](translation-spring-boot-4-cache.md) — migrate to Spring Cache with Redis, configure `spring.cache.type: redis`, bind CF Redis service, and verify domain classes are Serializable. |
| `Server Migration` | The application ships application-server descriptors or uses vendor packages. | Delete every descriptor named in `ServerMigrationRequirement.deployment_descriptors`. Replace every package in `vendor_dependencies`. `<context-root>` becomes `server.servlet.context-path`. Never carry a vendor JAR into the target build file. |
| `UI Migration` | The view layer is JSP, Tiles, Struts, or JSF, none of which Spring Boot supports directly. | Load [translation-spring-boot-4-ui.md](translation-spring-boot-4-ui.md) and follow `ViewRequirement.migration_target`. |
| `Distributed Transactions` | An XA datasource or a `UserTransaction` was detected. | Load [translation-spring-boot-4-transactions.md](translation-spring-boot-4-transactions.md). Check `TransactionRequirement.scope` first, and ask the user while it is `REQUIRES_USER_FEEDBACK`. |
| `Vendor Monitoring` | The application reads JMX MBeans, calls a vendor management API, or reads vendor system properties. | Load [translation-spring-boot-4-monitoring.md](translation-spring-boot-4-monitoring.md). Tag every metric per instance. |
| `LDAP Events` | The code registers a `NamingListener` for directory change events. | Load [translation-spring-boot-4-ldap.md](translation-spring-boot-4-ldap.md). Spring LDAP has no direct equivalent. Emit a `TODO` and offer polling with `@Scheduled` + `LdapTemplate.search()`. |
| `JNDI Enumeration` | The code walks the JNDI tree with `NamingEnumeration` / `NameClassPair`. | There is no JNDI tree in the target. The resources the code was discovering are now injected beans. Replace the walk with explicit injection, and report any resource the spec does not name. |

An unknown value is not an error. Report it to the user and continue.

## Cloud Foundry cross-check

Every topic above exists because a legacy pattern assumed one long-lived server process. Before finishing,
check the generated application against the two facts that break that assumption:

1. **More than one instance runs the same code.** Anything held in one instance's memory or local disk is
   invisible to the others: sessions, caches, scheduled-job locks, in-process counters, uploaded files.
2. **Any instance can be replaced at any moment.** A rolling deploy, `cf restage`, a crash restart, or an
   autoscaler scale-down destroys the container. Nothing on its filesystem survives.

For each one found, either externalize the state or say plainly in the summary that it is unresolved. Do not
generate code that appears to work on one instance and fails on four.

## Reporting

List every topic that was present and what was done for it. When a topic could not be resolved — an
unresolved XA scope, a session attribute that cannot be serialized, a vendor property with no equivalent —
name it as an open item rather than leaving it implicit in the code.
