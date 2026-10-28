# Spring Boot 4 Translation: IntegrationPoint, ExternalResourceRequirement

Map **IntegrationPoint** (HTTP, SOAP) and **ExternalResourceRequirement** (datasource, messaging, etc.) to Spring Boot 4 clients and configuration. For the core translation guide, see [../SKILL.md](../SKILL.md).

## Patterns

- **HTTP client:** Use `@HttpExchange` interface and `RestClient` or `WebClient`. Map endpoint, authentication, retry from IntegrationPoint.
- **Datasource:** Map to `spring.datasource.*` and HikariCP (see [translation-spring-boot-4-cloud-foundry.md](translation-spring-boot-4-cloud-foundry.md) for CF bindings).
- **Messaging:** Map to `spring.artemis.*` or JMS config; see [translation-spring-boot-4-messaging.md](translation-spring-boot-4-messaging.md).
- **LDAP:** When `ldap_requirements` is present or `IntegrationPoint type: LDAP` is detected, load [translation-spring-boot-4-ldap.md](translation-spring-boot-4-ldap.md) for full patterns (LdapTemplate, LdapContextSource, LDAP auth, paging, sorting, TLS, Cloud Foundry config).
- **RMI/EJB Remote:** When `IntegrationPoint type: RMI_EJB` is detected, or `migration_considerations` includes `"Remote EJB Interface"` / `"EJB 2.x Remote Interface"` / `"EJB 2.x Remote Home"`, load [translation-spring-boot-4-ejb-remote.md](translation-spring-boot-4-ejb-remote.md) for full patterns (replacing `@Remote`, `EJBObject`, `EJBHome` with `@RestController` + `@Service`).

For a worked example, see [translation-spring-boot-4-example-rest-crud.md](translation-spring-boot-4-example-rest-crud.md).
