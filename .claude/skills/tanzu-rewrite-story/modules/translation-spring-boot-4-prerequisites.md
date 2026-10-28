# Spring Boot 4 Translation: Prerequisites

Load this document when generating or validating the project POM. For the core translation guide, see [../SKILL.md](../SKILL.md).

## Project bootstrap via start.spring.io

**When to use**: Creating a new project (target folder is empty—no `pom.xml` and no `build.gradle`—or first translation) or when the user explicitly asks to (re)initialize the project.

Use [start.spring.io](https://start.spring.io) to generate the initial Maven or Gradle project, then unzip into the target folder. Dependencies are derived from the spec: `spring_boot_recommendations.starters`, per-entity `starter_dependency`, and schema inference. start.spring.io expects **short dependency IDs** (e.g. `web`, `data-jpa`), not Maven artifact IDs.

### Starter → Initializr ID mapping

| Spec / POM starter | start.spring.io dependency ID |
|--------------------|--------------------------------|
| spring-boot-starter-webmvc / spring-boot-starter-web | web |
| spring-boot-starter-data-jpa | data-jpa |
| spring-boot-starter-security | security |
| spring-boot-starter-artemis | artemis |
| spring-boot-starter-actuator | actuator |
| spring-boot-starter-thymeleaf | thymeleaf |
| spring-boot-starter-test | (Initializr includes test; add explicitly only if needed) |
| spring-shell-starter | shell |
| spring-boot-starter-batch | batch |
| — **`batch` requires `jdbc` or `data-jpa`** to provide `PlatformTransactionManager` (see [translation-spring-boot-4-batch.md](translation-spring-boot-4-batch.md)); without it the context fails to start | jdbc / data-jpa |
| spring-boot-starter-validation | validation |
| spring-boot-starter-jdbc | jdbc |
| spring-restclient (HTTP client) | spring-restclient |
| spring-boot-starter-data-ldap (LdapTemplate, LdapContextSource) | data-ldap |
| spring-security-ldap (LdapBindAuthenticationManagerFactory, DefaultLdapAuthoritiesPopulator) | **Add manually post-unzip** — not on start.spring.io: `org.springframework.security:spring-security-ldap` |
| com.unboundid:unboundid-ldapsdk (embedded LDAP for tests) | **Add manually post-unzip** (test scope): `com.unboundid:unboundid-ldapsdk` |
| Lombok | lombok |
| Spring Boot DevTools | devtools |
| Flyway / Liquibase | flyway / liquibase |
| PostgreSQL / MySQL / H2 driver | postgresql / mysql / h2 |
| Testcontainers (for persistence/integration tests) | testcontainers |

Map `spring_boot_recommendations.additional_dependencies` and DB/driver needs from the spec to these IDs. **JSpecify** is not on start.spring.io; add it to the build file after unzip if the skill requires it.

**Include `testcontainers` in the `dependencies` parameter when the story involves persistence** (e.g. JPA entities, data-jpa, or repository tests), so the generated project has Testcontainers available for integration tests.

### Parameter derivation

- **groupId**: Default `com.example` unless the spec or user provides one.
- **artifactId** / **name**: From `application_name` in the spec, sanitized to a valid artifact name (e.g. `order-management`).
- **packageName**: e.g. `com.example.ordermanagement` or `com.example.app`; must match `src/main/java` layout.
- **description**: From spec or "Spring Boot 4 application".
- **dependencies**: Comma-separated Initializr IDs, no spaces (e.g. `web,data-jpa,actuator,testcontainers`). Include `testcontainers` when the story involves persistence.

### Generate a Maven/Java project (zip)

```bash
curl -G https://start.spring.io/starter.zip \
  -d type=maven-project \
  -d language=java \
  -d bootVersion=4.0.2 \
  -d groupId=com.example \
  -d artifactId=app-name \
  -d name=app-name \
  -d description="Spring Boot 4 application" \
  -d packageName=com.example.app \
  -d packaging=jar \
  -d javaVersion=17 \
  -d dependencies=web,data-jpa,actuator,testcontainers \
  -o app.zip
```

### Generate a Gradle/Java project (zip)

```bash
curl -G https://start.spring.io/starter.zip \
  -d type=gradle-project \
  -d language=java \
  -d bootVersion=4.0.2 \
  -d groupId=com.example \
  -d artifactId=app-name \
  -d name=app-name \
  -d description="Spring Boot 4 application" \
  -d packageName=com.example.app \
  -d packaging=jar \
  -d javaVersion=17 \
  -d dependencies=web,data-jpa,actuator,testcontainers \
  -o app.zip
```

Replace `groupId`, `artifactId`, `name`, `description`, `packageName`, and `dependencies` with values derived from the spec. Use the mapping table above to convert spec starters to comma-separated Initializr IDs.

### Unzip and enter the project

```bash
unzip -o app.zip -d <target-folder>
```

If the zip contains a single root directory (e.g. `app-name/`), either unzip so that its contents are placed directly in the target folder (e.g. `unzip -o app.zip -d <target> && mv <target>/app-name/* <target>/ && rmdir <target>/app-name`), or treat the subdirectory as the project root so generated code goes under `src/main/java/<packageName>/`. The project root must be the target folder for the skill's generated paths.

### After unzip

- Generate controllers, services, entities, tests, and Cloud Foundry config into the target folder as usual.
- Add dependencies not available on start.spring.io (e.g. JSpecify) to `pom.xml` or `build.gradle` manually after unzip.

### ⚠️ Post-unzip: validate and fix the generated build file

**start.spring.io may generate invalid artifact IDs for Spring Boot 4.** Before running the first build, scan the generated `pom.xml` or `build.gradle` for the entries in the table below and apply the fix column.

| Do NOT use (start.spring.io may generate) | Fix |
|---|---|
| `spring-boot-h2console` | Replace with `com.h2database:h2` (scope: `runtime`) |
| Any `spring-boot-starter-*-test` artifact (e.g. `spring-boot-starter-webmvc-test`, `spring-boot-starter-data-jpa-test`, `spring-boot-starter-actuator-test`) | Remove entirely — `spring-boot-starter-test` already covers all test needs |
| `org.testcontainers:testcontainers-junit-jupiter` | Replace with `org.testcontainers:junit-jupiter` |
| `org.testcontainers:junit-jupiter` without an explicit version | Add explicit version `1.20.4` — it is **not** managed by the Spring Boot BOM |

#### Correct Testcontainers block (Maven)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>1.20.4</version>
    <scope>test</scope>
</dependency>
```

#### Correct Testcontainers block (Gradle)

```groovy
testImplementation 'org.springframework.boot:spring-boot-testcontainers'
testImplementation 'org.testcontainers:junit-jupiter:1.20.4'
```

Or with the Kotlin DSL:

```kotlin
testImplementation("org.springframework.boot:spring-boot-testcontainers")
testImplementation("org.testcontainers:junit-jupiter:1.20.4")
```

---

## Spring Boot 4 Baseline Requirements

| Requirement | Version | Notes |
|-------------|---------|-------|
| Java | 17 minimum, 25 recommended | Virtual threads available with Java 21+ |
| Jakarta EE | 11 | `jakarta.*` packages required |
| Spring Framework | 7.0 | Foundation for Spring Boot 4 |
| Spring Batch | 6.0.x | Shipped by `spring-boot-starter-batch` 4.x; core packages reorganized vs. 5.x — see [translation-spring-boot-4-batch.md](translation-spring-boot-4-batch.md) |
| Spring Security | 7.0 | Component-based configuration |
| Hibernate | 7.1 | JPA 3.2 alignment |
| Jackson | 3.x | `tools.jackson` package |
| JUnit | 6 | Updated annotations and lifecycle |

## Modular Starter Dependencies

Use the **full POM/Gradle reference below** when:
- **Adding dependencies to an existing project** (subsequent stories): add `<dependency>` blocks or Gradle coordinates instead of re-scaffolding.
- **Adding dependencies not on start.spring.io** (e.g. JSpecify): add manually after unzip.

Spring Boot 4 uses modular starters. Only include what you need:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>4.0.1</version>
    </parent>
    
    <groupId>com.example</groupId>
    <artifactId>my-application</artifactId>
    <version>1.0.0</version>
    
    <properties>
        <java.version>25</java.version>
    </properties>
    
    <dependencies>
        <!-- Web MVC - only if you need web endpoints -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-webmvc</artifactId>
        </dependency>
        
        <!-- Data JPA - only if you need database access -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        
        <!-- Security - only if you need authentication/authorization -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        
        <!-- JMS - only if you need messaging -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-artemis</artifactId>
        </dependency>
        
        <!-- Actuator - for health checks and observability -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        
        <!-- Thymeleaf - only if you need server-side rendering -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>
        
        <!-- Spring Shell - only if you need CLI commands -->
        <dependency>
            <groupId>org.springframework.shell</groupId>
            <artifactId>spring-shell-starter</artifactId>
        </dependency>
        
        <!-- JSpecify for null safety -->
        <dependency>
            <groupId>org.jspecify</groupId>
            <artifactId>jspecify</artifactId>
            <version>1.0.0</version>
        </dependency>
        
        <!-- Test dependencies -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```
