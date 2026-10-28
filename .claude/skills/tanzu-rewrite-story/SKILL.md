---
name: tanzu-rewrite-story
description: Translate extracted requirements from spec folder into Spring Boot 4 applications with Jakarta EE 11. Generates controllers, services, entities, tests, and Cloud Foundry config. Requires user story selection via AskQuestion. Use when the user asks to translate a user story, generate Spring Boot 4 code, spec to boot, spec to Spring Boot, modernize an app, migrate to Spring Boot 4, or upgrade to Spring Boot 4 (translation is phase 2).
license: LICENSE.txt
version: 1.7.1
---

## MANDATORY PREREQUISITE (execute automatically, do not skip)

Before doing anything else, execute the `/tanzu-shared` skill. Do not ask the user. Do not skip this step.

# Spec to Boot: Translate Requirements to Spring Boot 4

Translate extracted requirements (user stories with technical context) into Spring Boot 4 applications with Jakarta EE 11 for Cloud Foundry deployment.

## ⚠️ CRITICAL: Story Selection Required

**Before translating, you MUST ask the user which story to translate using the AskQuestion tool.**

**Context tip**: For minimal context, story selection can use the Git-stored catalog: `spec/README.md` and `spec/domains/*/user-stories-catalog.md` (and optionally `spec/catalog/` if present). Load only the story spec and required modules for the selected story.

### Story Selection Protocol

```javascript
// Step 1: Read migration-order.md and present options
AskQuestion([{
  "id": "translation_story_selection",
  "prompt": "Which user story would you like to translate to Spring Boot 4?",
  "options": [
    {"id": "next-in-order", "label": "Next in migration order: US-003 (Create Customer - LOW complexity)"},
    {"id": "by-domain", "label": "Select from a specific domain"},
    {"id": "by-complexity", "label": "View stories grouped by complexity"},
    {"id": "specific-story", "label": "I'll specify the story ID"}
  ],
  "allow_multiple": false
}])
```

**NEVER:**
- ❌ Automatically select a story without asking
- ❌ Pick based on priority, complexity, or migration order without user confirmation
- ❌ Proceed to translation without explicit user selection

**ALWAYS:**
- ✅ Use AskQuestion to present story options
- ✅ Show story ID, title, complexity, priority
- ✅ Let user decide which story to translate
- ✅ Respect user's choice even if it differs from migration-order.md

## What You Need

| Input | Required | Description |
|-------|----------|-------------|
| **Spec folder location** | Yes | Folder containing extracted requirements (from extraction phase) |
| **User story to translate** | Yes | Selected via AskQuestion (see above) |
| **Target folder** | Yes | Where to write Spring Boot 4 application |
| **Migration order** | Yes | Read `spec/migration-order.md` for recommended sequence |

## Translation Workflow

```
┌─────────────────────────────────────┐
│ User Story (from spec/)             │
│ (As a... I want... So that...)      │
├─────────────────────────────────────┤
│ Technical Context (YAML)            │
│ (Schema entities from extraction)   │
├─────────────────────────────────────┤
│ Generated Spring Boot 4 Code        │
│ - Controller/Service/Repository     │
│ - Entity/DTO classes                │
│ - Configuration                     │
├─────────────────────────────────────┤
│ Tests (from user stories)           │
│ - Generated from acceptance         │
│   criteria and test cases (TC-*)    │
│ - Unit tests (JUnit 6)              │
│ - Integration tests                 │
├─────────────────────────────────────┤
│ Cloud Foundry Config                │
│ - application.yml                   │
│ - manifest.yml                      │
└─────────────────────────────────────┘
```

**Tests are required**: Every user story must have corresponding generated tests (from AC and TC-* table).

## Spring Boot 4 Prerequisites

| Requirement | Version | Notes |
|-------------|---------|-------|
| Java | 17 minimum, 25 recommended | Virtual threads with Java 21+ |
| Jakarta EE | 11 | `jakarta.*` packages |
| Spring Framework | 7.0 | Foundation for Spring Boot 4 |
| Spring Security | 7.0 | Component-based configuration |
| Hibernate | 7.1 | JPA 3.2 alignment |
| Jackson | 3.x | `tools.jackson` package |
| JUnit | 6 | Updated annotations and lifecycle |

**Modular starters**: Spring Boot 4 uses modular dependencies. Include only what you need.

For complete POM configuration, see [modules/translation-spring-boot-4-prerequisites.md](modules/translation-spring-boot-4-prerequisites.md).

