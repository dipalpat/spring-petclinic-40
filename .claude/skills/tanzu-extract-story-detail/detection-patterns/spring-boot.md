# Spring Boot 2.x / 3.x / 4.x Patterns

This document defines detection patterns for applications already built with Spring Boot (2.x, 3.x, or 4.x). Detection is based **only on code and configuration**—do not use `pom.xml` or `build.gradle`, because Spring Boot dependencies are often **transitive** (the app may not declare Boot starters directly).

These patterns enable extraction and translation to treat the app as a "Boot → Boot 4" rewrite (behavior-preserving, same-test validation) rather than legacy-only.

## Table of Contents

- [Main Class and Bootstrap](#main-class-and-bootstrap)
- [Configuration Files](#configuration-files)
- [Web Layer](#web-layer)
- [Data Layer](#data-layer)
- [Version Discrimination (Code-Only)](#version-discrimination-code-only)

---

## Main Class and Bootstrap

### @SpringBootApplication

| Pattern | `@SpringBootApplication` |
|---------|--------------------------|
| **File Types** | `*.java` |
| **Indicates** | Spring Boot application main class |
| **Schema Entity** | `ModernizationRequirements.source_technology` (Spring Boot); record Boot version when inferable |

**Example:**
```java
@SpringBootApplication
public class OrdersApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrdersApplication.class, args);
    }
}
```

**Extraction Notes:** Presence of this annotation indicates a Spring Boot application. Map entry points and config to the same Technical Context Schema (ServiceEndpoint, ServiceOperation, DataStructure, etc.). Record Boot version from code (e.g. javax vs jakarta) when possible.

---

### SpringApplication.run

| Pattern | `SpringApplication\.run\s*\(` |
|---------|-----------------------------|
| **File Types** | `*.java` |
| **Indicates** | Spring Boot bootstrap (main method) |
| **Schema Entity** | Confirms Spring Boot application |

**Example:**
```java
SpringApplication.run(OrdersApplication.class, args);
```

**Extraction Notes:** Used with `@SpringBootApplication` to confirm Boot app. Do not rely on build files for version.

---

### Main Class Naming

| Pattern | `*Application\.java` (filename) |
|---------|-------------------------------|
| **File Types** | `*.java` |
| **Indicates** | Possible Boot main class (when combined with @SpringBootApplication) |
| **Schema Entity** | Application entry point |

**Extraction Notes:** Class name ending in `Application` is a common Boot convention; use with annotation patterns above.

---

## Configuration Files

### application.properties / application.yml

| Pattern | `application\.properties` or `application\.yml` (filename) |
|---------|-----------------------------------------------------------|
| **File Types** | `application.properties`, `application.yml` |
| **Indicates** | Spring Boot configuration |
| **Schema Entity** | `ExternalResourceRequirement`, `IntegrationPoint` (when config references datasources, URLs, etc.) |

**Example:**
- `src/main/resources/application.yml`
- `src/main/resources/application.properties`

**Extraction Notes:** Parse for datasource URLs, server port, profiles, and other config. Map to schema; do not use build files to infer Boot version.

---

### Profile-Specific Config

| Pattern | `application-\w+\.(properties|yml)` (filename) |
|---------|--------------------------------------------------|
| **File Types** | `application-*.properties`, `application-*.yml` |
| **Indicates** | Spring Boot profile-specific configuration |
| **Schema Entity** | Same as main config; note profile name |

**Example:**
- `application-dev.yml`, `application-cloud.properties`

**Extraction Notes:** Indicates Boot-style profile config. Extract keys relevant to schema (datasources, integrations).

---

## Web Layer

### @RestController

| Pattern | `@RestController` |
|---------|--------------------|
| **File Types** | `*.java` |
| **Indicates** | Spring Boot / Spring MVC REST controller |
| **Schema Entity** | `ServiceEndpoint` |

**Example:**
```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    @GetMapping("/{id}")
    public Order getOrder(@PathVariable Long id) { ... }
}
```

**Extraction Notes:** Extract `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping` for endpoints. Map to ServiceEndpoint (path, method, request/response, auth).

---

### Boot-Style HTTP Method Mappings

| Pattern | `@(Get|Post|Put|Delete|Patch)Mapping\s*\(` |
|---------|---------------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | REST endpoint (Boot/Spring MVC) |
| **Schema Entity** | `ServiceEndpoint` |

**Example:**
```java
@GetMapping("/{id}")
@PostMapping
@PutMapping("/{id}")
@DeleteMapping("/{id}")
```

**Extraction Notes:** Combine with class-level `@RequestMapping` for full path. Extract path variables, query params, request/response types.

---

### RestTemplate / WebClient / RestClient

| Pattern | `RestTemplate|WebClient|RestClient` |
|---------|---------------------------|
| **File Types** | `*.java` |
| **Indicates** | HTTP client (Boot/Spring); RestClient is Spring 6+) |
| **Schema Entity** | `IntegrationPoint` (type REST_API, OUTBOUND) |

**Example:**
```java
private final RestTemplate restTemplate;
private final WebClient webClient;
```

**Extraction Notes:** Indicates outbound REST calls. Extract base URLs and endpoints from usage; map to IntegrationPoint.

---

## Data Layer

### JpaRepository / CrudRepository

| Pattern | `JpaRepository|CrudRepository|ListCrudRepository` |
|---------|-------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | Spring Data JPA repository (common in Boot apps) |
| **Schema Entity** | `DataStructure` (entity type), `ServiceOperation` (repository methods) |

**Example:**
```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByStatus(OrderStatus status);
}
```

**Extraction Notes:** When combined with `@SpringBootApplication`, indicates Boot app with Spring Data JPA. Map entity types to DataStructure; repository methods to operations. Do not infer Boot version from repository interfaces (use javax/jakarta in same codebase).

---

### @Entity (with Boot main class)

| Pattern | `@Entity` |
|---------|-----------|
| **File Types** | `*.java` |
| **Indicates** | JPA entity; in a Boot app, implies Boot Data JPA |
| **Schema Entity** | `DataStructure` (persistence_type ENTITY) |

**Example:**
```java
@Entity
@Table(name = "orders")
public class Order {
    @Id
    private Long id;
    // ...
}
```

**Extraction Notes:** Extract table name, columns, relationships, constraints. Use with other Boot patterns to confirm Boot app.

---

## Version Discrimination (Code-Only)

Do **not** use `pom.xml` or `build.gradle` for version. Use only code and config.

### javax vs jakarta Imports

| Pattern | `import javax\.` or `import jakarta\.` |
|---------|----------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | Java EE / Jakarta EE namespace: `javax.*` → Boot 2.x; `jakarta.*` → Boot 3.x/4.x |
| **Schema Entity** | Record Boot version when inferable (2, 3, or 4) |

**Example:**
- Boot 2.x: `import javax.persistence.*;`, `import javax.validation.*;`
- Boot 3.x/4.x: `import jakarta.persistence.*;`, `import jakarta.validation.*;`

**Extraction Notes:** Scan Java files for `javax.` vs `jakarta.` in imports. Predominantly `javax` → Boot 2.x; predominantly `jakarta` → Boot 3.x or 4.x. Record in requirements so translation can apply correct API and config changes.

---

### org.springframework.boot Package

| Pattern | `org\.springframework\.boot` |
|---------|-----------------------------|
| **File Types** | `*.java` |
| **Indicates** | Direct use of Spring Boot APIs (e.g. `SpringApplication`, `@SpringBootApplication`) |
| **Schema Entity** | Confirms Spring Boot; version from javax/jakarta elsewhere |

**Extraction Notes:** Confirms Boot app. Do not infer version from this package alone; use javax/jakarta for version.

---

## Usage

1. **Run Boot detection first** (see [usage-guidelines.md](usage-guidelines.md)): Check for `@SpringBootApplication` and Boot config files before treating the app as legacy Spring Web/EJB.
2. **Extract to same schema**: Map to ServiceEndpoint, ServiceOperation, DataStructure, IntegrationPoint, etc., as with other pattern files.
3. **Record Boot version** when inferable (javax → 2.x, jakarta → 3.x/4.x) so translation can produce a Boot 4 app that preserves behavior and supports the same tests.
