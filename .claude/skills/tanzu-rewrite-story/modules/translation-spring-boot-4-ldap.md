# Spring Boot 4 Translation: LdapRequirement

Map **LdapRequirement** to Spring LDAP 4 (`LdapTemplate`, `LdapContextSource`) and Spring Security 7 LDAP (`LdapBindAuthenticationManagerFactory`, `DefaultLdapAuthoritiesPopulator`). For the core translation guide, see [SKILL.md](../SKILL.md).

---

## Dependencies

`spring-boot-starter-data-ldap` is on start.spring.io (ID: `data-ldap`). `spring-security-ldap` must be added manually post-unzip — it is not on start.spring.io.

**Maven:**

```xml
<!-- Spring LDAP (LdapTemplate, LdapContextSource, Spring Data LDAP repositories) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-ldap</artifactId>
</dependency>

<!-- Spring Security LDAP (LdapBindAuthenticationManagerFactory, DefaultLdapAuthoritiesPopulator) -->
<!-- Add manually post-unzip: not on start.spring.io -->
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-ldap</artifactId>
</dependency>

<!-- Embedded LDAP server for tests (UnboundID) -->
<dependency>
    <groupId>com.unboundid</groupId>
    <artifactId>unboundid-ldapsdk</artifactId>
    <scope>test</scope>
</dependency>
```

**Gradle:**

```groovy
implementation 'org.springframework.boot:spring-boot-starter-data-ldap'
implementation 'org.springframework.security:spring-security-ldap'  // add manually
testRuntimeOnly 'com.unboundid:unboundid-ldapsdk'
```

---

## application.yml — LdapRequirement Field Mapping

| LdapRequirement field | Spring Boot property |
|-----------------------|----------------------|
| `provider_url` | `spring.ldap.urls` |
| `search_base` | `spring.ldap.base` |
| `bind_dn` | `spring.ldap.username` |
| credentials (from `credentials_source`) | `spring.ldap.password` (use `${LDAP_PASSWORD}`) |
| `referral_handling: IGNORE` | `spring.ldap.base-environment.java.naming.referral: ignore` |
| `referral_handling: FOLLOW` | `spring.ldap.base-environment.java.naming.referral: follow` |
| `referral_handling: THROW` | `spring.ldap.base-environment.java.naming.referral: throw` |

**Example (standard LDAP with STARTTLS):**

```yaml
spring:
  ldap:
    urls: ldap://ldap.example.com:389
    base: dc=example,dc=com
    username: cn=svc-app,ou=serviceaccounts,dc=example,dc=com
    password: ${LDAP_PASSWORD}
    base-environment:
      java.naming.referral: ignore
```

**Example (LDAPS — SSL from start):**

```yaml
spring:
  ldap:
    urls: ldaps://ldap.example.com:636
    base: dc=example,dc=com
    username: cn=svc-app,ou=serviceaccounts,dc=example,dc=com
    password: ${LDAP_PASSWORD}
```

---

## TLS Mode

### LDAPS (`tls_mode: LDAPS`)

Use `ldaps://` in `spring.ldap.urls`. No additional bean required. Ensure the server certificate is trusted (import into JVM truststore or use `javax.net.ssl.trustStore`).

### STARTTLS (`tls_mode: STARTTLS`)

Declare a `DefaultTlsDirContextAuthenticationStrategy` bean and associate it to `LdapContextSource`. Spring Boot auto-configuration will pick it up if it is available in the context.

```java
@Configuration
public class LdapConfig {

    @Bean
    public DirContextAuthenticationStrategy tlsAuthenticationStrategy() {
        return new DefaultTlsDirContextAuthenticationStrategy();
    }
}
```

The auto-configured `LdapContextSource` will detect the `DirContextAuthenticationStrategy` bean and associate it automatically.

---

## LDAP Authentication (`authentication_use: BIND` or `PASSWORD_COMPARE`)

When `SecurityRequirement.authentication_method = LDAP`, generate an `AuthenticationManager` bean alongside the `SecurityFilterChain`.

### Bind Authentication (most common)

```java
@Bean
AuthenticationManager ldapAuthenticationManager(BaseLdapPathContextSource contextSource) {
    LdapBindAuthenticationManagerFactory factory =
        new LdapBindAuthenticationManagerFactory(contextSource);
    // Use search filter when users are not at a predictable DN:
    factory.setUserSearchFilter("(uid={0})");
    factory.setUserSearchBase("ou=people");
    // OR use DN pattern when all users are under a fixed OU:
    // factory.setUserDnPatterns("uid={0},ou=people");
    factory.setLdapAuthoritiesPopulator(ldapAuthoritiesPopulator(contextSource));
    return factory.createAuthenticationManager();
}
```

- Use `setUserSearchFilter` + `setUserSearchBase` when users are spread across subtrees
- Use `setUserDnPatterns` when all users are in a single OU and the DN is predictable

### Password Compare Authentication

Use when the application performs a local password hash comparison rather than an LDAP bind:

```java
@Bean
AuthenticationManager ldapAuthenticationManager(BaseLdapPathContextSource contextSource) {
    LdapPasswordComparisonAuthenticationManagerFactory factory =
        new LdapPasswordComparisonAuthenticationManagerFactory(
            contextSource, new BCryptPasswordEncoder());
    factory.setUserDnPatterns("uid={0},ou=people");
    factory.setPasswordAttribute("userPassword");
    return factory.createAuthenticationManager();
}
```