## Schema Entity Mapping

Map Technical Context Schema entities to Spring Boot 4 components:

| Schema Entity | Spring Boot 4 Target | Key Annotations/Classes |
|---------------|---------------------|------------------------|
| `ServiceEndpoint` | REST Controller | `@RestController`, `@GetMapping`, `@PostMapping` |
| `ServiceOperation` | Service Layer | `@Service`, `@Transactional` |
| `DataStructure` (entity) | JPA Entity | `@Entity`, `@Table`, `@Id` |
| `DataStructure` (dto) | Record/DTO | Java Records with JSpecify |
| `ViewRequirement` | Thymeleaf/REST | `@Controller` + templates or `@RestController` |
| `FormDefinition` | Form Handling | `@ModelAttribute`, `@Valid`, Thymeleaf forms |
| `MessagingRequirement` | JMS | `JmsClient`, `@JmsListener` |
| `SecurityRequirement` | Spring Security 7 | `@PreAuthorize`, `SecurityFilterChain` |
| `TransactionRequirement` | Transaction Mgmt | `@Transactional`, `@EnableTransactionManagement` |
| `ServiceTask` | Scheduling | `@Scheduled`, `@EnableScheduling` |
| `ServiceBatchProcess` | Spring Batch 6 | `Job`, `Step`, `ItemReader`, `ItemWriter` |
| `ExternalResourceRequirement` | CF Bindings | `application.yml`, environment variables |
| `IntegrationPoint` | HTTP Client | `@HttpExchange`, `RestClient` |
| `LdapRequirement` | Spring LDAP 4 + Spring Security 7 LDAP | `LdapTemplate`, `LdapContextSource`, `LdapBindAuthenticationManagerFactory` |
| `JNDIDatasource` | Datasource + pool | `spring.datasource.*`, `spring.datasource.hikari.*`. **Recompute the pool size per instance.** |
| `MailRequirement` | Mail sender | `spring.mail.*`, `JavaMailSender` |
| `SessionManagementRequirement` | Spring Session | `spring.session.store-type`, `spring-session-data-redis` |
| `MonitoringRequirement` | Actuator + Micrometer | `MeterRegistry`, `management.*`, per-instance tags |
| `MultiTenancyRequirement` | Routing datasource | `AbstractRoutingDataSource` |
| `ServerMigrationRequirement` | Removals | Delete vendor descriptors and vendor packages; `server.servlet.context-path` |

## Context Management (Load Only What You Need)

To avoid context limits, load only modules needed for the current story:

### Default Load (load for every story)
- This SKILL.md (core translation guide)
- [modules/translation-spring-boot-4-testing.md](modules/translation-spring-boot-4-testing.md) — **ALWAYS load** (Spring Boot 4 test patterns differ significantly from Boot 3: `@WebMvcTest` removed, `@MockitoBean` package changed)
- [../tanzu-extract-story-detail/technical-context-schema.md](../tanzu-extract-story-detail/technical-context-schema.md) (or only sections for entity types in story)

### Load Modules Based on Story Content

