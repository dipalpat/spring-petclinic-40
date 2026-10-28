# Pattern Usage Guidelines

This document provides guidelines for using the detection patterns effectively, including the recommended workflow, pattern priority, handling ambiguity, and expected output format.

## Table of Contents

- [Detection Workflow](#detection-workflow)
- [Pattern Priority](#pattern-priority)
- [Handling Ambiguity](#handling-ambiguity)
- [Output Format](#output-format)

---

## Detection Workflow

Follow this workflow when using detection patterns to analyze a legacy application:

1. **File Discovery**: Scan the project for relevant file types
   - Java files (`*.java`)
   - XML configuration files (`*.xml`)
   - View templates (`*.jsp`, `*.xhtml`, `*.html`)
   - COBOL sources (`*.cbl`, `*.cob`, `*.cobol`, `*.cpy`)
   - JCL files (`*.jcl`, `*.job`)
   - Config files (`application.properties`, `application.yml`)

2. **Technology Detection**: Run technology-specific patterns to identify source framework
   - **Check for Spring Boot first** (see [spring-boot.md](spring-boot.md)). If the app is Spring Boot 2.x/3.x/4.x (detected via code and config only; do not use `pom.xml` or `build.gradle`), use Spring Boot patterns for extraction and record Boot version. Translation will produce a Boot 4 app that should support the same tests.
   - Check for Spring Web patterns (see [spring-web.md](spring-web.md))
   - Check for Picocli CLI patterns (see [picocli.md](picocli.md))
   - Check for EJB patterns (see [ejb.md](ejb.md))
   - Check for COBOL patterns (see [cobol.md](cobol.md))
   - Check for UI layer patterns (see [ui-layer.md](ui-layer.md))
   - **When EJB or any Java EE / Spring Web app is detected**, also run [jndi-ldap.md](jndi-ldap.md) to detect programmatic JNDI lookups (`InitialContext`) and direct LDAP access (`InitialLdapContext`, `DirContext`, LDAP controls, event listeners)

3. **Server Detection**: Run server-specific patterns to identify deployment target
   - Run [app-servers.md](app-servers.md) first. It identifies the server and routes to the per-vendor file.
   - **When GlassFish, Payara, or Sun Application Server is identified, run [glassfish.md](glassfish.md)**
     for datasources and vendor pools, JNDI mail and JMS resources, session replication, JMX and admin API
     monitoring, and security realms
   - Record the result in `ModernizationRequirements.source_server` and in the `server_type` field of every
     entity that has one

4. **Entity Extraction**: Run detailed patterns to extract schema entities
   - When Spring Boot, Spring Web, or EJB is detected, **always** run the [JPA persistence patterns](jpa-persistence.md) to extract `DataStructure` (entity) fields and relationships from `@Entity`, `@Table`, `@Column`, and relationship annotations. This applies to both `javax.persistence` and `jakarta.persistence`.
   - Extract `ServiceEndpoint` from controller patterns
   - Extract `ServiceOperation` from service/EJB patterns
   - Extract `DataStructure` from entity/model patterns
   - Extract `JNDIDatasource` from JNDI datasource declarations and lookups, and `MailRequirement` from
     JNDI mail sessions. Use `ExternalResourceRequirement` only for a resource that neither entity covers.
   - Extract `SessionManagementRequirement` from server session configuration,
     `MonitoringRequirement` from JMX / vendor API / vendor property usage, and
     `MultiTenancyRequirement` when the code routes between tenant datasources at runtime
   - Extract `ViewComponent` and `FormDefinition` from UI patterns

5. **Cross-Reference**: Link extracted entities to form complete requirements
   - For every endpoint that stores or retrieves data, ensure a corresponding ServiceOperation exists and is linked; ensure each such operation has inputs and output (or view model for VIEW) specified
   - Connect operations to data structures
   - Connect UI components to their backing controllers
   - Build navigation flows from form actions and forwards

---

## Pattern Priority

When multiple patterns match, use this priority order:

1. **Annotations** (most specific, modern approach)
   - `@Controller`, `@Service`, `@Entity`, `@Stateless`, etc.
   - These provide the most accurate and intentional declarations

2. **XML Configuration** (explicit configuration)
   - Spring XML beans, EJB deployment descriptors, Struts config
   - Explicit configuration that may override or supplement annotations

3. **Code Patterns** (inferred from code structure)
   - Class hierarchies, method signatures, naming conventions
   - Less reliable but useful for legacy code without annotations

4. **File Names** (deployment descriptors, config files)
   - `web.xml`, `jboss-web.xml`, `faces-config.xml`, etc.
   - Indicates technology presence but may need deeper analysis

---

## Handling Ambiguity

### Hybrid Applications

If both Spring and EJB patterns match, the application may be hybrid:
- Document both technologies in `ModernizationRequirements`
- EJB components may be wrapped by Spring services
- Consider the migration path for each component type separately

### Spring Boot Applications

When Spring Boot patterns match (see [spring-boot.md](spring-boot.md)):
- Treat the app as a Spring Boot 2.x/3.x/4.x application. Use Spring Boot patterns for extraction; map to the same Technical Context Schema.
- If both Spring Boot and Spring Web patterns match, treat as Spring Boot (Boot takes precedence for entry points and config).
- Record Boot version when inferable from code (e.g. `javax.*` → 2.x, `jakarta.*` → 3.x/4.x) so translation can target Boot 4 correctly.

### Multiple Server Indicators

If multiple servers are detected:
- Prioritize based on deployment descriptor presence
- Check which server-specific JNDI patterns are actually used in code
- The most recently used or primary deployment target takes precedence
- Record the descriptors of the other server in `ServerMigrationRequirement.deployment_descriptors`. Two
  descriptor families often coexist after an in-place server upgrade (for example `weblogic.xml` beside
  `jboss-web.xml`), and both must be removed.

### COBOL Applications

For COBOL applications:
- Rely heavily on copybooks (`.cpy`) for structure understanding
- JCL files provide batch job flow information
- Standalone COBOL programs with FILE SECTION and sequential file I/O (and no CICS/BMS) are batch programs; use PROGRAM-ID for the job name and derive input/output from SELECT ASSIGN
- BMS maps define screen/UI interactions
- CICS commands indicate transaction boundaries

#### Fixed-Width File Column Ranges (mandatory verification)

After computing column ranges from PIC declarations, **always** verify them against a physical sample file before publishing ranges in the spec. Wrong ranges cause `IncorrectLineLengthException` at runtime.

1. Run `awk '{print length, $0}' input/file.dat | head -5` to get the actual line length.
2. Compute the declaration-derived total using physical character widths (see [cobol.md](cobol.md) FILE SECTION notes — `PIC 9(m)V9(n)` on `LINE SEQUENTIAL` = `m + 1 + n` chars, not `m + n`).
3. If declaration-derived total ≠ measured length, use the **measured** length and document the discrepancy in a strategic question.
4. Publish column ranges in the spec only after this check. Record them in `input_sources.format` as: `"FIXED_WIDTH N chars/record (field1 cols 1-A, field2 cols B-C, ...)"`.
5. Cross-check: if the spec contains test data examples (`valid_input` records), ensure each example record's character count matches the declared column ranges. A mismatch is a signal to re-examine the physical format.

### Missing Information

When patterns don't provide complete information:
- Mark fields as `TO_BE_DETERMINED` in the schema
- Document assumptions made during extraction
- Flag for manual review in the migration notes

---

## Output Format

When using these patterns for extraction, output should conform to the Technical Context Schema format defined in [technical-context-schema.md](../technical-context-schema.md).

### Example Extraction Output

```yaml
ModernizationRequirements:
  application_name: "order-management"
  source_technology: SPRING_WEB
  source_server: JBOSS
  target_platform: "Spring Boot 3.x on Cloud Foundry"
  
  service_endpoints:
    - path: "/orders"
      method: GET
      authentication: AUTHENTICATED
      response_type: VIEW
      
    - path: "/orders/{id}"
      method: GET
      path_variables:
        - name: id
          type: Long
      authentication: AUTHENTICATED
      response_type: VIEW
      
  service_operations:
    - name: "findAllOrders"
      type: QUERY
      transactional: true
      read_only: true
      
    - name: "cancelOrder"
      type: COMMAND
      transactional: true
      steps:
        - "Get Order as $order (by orderId)"
        - "Update $order status to CANCELLED"
        - "Send cancellation notification"
      structural_events:
        - kind: ENTITY_UPDATED
          entity: Order
          instance_variable: "$order"
          description: "Order status set to CANCELLED"
        - kind: MESSAGE_SENT
          description: "Cancellation notification sent"
          
  external_resources:
    - name: "ordersDataSource"
      type: DATASOURCE
      legacy_jndi_name: "java:jboss/datasources/OrdersDS"
      
  view_requirement:
    view_technology: JSP
    layout_framework: TILES
    migration_target: THYMELEAF
    
  # ... additional entities follow schema structure
```

### Validation Checklist

After extraction, verify:
- [ ] All detected technologies are documented
- [ ] Service endpoints that store or retrieve data have corresponding service operations; each such operation has inputs and output (or view structure for VIEW) specified
- [ ] Every endpoint with `response_type: VIEW` has `response_body` specified (view model / structure passed to the view)
- [ ] Steps use variables in `$name` format; Get steps state the criteria (e.g. "Get Order as $order (by orderId)"); `structural_events` describe what changes (entity/link create/update/delete, message sent) with `instance_variable`, `entity`, `relationship`, `value_source`, and `description` as applicable
- [ ] Steps do not contain UI-related actions (e.g. "Return for view", "Redirect to view"); only data operations (Get, Add, Update, Return, Send)
- [ ] When a DTO has fields whose type is an entity (or collection of entity), either `loaded_shape` is populated for those fields or the field type is a view DTO that fully defines the shape
- [ ] CLI commands (ServiceCommand entities) reference valid ServiceOperation; option names map to operation inputs
- [ ] Data structures are linked to persistence configuration
- [ ] External resources have migration recommendations
- [ ] Every JNDI-bound datasource yields a `JNDIDatasource` (not an `ExternalResourceRequirement` with `type: DATASOURCE`), and every JNDI mail session yields a `MailRequirement` (not `type: MAIL_SESSION`). No resource appears in both forms.
- [ ] Every `JNDIDatasource` with a vendor `max_pool_size` also has a per-instance `spring_boot_migration.hikari_maximum_pool_size`, or a remediation step asking for the instance count and the database connection limit
- [ ] When a `javax.sql.XADataSource` or `UserTransaction` is detected, a `TransactionRequirement` is populated with `xa_datasource_detected: true` and a `scope`, and `"Distributed Transactions"` is added to `migration_considerations`
- [ ] When server session configuration is detected, a `SessionManagementRequirement` is populated with `cloud_foundry_compatible` set, and `"Session State"` is added to `migration_considerations`
- [ ] File-based session persistence sets `file_based_persistence: true` and `cloud_foundry_compatible: false`
- [ ] When JMX, a vendor management API, or vendor system properties are detected, a `MonitoringRequirement` is populated with `per_instance_tagging: true` and `requires_centralized_backend: true`
- [ ] When an application-server realm is detected, `SecurityRequirement.realm_type` and `server_type` are set, and an LDAP realm also yields an `LdapRequirement`
- [ ] When LDAP classes are detected (`InitialLdapContext`, `DirContext`, LDAP controls), a `LdapRequirement` entry is populated and `ldap_requirements` is present in the output
- [ ] When `NamingEvent`, `NamingExceptionEvent`, or `UnsolicitedNotificationEvent` are detected, `"LDAP Events"` is added to `migration_considerations`
- [ ] When `NamingEnumeration` / `NameClassPair` JNDI tree enumeration is detected, `"JNDI Enumeration"` is added to `migration_considerations`
- [ ] UI components have migration targets specified
- [ ] Security requirements are captured
- [ ] Transaction boundaries are identified
