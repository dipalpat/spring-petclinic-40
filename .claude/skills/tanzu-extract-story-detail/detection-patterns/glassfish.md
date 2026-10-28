# GlassFish Detection Patterns

Patterns for GlassFish, Eclipse GlassFish, Payara, and the older Sun Java System Application Server. There
is no GlassFish buildpack on Tanzu Platform, so every application in this family must move to Spring Boot
on Apache Tomcat.

Read this file when [app-servers.md](app-servers.md) identifies GlassFish. Populate the entities in
[technical-context-schema.md](../technical-context-schema.md). Record detected facts only — the Spring Boot
and Cloud Foundry mapping is the translation phase's job.

## Table of Contents

- [Server Identification](#server-identification)
- [Category 1: Deployment Descriptors and Datasources](#category-1-deployment-descriptors-and-datasources)
- [Category 2: JNDI Resource Lookups](#category-2-jndi-resource-lookups)
- [Category 3: Session Management and Clustering](#category-3-session-management-and-clustering)
- [Category 4: Monitoring](#category-4-monitoring)
- [Category 5: Security Realms and Authentication](#category-5-security-realms-and-authentication)
- [Corner Cases](#corner-cases)

---

## Server Identification

| Pattern | File Types | Indicates | Schema Entity |
|---------|------------|-----------|---------------|
| `glassfish-web.xml`, `glassfishweb.xml`, `glassfish-application.xml`, `glassfish-resources.xml`, `glassfish-ejb-jar.xml` | filename | GlassFish 3+ deployment | `ModernizationRequirements.source_server = GLASSFISH` |
| `sun-web.xml`, `sun-application.xml`, `sun-resources.xml`, `sun-ejb-jar.xml` | filename | Sun Application Server / GlassFish 2 deployment. Same patterns, older element names. | `ModernizationRequirements.source_server = GLASSFISH` |
| `payara-resources.xml`, `<payara-` elements | filename, `*.xml` | Payara deployment | `ModernizationRequirements.source_server = GLASSFISH` |
| `domain.xml` | filename | GlassFish domain configuration. Holds realms and server-wide resources that the application depends on but does not ship. | `ServerMigrationRequirement.deployment_descriptors` |
| `org\.glassfish\.`, `com\.sun\.enterprise\.`, `com\.sun\.appserv\.` | `*.java` | Vendor API dependency in application code | `ServerMigrationRequirement.vendor_dependencies` |

**Notes:**

- File names vary by version. Search for all spellings; do not assume one.
- Both spellings can coexist in one application after an in-place upgrade. Record every file found in
  `ServerMigrationRequirement.deployment_descriptors` with `action: REMOVE`.
- `<context-root>` in `glassfish-web.xml` maps to `server.servlet.context-path`.

---

## Category 1: Deployment Descriptors and Datasources

Resources are declared in two places. `web.xml` declares the *reference* (`<resource-ref>`), and
`glassfish-resources.xml` (or `domain.xml`) declares the *implementation* (`<jdbc-connection-pool>` plus
`<jdbc-resource>`). Read both. The reference alone gives the JNDI name; only the pool gives the sizing.

### 1a — JNDI datasource with a vendor pool

| Pattern | `<jdbc-connection-pool`, `<jdbc-resource jndi-name=`, `<resource-ref>` + `<res-ref-name>jdbc/` |
|---------|--------------------------------------------------------------------------------------------------|
| **File Types** | `glassfish-resources.xml`, `sun-resources.xml`, `domain.xml`, `web.xml`, `glassfish-web.xml` |
| **Indicates** | A container-managed datasource that the application reaches through JNDI |
| **Schema Entity** | `JNDIDatasource` |

**Example:**

```xml
<!-- glassfish-resources.xml -->
<jdbc-connection-pool name="OrdersDBPool"
                      datasource-classname="com.mysql.cj.jdbc.MysqlDataSource"
                      res-type="javax.sql.DataSource"
                      max-pool-size="20"
                      min-pool-size="5"
                      idle-timeout-in-seconds="900"
                      max-wait-time-in-millis="6000"
                      connection-validation-method="custom-validation"
                      validation-classname="com.mysql.cj.jdbc.Driver"
                      fail-all-connections="false">
    <property name="url" value="jdbc:mysql://db.example.com:3306/orders"/>
</jdbc-connection-pool>

<jdbc-resource jndi-name="jdbc/OrdersDB" pool-name="OrdersDBPool"/>
```

```xml
<!-- web.xml -->
<resource-ref>
    <res-ref-name>jdbc/OrdersDB</res-ref-name>
    <res-type>javax.sql.DataSource</res-type>
    <res-auth>Container</res-auth>
</resource-ref>
```

**Extraction Notes:**

- Join the two files on `jndi-name` / `<res-ref-name>`. One `JNDIDatasource` per joined pair.
- `<jdbc-resource jndi-name>` → `jndi_name`. `<res-ref-name>` → `resource_ref_name`. A code lookup usually
  prefixes it with `java:comp/env/`; record the form the code uses in `jndi_name`.
- `<jdbc-connection-pool name>` → `vendor_pool_name`. `res-type` → `datasource_type`.
- Pool attributes → `pool_config`: `max-pool-size` → `max_pool_size`, `min-pool-size` → `min_pool_size`,
  `idle-timeout-in-seconds` → `idle_timeout_seconds`, `max-wait-time-in-millis` → `max_wait_time_millis`,
  `connection-validation-method` → `connection_validation_method`, `validation-table-name` or
  `validation-classname` → `validation_table`.
- Record `max_pool_size` unchanged. Compute `spring_boot_migration.hikari_maximum_pool_size` per instance —
  see the pool sizing rule in `JNDIDatasource`.
- `fail-all-connections="true"` has no HikariCP equivalent. Add a remediation step.
- Do **not** also emit an `ExternalResourceRequirement` with `type: DATASOURCE` for the same resource.

### 1b — Several datasources (multi-tenancy)

| Pattern | More than one `<jdbc-resource>` with distinct `jndi-name` values, or a class that selects a datasource at runtime |
|---------|--------------------------------------------------------------------------------------------------------------------|
| **File Types** | `glassfish-resources.xml`, `web.xml`, `*.java` |
| **Indicates** | Multi-tenant or split-database routing |
| **Schema Entity** | one `JNDIDatasource` per tenant, plus one `MultiTenancyRequirement` |

**Extraction Notes:**

- Emit one `JNDIDatasource` per datasource, each with its own `pool_config` and a `multi_tenancy_context`
  naming the tenant.
- Emit `MultiTenancyRequirement` only when the code chooses between the datasources at runtime. Several
  unrelated datasources (for example an orders database and an audit database) are not multi-tenancy.
- Find the routing class from a lookup whose JNDI name is built from a request value, a thread-local, or a
  security principal. Record it in `legacy_router_class` and set `tenant_identifier_source` accordingly.
- Sum the pools. The connection budget is `sum(max_pool_size) × instance_count`, not one pool.

### 1c — Heavily tuned pool

| Pattern | `max-pool-size` above roughly 30, or `statement-cache-size`, `pool-resize-quantity`, `steady-pool-size` |
|---------|-----------------------------------------------------------------------------------------------------------|
| **File Types** | `glassfish-resources.xml`, `domain.xml` |
| **Indicates** | A pool tuned for one always-on server |
| **Schema Entity** | `JNDIDatasource.pool_config` + `migration_effort: MEDIUM` |

**Extraction Notes:**

- The vendor number is a per-server number. It becomes a per-instance number on Cloud Foundry, so the same
  value multiplies by the instance count. Record the vendor number, and add a remediation step that asks for
  the target instance count and the database `max_connections` limit.
- `steady-pool-size` is the GlassFish name for the minimum. Map it to `min_pool_size`.

### 1d — XA / JTA datasource

| Pattern | `res-type="javax.sql.XADataSource"`, `transaction-support="XATransaction"`, `UserTransaction`, `javax.transaction.` / `jakarta.transaction.` imports |
|---------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| **File Types** | `glassfish-resources.xml`, `domain.xml`, `*.java` |
| **Indicates** | Distributed (two-phase commit) transactions |
| **Schema Entity** | `TransactionRequirement` + `JNDIDatasource.datasource_type` |

**Example:**

```xml
<jdbc-connection-pool name="XADataSourcePool"
                      res-type="javax.sql.XADataSource"
                      datasource-classname="com.mysql.cj.jdbc.MysqlXADataSource"
                      max-pool-size="20"/>
<jdbc-resource jndi-name="jdbc/XADataSource" pool-name="XADataSourcePool"/>
```

**Extraction Notes:**

- Set `TransactionRequirement.xa_datasource_detected: true`, `distributed_transactions: true`,
  `transaction_manager_type: JTA`, `legacy_datasource_jndi` to the XA JNDI name, and
  `server_type: GLASSFISH`.
- Set `scope` from the code. `GLOBAL` when most flows enlist more than one resource; `LOCAL` when one or two
  components do. When the code does not show which, set `REQUIRES_USER_FEEDBACK` and ask the user.
- Add `"Distributed Transactions"` to `ModernizationRequirements.migration_considerations`.
- A pool declared `XADataSource` but never enlisted in a `UserTransaction` is XA-capable, not XA-using.
  Record it, and say so in `legacy_config`.

---

## Category 2: JNDI Resource Lookups

### 2a — Programmatic datasource lookup

| Pattern | `new InitialContext()`, `.lookup("java:comp/env/jdbc/`, `.lookup("jdbc/`, `.lookup("pool/` |
|---------|----------------------------------------------------------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | A connection acquired outside any dependency-injection container |
| **Schema Entity** | `JNDIDatasource.lookup_pattern` |

**Example:**

```java
InitialContext ctx = new InitialContext();
DataSource ds = (DataSource) ctx.lookup("java:comp/env/jdbc/OrdersDB");
```

**Extraction Notes:**

- Record the literal lookup string in `lookup_pattern`. Match it to the `JNDIDatasource` from Category 1 by
  the JNDI name; do not create a second entity.
- A lookup with no matching declaration means the resource lives only in `domain.xml`, which is outside the
  repository. Record the JNDI name, leave the pool fields absent, and raise a strategic question.
- Also run [jndi-ldap.md](jndi-ldap.md) — it covers the general `InitialContext` and LDAP patterns.

### 2b — Mail session lookup

| Pattern | `<mail-resource jndi-name=`, `.lookup("mail/`, `javax.mail.Session`, `jakarta.mail.Session` |
|---------|----------------------------------------------------------------------------------------------|
| **File Types** | `glassfish-resources.xml`, `web.xml`, `*.java` |
| **Indicates** | A container-managed mail session |
| **Schema Entity** | `MailRequirement` |

**Example:**

```xml
<mail-resource jndi-name="mail/GlassfishMailSession"
               transport-protocol="smtp"
               host="smtp.example.com"
               port="587"
               user="notifications@example.com"
               from="notifications@example.com">
    <property name="mail.smtp.auth" value="true"/>
    <property name="mail.smtp.starttls.enable" value="true"/>
</mail-resource>
```

**Extraction Notes:**

- `host` → `properties.smtp_host`, `port` → `properties.smtp_port`, `mail.smtp.auth` →
  `properties.smtp_auth`, `mail.smtp.starttls.enable` → `properties.starttls_enabled`.
- Do **not** also emit an `ExternalResourceRequirement` with `type: MAIL_SESSION`.
- `store-protocol` (IMAP or POP3) means the application reads mail as well as sends it. Spring Boot's
  `JavaMailSender` sends only. Record the store protocol in a remediation step.

### 2c — JMS connection factory and destination

| Pattern | `<connector-resource jndi-name="jms/`, `<connector-connection-pool`, `<admin-object-resource res-type="javax.jms.Queue"`, `.lookup("jms/`, `javax.jms.` / `jakarta.jms.` imports |
|---------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **File Types** | `glassfish-resources.xml`, `web.xml`, `*.java` |
| **Indicates** | JMS messaging through the container |
| **Schema Entity** | `MessagingRequirement` |

**Example:**

```xml
<connector-resource jndi-name="jms/GlassfishConnectionFactory"
                    pool-name="jms-connection-factory-pool"/>
<connector-connection-pool name="jms-connection-factory-pool"
                           resource-adapter-name="jmsra"
                           connection-definition-name="javax.jms.ConnectionFactory"
                           transaction-support="XATransaction"
                           max-pool-size="32"/>
<admin-object-resource jndi-name="jms/OrderQueue"
                       res-type="javax.jms.Queue"
                       res-adapter-name="jmsra">
    <property name="PhysicalName" value="OrderQueue"/>
</admin-object-resource>
```

**Extraction Notes:**

- `<connector-resource jndi-name>` → `legacy_config.connection_factory_jndi`.
  `<admin-object-resource jndi-name>` → `legacy_config.destination_jndi`.
- `res-type="javax.jms.Queue"` → `destination_type: QUEUE` and `pattern: POINT_TO_POINT`.
  `javax.jms.Topic` → `destination_type: TOPIC` and `pattern: PUBLISH_SUBSCRIBE`.
- `PhysicalName` → `destination_name`. Fall back to the last segment of the JNDI name.
- Set `direction` from the code: a `MessageConsumer` or `@MessageDriven` class is `CONSUMER`, a
  `MessageProducer` is `PRODUCER`.
- `transaction-support="XATransaction"` on the connector pool is an XA marker. Handle it as in 1d.

---

## Category 3: Session Management and Clustering

### 3a — Session replication

| Pattern | `<session-config>` with `<persistence-type>replicated</persistence-type>`, `replicationScope`, `persistenceFrequency` |
|---------|-------------------------------------------------------------------------------------------------------------------------|
| **File Types** | `glassfish-web.xml`, `glassfishweb.xml`, `sun-web.xml`, `domain.xml` |
| **Indicates** | A clustered deployment that replicates HTTP sessions between server instances |
| **Schema Entity** | `SessionManagementRequirement` |

**Example:**

```xml
<session-config>
    <persistence-type>replicated</persistence-type>
    <session-manager>
        <manager-properties>
            <property name="replicationScope" value="instance"/>
            <property name="persistenceFrequency" value="web-method"/>
        </manager-properties>
    </session-manager>
</session-config>
```

**Extraction Notes:**

- `persistence_type: REPLICATED`, `clustering_enabled: true`, `replication_scope` from `replicationScope`,
  `persistence_frequency` from `persistenceFrequency`,
  `replication_mechanism: "GlassFish HA session replication"`.
- Set `cloud_foundry_compatible: false`. The platform has no equivalent of server-to-server replication.
- Set `spring_boot_store_type: redis`.
- Add `"Session State"` to `ModernizationRequirements.migration_considerations`.
- Also record what the application puts in the session (see [spring-web.md](spring-web.md) and
  [ejb.md](ejb.md)). The store choice is only half of the migration; the objects must be serializable.

### 3b — In-memory sessions with a file store

| Pattern | `<persistence-type>memory</persistence-type>` with `enableFileStore`, `sessionFileName`, or a `<store-properties>` `directory` |
|---------|---------------------------------------------------------------------------------------------------------------------------------|
| **File Types** | `glassfish-web.xml`, `sun-web.xml` |
| **Indicates** | Sessions held in one instance's memory and written to that instance's local disk |
| **Schema Entity** | `SessionManagementRequirement` |

**Extraction Notes:**

- `persistence_type: MEMORY_WITH_FILESTORE`, `file_based_persistence: true`,
  `cloud_foundry_compatible: false`.
- Record the configured path in a remediation step. A Cloud Foundry instance's local disk does not survive a
  restage, a crash restart, or a scale-down, so the path itself cannot be carried over.
- Also check for direct file writes in code. A file session store often comes with other local-disk
  assumptions.

### 3c — Session timeout and cookie flags

| Pattern | `<session-timeout>`, `<cookie-properties>` with `secure`, `httpOnly`, `domain`, `path` |
|---------|-----------------------------------------------------------------------------------------|
| **File Types** | `glassfish-web.xml`, `sun-web.xml`, `web.xml` |
| **Indicates** | Per-module session and cookie policy |
| **Schema Entity** | `SessionManagementRequirement` |

**Example:**

```xml
<session-config>
    <session-timeout>30</session-timeout>
    <cookie-properties>
        <property name="secure" value="true"/>
        <property name="httpOnly" value="true"/>
    </cookie-properties>
</session-config>
```

**Extraction Notes:**

- `<session-timeout>` is in minutes in both `web.xml` and `glassfish-web.xml` → `session_timeout_minutes`.
- `secure` → `cookie_secure`, `httpOnly` → `cookie_http_only`.
- A vendor `cookie-properties` `domain` or `path` value is deployment configuration, not application
  configuration. Record it as a remediation step, not as a Spring property.
- 3a, 3b and 3c usually appear inside one `<session-config>` block. Emit one
  `SessionManagementRequirement`, not three.

---

## Category 4: Monitoring

### 4a — Vendor admin API

| Pattern | `GlassfishRuntime`, `GlassFishRuntime`, `AdminCommandContext`, `com\.sun\.enterprise\.`, `org\.glassfish\.embeddable\.` |
|---------|--------------------------------------------------------------------------------------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | Application code that drives the application server itself |
| **Schema Entity** | `MonitoringRequirement` with `metrics_source: VENDOR_API`, plus `ServerMigrationRequirement.vendor_dependencies` |

**Extraction Notes:**

- Set `management_interface_present: true` and `requires_centralized_backend: true`.
- Record the class in `configuration_source` and the package in `vendor_dependencies`.
- This code has no portable equivalent. It is removed, not translated. Set `migration_effort: MEDIUM` and
  add `"Vendor Monitoring"` to `migration_considerations`.

### 4b — JMX MBeans

| Pattern | `com.sun.appserv:`, `amx:`, `ObjectName(`, `MBeanServerConnection`, `ManagementFactory.getPlatformMBeanServer`, `*MBean` interfaces |
|---------|-------------------------------------------------------------------------------------------------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | Metrics read from, or published to, the server's JMX registry |
| **Schema Entity** | `MonitoringRequirement` with `metrics_source: JMX` |

**Example:**

```java
MBeanServer mbs = ManagementFactory.getPlatformMBeanServer();
ObjectName objName = new ObjectName("com.sun.appserv:type=server,name=server");
```

**Extraction Notes:**

- Record the literal `ObjectName` string in `mbean_pattern`.
- `com.sun.appserv:*` is the Sun Application Server 8/9 naming. GlassFish 3+ and Payara use `amx:*`. Search
  for both.
- Distinguish two cases and record which one applies in `migration_notes`:
  - The application **reads** server MBeans → the data is server state. Platform metrics replace it, not
    application code.
  - The application **registers** its own MBean → the data is application state. A Micrometer meter
    replaces it.
- Set `per_instance_tagging: true` and `requires_centralized_backend: true`. An MBean describes one
  instance, and that instance is replaced without warning.

### 4c — Vendor system properties

| Pattern | `System.getProperty("glassfish.`, `System.getProperty("com.sun.aas.`, `System.getenv("AS_`, `${com.sun.aas.instanceRoot}` |
|---------|----------------------------------------------------------------------------------------------------------------------------|
| **File Types** | `*.java`, `*.xml`, `*.properties` |
| **Indicates** | Code coupled to the server installation |
| **Schema Entity** | `MonitoringRequirement` with `metrics_source: VENDOR_PROPERTIES` |

**Extraction Notes:**

- Record each property name in `metrics_tags` or `migration_notes`.
- `com.sun.aas.instanceRoot` and `com.sun.aas.installRoot` are filesystem paths into the server
  installation. There is no equivalent. Raise a strategic question for each one.
- `migration_effort: LOW` when the property is only logged. `MEDIUM` or higher when a path or a decision
  depends on it.

---

## Category 5: Security Realms and Authentication

A GlassFish realm lives in `domain.xml`, outside the application. The application ships only the role
mapping. Expect the realm definition to be absent from the repository; record what the application declares
and raise a strategic question for the rest.

### 5a — File realm with role mapping

| Pattern | `<security-role-mapping>` with `<role-name>`, `<principal-name>`, `<group-name>`; `classname=".*realm\.file\.FileRealm"` |
|---------|---------------------------------------------------------------------------------------------------------------------------|
| **File Types** | `glassfish-web.xml`, `glassfish-application.xml`, `sun-web.xml`, `domain.xml` |
| **Indicates** | Users and groups held inside the application server |
| **Schema Entity** | `SecurityRequirement` |

**Example:**

```xml
<security-role-mapping>
    <role-name>admin</role-name>
    <principal-name>adminUser</principal-name>
    <group-name>admins</group-name>
</security-role-mapping>
```

**Extraction Notes:**

- `authentication_method: FILE_REALM`, `realm_type: FileRealm`, `role_mapping_present: true`,
  `server_type: GLASSFISH`.
- One `legacy_config.role_mappings` entry per `<security-role-mapping>` block.
- Read the `<security-constraint>` and `<login-config>` blocks in `web.xml` for the URL rules and the
  authentication method (`FORM`, `BASIC`, `CLIENT-CERT`), and put them in `protected_endpoints`.
- The user list is not in the repository. Add a remediation step: the target needs a real user store.

### 5b — LDAP realm

| Pattern | `classname=".*realm\.ldap\.LDAPRealm"`, `<property name="directory"`, `base-dn`, `search-filter`, `group-search-filter`, `group-base-dn`, `search-bind-dn` |
|---------|--------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **File Types** | `domain.xml`, `glassfish-web.xml`, `login.conf` |
| **Indicates** | Authentication against a corporate directory |
| **Schema Entity** | `LdapRequirement` + `SecurityRequirement` |

**Example:**

```xml
<auth-realm name="ldap-realm" classname="com.sun.enterprise.security.auth.realm.ldap.LDAPRealm">
    <property name="directory" value="ldap://ldap.example.com:389"/>
    <property name="base-dn" value="dc=example,dc=com"/>
    <property name="search-bind-dn" value="cn=admin,dc=example,dc=com"/>
    <property name="search-filter" value="(&amp;(objectClass=person)(uid=%s))"/>
    <property name="group-base-dn" value="ou=groups,dc=example,dc=com"/>
    <property name="group-search-filter" value="member=%s"/>
</auth-realm>
```

**Extraction Notes:**

- `SecurityRequirement`: `authentication_method: LDAP`, `realm_type: LDAPRealm`.
- `LdapRequirement`: `directory` → `provider_url`, `base-dn` → `search_base`, `search-bind-dn` → `bind_dn`,
  `search-filter` → `search_filter`, `group-base-dn` → `group_search_base`, `group-search-filter` →
  `group_search_filter`, `authentication_use: BIND`.
- GlassFish filters use `%s` as the user placeholder. Spring Security uses `{0}`. Record the GlassFish form
  as detected; the translation converts it.
- `search-bind-password` in a descriptor is a hardcoded credential. Set `credentials_source: HARDCODED` and
  flag it.
- An `ldaps://` URL sets `tls_mode: LDAPS`. A plain `ldap://` URL with no TLS property sets
  `tls_mode: NONE`.
- See [jndi-ldap.md](jndi-ldap.md) for directory access from code, which is a separate concern from realm
  authentication.

### 5c — Certificate realm

| Pattern | `classname=".*realm\.certificate\.CertificateRealm"`, `<auth-method>CLIENT-CERT</auth-method>`, `assign-groups`, `jaas-context` |
|---------|----------------------------------------------------------------------------------------------------------------------------------|
| **File Types** | `domain.xml`, `web.xml`, `glassfish-web.xml` |
| **Indicates** | Client-certificate (mutual TLS) authentication |
| **Schema Entity** | `SecurityRequirement` |

**Extraction Notes:**

- `authentication_method: CERTIFICATE`, `realm_type: CertificateRealm`, `mtls_required: true`.
- `assign-groups` lists the roles every valid certificate holder receives. Record it in
  `legacy_config.realm_properties` and in `role_mappings`.
- The trust store lives in the server installation. Record that it must be supplied to the application, and
  raise a strategic question about where the certificate reaches the application when a platform router
  terminates TLS.

---

## Corner Cases

Each case sets an action. `ERROR` means the configuration is ambiguous or incomplete and a person must
decide. `WARN` means the configuration is valid but couples the application to the server.

| # | Detected | Action | Record it as |
|---|----------|--------|--------------|
| 1 | `<persistence-type>ha</persistence-type>` | WARN — deprecated in GlassFish 5+ | `SessionManagementRequirement.persistence_type: HA_DEPRECATED`, plus a `conflicts` entry. Map to `redis`. |
| 2 | More than one `<session-config>`, or two different `<persistence-type>` values | ERROR — ambiguous | `SessionManagementRequirement.conflicts`, one entry per declaration. Raise a strategic question; do not pick one silently. |
| 3 | `<resource-ref>` with no matching `<jdbc-resource>` or `<mail-resource>` | ERROR — incomplete | A `JNDIDatasource` (or `MailRequirement`) with `jndi_name` set and the pool fields absent. Raise a strategic question. |
| 4 | `System.getProperty(...)` supplying a JNDI name or a URL, e.g. `ctx.lookup(System.getProperty("jdbc.datasource.name", "pool/Default"))` | WARN — environment coupling | The default value in `JNDIDatasource.jndi_name`, the property name in `lookup_pattern`, and a remediation step to externalize it |
| 5 | `ctx.rebind(...)` or `ctx.bind(...)` at runtime, often in `@PostConstruct` | WARN — the resource does not exist until the code runs | A `JNDIDatasource` with `lookup_pattern` naming the rebind, and a remediation step. There is no descriptor to read, so the pool fields stay absent. |

**Extraction Notes:**

- A corner case never blocks extraction. Record what is there, mark the gap, and continue.
- Cases 3, 4 and 5 all mean the JNDI name alone is known. Do not invent pool sizes for them.