| Story Has | Load Module |
|-----------|-------------|
| REST/JSON endpoints | [modules/translation-spring-boot-4-service-endpoint.md](modules/translation-spring-boot-4-service-endpoint.md) |
| Business logic operations | [modules/translation-spring-boot-4-service-operation.md](modules/translation-spring-boot-4-service-operation.md) |
| Entities or DTOs | [modules/translation-spring-boot-4-data-structure.md](modules/translation-spring-boot-4-data-structure.md) |
| VIEW endpoints or forms | [modules/translation-spring-boot-4-ui.md](modules/translation-spring-boot-4-ui.md) |
| JMS/messaging | [modules/translation-spring-boot-4-messaging.md](modules/translation-spring-boot-4-messaging.md) |
| JNDI datasources or multi-tenant routing (`jndi_datasources` or `multi_tenancy_requirement` present) | [modules/translation-spring-boot-4-jndi-datasource.md](modules/translation-spring-boot-4-jndi-datasource.md) |
| Transactions, and any XA/JTA pattern (`transaction_requirement` present) | [modules/translation-spring-boot-4-transactions.md](modules/translation-spring-boot-4-transactions.md) |
| JNDI mail sessions (`mail_requirements` present) | [modules/translation-spring-boot-4-mail.md](modules/translation-spring-boot-4-mail.md) |
| Session persistence or clustering (`session_management_requirement` present) | [modules/translation-spring-boot-4-session.md](modules/translation-spring-boot-4-session.md) |
| JMX, vendor management APIs, or vendor system properties (`monitoring_requirements` present) | [modules/translation-spring-boot-4-monitoring.md](modules/translation-spring-boot-4-monitoring.md) |
| Auth/authz requirements, including application-server realms (`SecurityRequirement.realm_type` set) | [modules/translation-spring-boot-4-security.md](modules/translation-spring-boot-4-security.md) |
| LDAP integration or LDAP authentication (`ldap_requirements` present, or `authentication_method: LDAP`) | [modules/translation-spring-boot-4-ldap.md](modules/translation-spring-boot-4-ldap.md) |
| EJB remote interfaces (`migration_considerations` has `"Remote EJB Interface"` / `"EJB 2.x Remote Interface"` / `"EJB 2.x Remote Home"`, or `ServiceOperation.remote_interface: true`, or `IntegrationPoint type: RMI_EJB`) | [modules/translation-spring-boot-4-ejb-remote.md](modules/translation-spring-boot-4-ejb-remote.md) |
| External APIs/resources | [modules/translation-spring-boot-4-integration.md](modules/translation-spring-boot-4-integration.md) |
| Scheduled tasks | [modules/translation-spring-boot-4-scheduled-task.md](modules/translation-spring-boot-4-scheduled-task.md) |
| Batch jobs | [modules/translation-spring-boot-4-batch.md](modules/translation-spring-boot-4-batch.md) |
| CLI commands | [modules/translation-spring-boot-4-cli.md](modules/translation-spring-boot-4-cli.md) |
| Caching (`CacheRequirement` present) | [modules/translation-spring-boot-4-cache.md](modules/translation-spring-boot-4-cache.md) |
| Integration flows, connectors, DataWeave transforms or flow error handlers (`integration_flow_requirements`, `connector_requirements`, `transformation_requirements` or `error_handling_requirements` present) | [modules/translation-spring-boot-4-integration-flow.md](modules/translation-spring-boot-4-integration-flow.md) |
| Tests to generate | (always loaded — see Default Load above) |
| CF deployment config | [modules/translation-spring-boot-4-cloud-foundry.md](modules/translation-spring-boot-4-cloud-foundry.md) |
| POM generation/validation | [modules/translation-spring-boot-4-prerequisites.md](modules/translation-spring-boot-4-prerequisites.md) |

### Load When Spec Has Migration Considerations

If `spec/{story}/technical-context.yaml` has non-empty `migration_considerations` array, load:
- [modules/translation-instructions-spring-boot-4-migration-considerations.md](modules/translation-instructions-spring-boot-4-migration-considerations.md)

Topics include: Session State, Caching, Server Migration, UI Migration, Distributed Transactions, Vendor
Monitoring, LDAP Events, JNDI Enumeration.

## Translation Steps (Per Story)

### 1. Bootstrap project when target is empty

**When**: The target folder has no `pom.xml` and no `build.gradle` (first story) or the user explicitly asks to (re)initialize the project.

**Build type**: Ask the user for Maven or Gradle via AskQuestion when the target is empty, unless they have already specified a preference (e.g. "use Gradle"). Default to Maven if not specified.

**Steps**:
1. **Collect dependency IDs** from the spec: `ModernizationRequirements.spring_boot_recommendations.starters`, per-entity `starter_dependency`, and schema-driven inference (REST → web, JPA → data-jpa, SecurityRequirement → security, Thymeleaf → thymeleaf, JMS → artemis, CLI → shell, observability → actuator). When the story involves persistence (e.g. JPA entities, Spring Data JPA, or repository/integration tests), include the Initializr ID `testcontainers` in the comma-separated dependencies list. Map each starter to start.spring.io dependency IDs using the table in [modules/translation-spring-boot-4-prerequisites.md](modules/translation-spring-boot-4-prerequisites.md).
2. **Build the curl command** for `https://start.spring.io/starter.zip` with: `type=maven-project` or `type=gradle-project`, `language=java`, `bootVersion=4.0.2`, `packaging=jar`, `javaVersion=17` (or from `spring_boot_recommendations.java_version` if present), and `groupId`, `artifactId`, `name`, `description`, `packageName` derived from the spec (e.g. `application_name` → artifactId/name; default groupId `com.example`, packageName `com.example.<app>`).
3. **Add** `dependencies=<comma-separated Initializr IDs>` (no spaces).
4. **Run** `curl -G ... -o app.zip`, then unzip into the target folder so the project root (where `pom.xml` or `build.gradle` lives) is the target folder. If the zip contains a single root directory, unzip so its contents become the target folder contents.