### Active Directory (`provider_url` points to AD, `spring_boot_recommendation` mentions AD)

```java
@Bean
ActiveDirectoryLdapAuthenticationProvider adAuthProvider() {
    return new ActiveDirectoryLdapAuthenticationProvider(
        "example.com",                    // AD domain
        "ldap://company.example.com/"     // LDAP URL
    );
}
```

Active Directory uses `user@domain` format — `UserDnPatterns` is not required.

---

## Group / Role Retrieval (`group_search_base`, `group_search_filter`)

```java
@Bean
LdapAuthoritiesPopulator ldapAuthoritiesPopulator(BaseLdapPathContextSource contextSource) {
    DefaultLdapAuthoritiesPopulator populator =
        new DefaultLdapAuthoritiesPopulator(
            contextSource,
            "ou=groups,dc=example,dc=com"   // group_search_base
        );
    populator.setGroupSearchFilter("member={0}");  // group_search_filter
    populator.setRolePrefix("ROLE_");
    return populator;
}
```

---

## Programmatic Directory Search (`DirContext` → `LdapTemplate`)

Map `search_base`, `search_filter`, and `returning_attributes` to `LdapTemplate.search()`:

```java
@Service
public class UserDirectoryService {

    private final LdapTemplate ldapTemplate;

    public UserDirectoryService(LdapTemplate ldapTemplate) {
        this.ldapTemplate = ldapTemplate;
    }

    public List<UserDto> findByUid(String uid) {
        return ldapTemplate.search(
            "ou=people",                          // search_base (relative to spring.ldap.base)
            "(uid=" + uid + ")",                  // search_filter
            (Attributes attrs) -> new UserDto(    // AttributesMapper
                (String) attrs.get("uid").get(),
                (String) attrs.get("cn").get(),
                (String) attrs.get("mail").get()
            )
        );
    }
}
```

For a typed query builder approach:

```java
LdapQuery query = LdapQueryBuilder.query()
    .base("ou=people")
    .filter("(uid={0})", uid);
return ldapTemplate.search(query, new UserAttributesMapper());
```

---

## Paging (`paging_enabled: true`, `page_size`)

```java
public List<UserDto> findAllPaged() {
    PagedResultsDirContextProcessor processor =
        new PagedResultsDirContextProcessor(100);  // page_size
    List<UserDto> allResults = new ArrayList<>();

    do {
        List<UserDto> page = ldapTemplate.search(
            "ou=people", "(objectClass=person)", mapper, processor);
        allResults.addAll(page);
    } while (processor.hasMore());

    return allResults;
}
```

---

## Sorting (`sort_enabled: true`, `sort_attributes`)

```java
public List<UserDto> findAllSorted() {
    SortControlDirContextProcessor processor =
        new SortControlDirContextProcessor("cn");  // sort_attributes[0]
    return ldapTemplate.search("ou=people", "(objectClass=person)", mapper, processor);
}
```

---

## LDAP Events (`event_listeners` non-empty, `migration_considerations: "LDAP Events"`)

Spring LDAP 4 has no direct equivalent for `NamingListener` / `UnsolicitedNotificationListener`. Emit a TODO and implement polling:

```java
// TODO: Legacy code used NamingListener/UnsolicitedNotificationListener for
// real-time LDAP change notification. Spring LDAP has no direct equivalent.
// Implement change detection via polling with @Scheduled + LdapTemplate.search(),
// or enable LDAP Persistent Search if the server supports it.

@Scheduled(fixedDelay = 60_000)
public void pollDirectoryChanges() {
    // Search for recently modified entries and process changes
    ldapTemplate.search(
        "ou=people",
        "(modifyTimestamp>=" + lastPollTimestamp + ")",
        mapper
    );
}
```

---

## Cloud Foundry

LDAP connection details (URL, bind DN, password) must be externalized via environment variables or a Config Server binding — never hardcoded.

**manifest.yml env block:**

```yaml
env:
  LDAP_PASSWORD: ((ldap-service-password))
  SPRING_LDAP_URLS: ldap://ldap.example.com:389
  SPRING_LDAP_USERNAME: cn=svc-app,ou=serviceaccounts,dc=example,dc=com
```

**application.yml (cloud profile):**

```yaml
---
spring:
  config:
    activate:
      on-profile: cloud
  ldap:
    urls: ${SPRING_LDAP_URLS}
    username: ${SPRING_LDAP_USERNAME}
    password: ${LDAP_PASSWORD}
```

---

## Test Support — Embedded LDAP (UnboundID)

Use `spring.ldap.embedded.*` to start an in-memory LDAP server for integration tests. Place a `schema.ldif` and `test-users.ldif` on the test classpath.

```yaml
# src/test/resources/application-test.yml
spring:
  ldap:
    embedded:
      base-dn: dc=example,dc=com
      ldif: classpath:test-users.ldif
      port: 0   # random port
```

```java
@SpringBootTest
@ActiveProfiles("test")
class UserDirectoryServiceIT {
    // LdapTemplate is auto-configured against the embedded server
}
```