For the starter → Initializr ID mapping and exact curl examples, see [modules/translation-spring-boot-4-prerequisites.md](modules/translation-spring-boot-4-prerequisites.md). Add dependencies not on start.spring.io (e.g. JSpecify) to the build file after unzip if required.

### 2. User Story Selection
Use AskQuestion to present story options (see protocol above).

### 3. Load Story Files
From spec folder, load:
- `spec/domains/{domain}/US-{NNN}-{title}.md` (selected story)
- `spec/domains/{domain}/technical-context.md` (domain context)
- Only required entity definitions from `spec/data-model/entities.md`

### 4. Load Translation Modules
Based on story content, load only required modules (see Context Management above).

### 5. Generate Code

**Controller** (`@RestController` or `@Controller`):
- Map from `ServiceEndpoint` in technical context
- API versioning when specified
- Authentication/authorization annotations

**Service** (`@Service`):
- Map from `ServiceOperation` in technical context
- `@Transactional` where required
- Preconditions → validation logic
- Steps → business logic
- Structural events → persistence operations

**Repository** (Spring Data JPA):
- Interface extending `JpaRepository<Entity, ID>`
- Custom queries from ServiceOperation queries

**Entity/DTO** (JPA Entity or Record):
- Map from `DataStructure` in technical context
- JPA annotations (`@Entity`, `@Table`, `@Column`)
- Relationships (`@ManyToOne`, `@OneToMany`, etc.)
- Validation constraints
- Java Records for DTOs with JSpecify

**Configuration**:
- Security: `SecurityFilterChain`, `@PreAuthorize`
- HTTP Client: `RestClient`, `@HttpExchange`
- JMS: `JmsClient`, `@JmsListener`
- Scheduling: `@EnableScheduling`
- Batch: Job/Step configuration

### 6. Generate Tests (Required)

Generate **JUnit 6** tests from:
- Acceptance criteria (AC-*)
- Test case table (TC-*) - ALL categories:
  - Happy path
  - Validation errors
  - Authorization (403)
  - Not found (404)
  - Business rule violations
  - Edge cases
  - UI tests (if VIEW endpoint)

**Test types**:
- **Unit tests**: `@ExtendWith(MockitoExtension.class)` for service logic
- **Integration tests**: `@SpringBootTest` with RestTestClient or MockMvc for endpoints
- **Testcontainers**: For database-dependent tests

See [modules/translation-spring-boot-4-testing.md](modules/translation-spring-boot-4-testing.md) for patterns.

### 7. Generate Cloud Foundry Config

Always generate both files:
- `application.yml`: Actuator, server config. Add datasource (HikariCP), JMS, etc. when `ExternalResourceRequirement` entries exist in the technical context.
- `manifest.yml`: Service bindings, env vars, memory, instances

**`ddl-auto` + `spring.sql.init` rule:** When the spec contains a `SeedData` block, OR any TC-* precondition requires specific rows to exist at runtime, you MUST apply ALL of the following in `application.yml`:
1. Set `spring.jpa.hibernate.ddl-auto: none` (never `create-drop` or `create` — those cause Hibernate to re-create tables **after** `spring.sql.init` runs, wiping seed data)
2. Set `spring.sql.init.schema-locations: classpath:db/schema.sql` (or equivalent)
3. Set `spring.sql.init.data-locations: classpath:db/data.sql` (or equivalent)
4. Set `spring.sql.init.mode: always`

See [modules/translation-spring-boot-4-cloud-foundry.md](modules/translation-spring-boot-4-cloud-foundry.md).

## ServiceOperation Translation Pattern

`ServiceOperation` is central to business logic translation:

### Steps → Business Logic
- Use `$variable` format: "Get Entity as $var (by input)"
- Get steps introduce variables
- Later steps reference variables
- For QUERY operations: one step per output field
  - `"Set output.field = $source to load the data"`

### Preconditions → Validation
- Map to validation guards/validators
- Throw specified exception on failure
- Do NOT repeat in steps

### Structural Events → Persistence
- `ENTITY_CREATED`/`ENTITY_UPDATED`/`ENTITY_DELETED` → repository saves/deletes
- `instance_variable` → entity instance
- `value_sources` and `attributes` → map inputs to entity fields
- `MESSAGE_SENT` → message publishing

For complete patterns and examples, load [modules/translation-spring-boot-4-service-operation.md](modules/translation-spring-boot-4-service-operation.md).

## Output Checklist

- ✅ **Bootstrap project via start.spring.io when target is empty** (Maven or Gradle). See step 1 and [modules/translation-spring-boot-4-prerequisites.md](modules/translation-spring-boot-4-prerequisites.md).

For each user story, generate:

- ✅ Controller (`@RestController` or `@Controller`)
- ✅ Service (`@Service` with `@Transactional`)
- ✅ Repository (Spring Data JPA interface)
- ✅ Entity/DTO (JPA Entity or Record)
- ✅ Configuration (Security, HTTP client, JMS, scheduling, batch as needed)
- ✅ Tests (JUnit 6 unit and integration tests from AC and TC-*)
- ✅ Cloud Foundry config (`application.yml`, `manifest.yml`)
- ✅ If spec has `SeedData` or TC-* rows require pre-existing rows: `ddl-auto: none` and `spring.sql.init` configured with schema + data locations

## Module Index

All translation modules are in the `modules/` directory:

| Module | Schema Entity / Use Case |
|--------|--------------------------|
| [translation-spring-boot-4-service-endpoint.md](modules/translation-spring-boot-4-service-endpoint.md) | ServiceEndpoint, REST controllers |
| [translation-spring-boot-4-service-operation.md](modules/translation-spring-boot-4-service-operation.md) | ServiceOperation, business logic |
| [translation-spring-boot-4-data-structure.md](modules/translation-spring-boot-4-data-structure.md) | DataStructure, entities, DTOs |
| [translation-spring-boot-4-ui.md](modules/translation-spring-boot-4-ui.md) | ViewRequirement, FormDefinition, Thymeleaf |
| [translation-spring-boot-4-messaging.md](modules/translation-spring-boot-4-messaging.md) | MessagingRequirement, JMS |
| [translation-spring-boot-4-jndi-datasource.md](modules/translation-spring-boot-4-jndi-datasource.md) | JNDIDatasource, MultiTenancyRequirement, HikariCP per-instance pool sizing |
| [translation-spring-boot-4-transactions.md](modules/translation-spring-boot-4-transactions.md) | TransactionRequirement, `@Transactional`, XA → Transactional Outbox / Saga |
| [translation-spring-boot-4-mail.md](modules/translation-spring-boot-4-mail.md) | MailRequirement, `spring.mail.*`, JavaMailSender |
| [translation-spring-boot-4-session.md](modules/translation-spring-boot-4-session.md) | SessionManagementRequirement, Spring Session + Redis |
| [translation-spring-boot-4-monitoring.md](modules/translation-spring-boot-4-monitoring.md) | MonitoringRequirement, Actuator + Micrometer, per-instance tags |
| [translation-spring-boot-4-security.md](modules/translation-spring-boot-4-security.md) | SecurityRequirement, auth/authz, application-server realms |
| [translation-spring-boot-4-ldap.md](modules/translation-spring-boot-4-ldap.md) | LdapRequirement, LDAP auth, Spring LDAP 4 |
| [translation-spring-boot-4-ejb-remote.md](modules/translation-spring-boot-4-ejb-remote.md) | EJB remote interfaces (`@Remote`, `EJBObject`, `EJBHome`) → REST / Service |
| [translation-spring-boot-4-integration.md](modules/translation-spring-boot-4-integration.md) | IntegrationPoint, ExternalResource, HTTP clients |
| [translation-spring-boot-4-scheduled-task.md](modules/translation-spring-boot-4-scheduled-task.md) | ServiceTask, @Scheduled |
| [translation-spring-boot-4-batch.md](modules/translation-spring-boot-4-batch.md) | ServiceBatchProcess, Spring Batch 6 |
| [translation-spring-boot-4-cache.md](modules/translation-spring-boot-4-cache.md) | CacheRequirement, Redis distributed caching |
| [translation-spring-boot-4-integration-flow.md](modules/translation-spring-boot-4-integration-flow.md) | IntegrationFlowRequirement, ConnectorRequirement, TransformationRequirement, ErrorHandlingRequirement (MuleSoft) |
| [translation-spring-boot-4-cli.md](modules/translation-spring-boot-4-cli.md) | ServiceCommand, CLI commands |
| [translation-spring-boot-4-testing.md](modules/translation-spring-boot-4-testing.md) | JUnit 6 test generation — **always load** |
| [translation-spring-boot-4-cloud-foundry.md](modules/translation-spring-boot-4-cloud-foundry.md) | Cloud Foundry deployment |
| [translation-spring-boot-4-prerequisites.md](modules/translation-spring-boot-4-prerequisites.md) | POM, dependencies, modular starters |
| [translation-instructions-spring-boot-4-migration-considerations.md](modules/translation-instructions-spring-boot-4-migration-considerations.md) | `migration_considerations` router: Session State, Caching, Server Migration, UI Migration, Distributed Transactions, Vendor Monitoring, LDAP Events, JNDI Enumeration |

## Examples

Complete end-to-end examples are available:

- [modules/translation-spring-boot-4-example-rest-crud.md](modules/translation-spring-boot-4-example-rest-crud.md) - REST CRUD operations
- [modules/translation-spring-boot-4-example-ui.md](modules/translation-spring-boot-4-example-ui.md) - Thymeleaf UI

Load these when you need complete reference examples.

## Technical Context Schema Reference

The schema is owned by the `tanzu-extract-story-detail` skill, which `cf repo unpack-skills` always
installs beside this skill. Read it from the sibling folder:
[../tanzu-extract-story-detail/technical-context-schema.md](../tanzu-extract-story-detail/technical-context-schema.md).

Key entities:
- `ServiceEndpoint` - REST/web endpoints with paths, methods, auth
- `ServiceCommand` - CLI commands with options
- `ServiceOperation` - Business logic with steps, events, transactions
- `DataStructure` - Entities, DTOs with fields and relationships
- `IntegrationPoint` - External systems (APIs, databases, queues)
- `SecurityRequirement` - Authentication and authorization
- `MessagingRequirement` - Async messaging
- `ServiceBatchProcess` - Batch jobs with steps
- `ServiceTask` - Scheduled tasks with cron expressions

## Quality Guidelines

### Code Generation
- Use Jakarta EE 11 packages (`jakarta.*`)
- Use Spring Boot 4 modular starters
- Generate Java Records for DTOs (with JSpecify)
- Include JPA validation constraints
- Proper exception handling
- Transaction boundaries (@Transactional)

### Test Generation
- Generate tests for ALL TC-* table entries
- Cover happy path AND corner cases (validation, auth, not found, business rules, edge)
- Use JUnit 6 annotations
- Use Testcontainers for persistence tests
- Follow Given-When-Then pattern from spec

### Configuration
- Externalize configuration to application.yml
- Use Cloud Foundry service bindings
- Enable actuator endpoints (health, info, metrics)
- Configure HikariCP for datasources

## Migration from Spring Boot 2.x/3.x

When source is Spring Boot 2.x or 3.x:

1. **Javax → Jakarta**: Update all package imports
2. **Modular starters**: Split spring-boot-starter-web into webmvc, etc.
3. **Spring Security 7**: Use component-based SecurityFilterChain
4. **JUnit 6**: Update test annotations and lifecycle
5. **Jackson 3.x**: Update to tools.jackson package
6. **Preserve behavior**: Keep test scenarios equivalent

See modules for detailed migration patterns.

## One Story Per Context

**Translation efficiency**: Translate one user story per context window (new session).

For each story:
1. Open new context/session
2. Use AskQuestion to select story
3. Load only that story's spec files
4. Load only required translation modules
5. Generate code and tests
6. Write to target folder (merge with existing project)

Next story: Repeat in new context.

## Summary

1. **Story selection required**: Use AskQuestion before every translation
2. **Requirements-driven**: Work from schema entities in technical context
3. **Context efficiency**: Load only modules needed for current story
4. **Tests required**: Generate JUnit 6 tests from AC and TC-*
5. **Spring Boot 4**: Jakarta EE 11, modular starters, Spring Security 7
6. **Cloud Foundry**: Generate application.yml and manifest.yml
7. **One story per context**: New session for each story

## Apply mandatory platform optimizations

After code generation, `cd` into the target folder and run the following commands automatically (do not ask the user; continue even if a command fails):

```bash
cf repo apply-advice -n tanzu-spring-starter
```

Use the application name from `manifest.yml` (the `name` attribute, matching the `artifactId` chosen in step 1):

```bash
cf repo apply-advice --name=spring-aot --avoid-interactive --inputs=manifest.yml,<application-name>
```

Then add the git-info plugin for the project's build tool (determined in step 1):

- Maven: `cf repo apply-advice --name=add-git-plugin-maven`
- Gradle: `cf repo apply-advice --name=add-git-plugin-gradle`
