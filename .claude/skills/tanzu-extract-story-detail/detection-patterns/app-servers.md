# Application Server Detection Patterns

This document identifies which application server a legacy Java EE application was designed for. Run it
first. When it names a server that has its own pattern file, read that file next for the detailed patterns.

## Which File To Read Next

| Server identified | Read next |
|-------------------|-----------|
| GlassFish, Payara, Sun Application Server | [glassfish.md](glassfish.md) — datasources and vendor pools, JNDI mail and JMS resources, session replication, JMX and admin API, realms |
| JBoss, WildFly | this file (`#jboss--wildfly`) |
| WebLogic | this file (`#weblogic`) |
| WebSphere, WebSphere Liberty | this file (`#websphere`) |
| Tomcat, TomEE | this file (`#tomcat`) |
| MuleSoft Mule Runtime | [mulesoft.md](mulesoft.md) — HTTP flows, connectors, DataWeave transformations, error handling |
| None of the above | this file (`#generic-java-ee`), and set `source_server: GENERIC` |

A story that adds a per-vendor pattern file adds its row here.

Record the identified server in `ModernizationRequirements.source_server` and in the `server_type` field of
every entity that has one. The enum is in
[technical-context-schema.md](../technical-context-schema.md#server-types).

## Table of Contents

- [Which File To Read Next](#which-file-to-read-next)
- [GlassFish](#glassfish)
- [JBoss / WildFly](#jboss--wildfly)
- [WebLogic](#weblogic)
- [WebSphere](#websphere)
- [Tomcat](#tomcat)
- [Generic Java EE](#generic-java-ee)
- [Datasource connection pool extraction](#datasource-connection-pool-extraction)

---

## GlassFish

### GlassFish Deployment Descriptors

| Pattern | `glassfish-web.xml`, `glassfishweb.xml`, `glassfish-application.xml`, `glassfish-resources.xml`, `sun-web.xml`, `sun-resources.xml` (filenames) |
|---------|--------------------------------------------------------------------------------------------------------------------------------------------------|
| **File Types** | filename |
| **Indicates** | GlassFish, Eclipse GlassFish, Payara, or Sun Application Server deployment |
| **Schema Entity** | `ServerMigrationRequirement.source_server = GLASSFISH` |

### GlassFish Package Imports

| Pattern | `org\.glassfish\.`, `com\.sun\.enterprise\.`, `com\.sun\.appserv\.` |
|---------|------------------------------------------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | GlassFish-specific code dependency |
| **Schema Entity** | `ServerMigrationRequirement.vendor_dependencies` |

**There is no GlassFish buildpack on Tanzu Platform.** The application must move to Spring Boot on Apache
Tomcat. See [glassfish.md](glassfish.md) for the full patterns — datasources and vendor pools (`JNDIDatasource`),
mail sessions (`MailRequirement`), JMS resources (`MessagingRequirement`), session replication
(`SessionManagementRequirement`), JMX and the admin API (`MonitoringRequirement`), and realms
(`SecurityRequirement` / `LdapRequirement`).

---

## JBoss / WildFly

### Server Identification

| Pattern | File Types | Indicates | Schema Entity |
|---------|------------|-----------|---------------|
| `jboss-web.xml`, `jboss-app.xml`, `jboss-ejb3.xml`, `jboss-deployment-structure.xml` | filename | JBoss / WildFly deployment descriptor | `ModernizationRequirements.source_server = JBOSS` |
| `standalone.xml`, `standalone-ha.xml`, `domain.xml`, `*-ds.xml` | filename | JBoss / WildFly server/subsystem configuration | `ServerMigrationRequirement.deployment_descriptors` |
| `java:jboss/` | `*.xml`, `*.java` | JBoss-proprietary JNDI namespace | `JNDIDatasource.jndi_name` / `legacy_jndi_name` |
| `org\.jboss\.`, `org\.wildfly\.` | `*.java` | JBoss / WildFly vendor API dependency | `ServerMigrationRequirement.vendor_dependencies` |

### Category 6: Datasources & JNDI

#### 6a — Standard Datasource with Pool Tuning

| Pattern | `<datasource jndi-name="java:jboss/datasources/`, `<pool>`, `<min-pool-size>`, `<max-pool-size>` |
|---------|--------------------------------------------------------------------------------------------------|
| **File Types** | `*-ds.xml`, `standalone.xml`, `standalone-ha.xml`, `domain.xml` |
| **Indicates** | Container-managed datasource with JBoss pool configuration |
| **Schema Entity** | `JNDIDatasource` |

**Example:**
```xml
<datasource jndi-name="java:jboss/datasources/OrdersDS" pool-name="OrdersPool">
  <connection-url>jdbc:mysql://db.example.com:3306/orders</connection-url>
  <driver-name>mysql</driver-name>
  <security>
    <user-name>appuser</user-name>
    <password>apppass</password>
  </security>
  <pool>
    <min-pool-size>5</min-pool-size>
    <max-pool-size>20</max-pool-size>
    <prefill>true</prefill>
    <use-ccm>true</use-ccm>
    <flush-strategy>EntirePool</flush-strategy>
  </pool>
</datasource>
```

**DSL Schema Mapping:**
```yaml
JNDIDatasource:
  name: "ordersDataSource"
  jndi_name: "java:jboss/datasources/OrdersDS"
  server_type: JBOSS
  vendor_pool_name: "OrdersPool"
  pool_config:
    max_pool_size: 20
    min_pool_size: 5
    prefill: true
    use_ccm: true
  spring_boot_migration:
    datasource_url: "jdbc:mysql://db.example.com:3306/orders"
    hikari_maximum_pool_size: 20
    hikari_minimum_idle: 5
    hikari_idle_timeout_ms: 900000
    hikari_connection_timeout_ms: 30000
```

**Migration Effort:** LOW

**Scaling Note:** `maximum-pool-size: 20` is per-instance. With 4 CF instances, total = 80 connections against DB limit. Sizing formula: `floor((db_max_connections × 0.8) / instance_count)`.

---

#### 6b — XA Datasource (Distributed Transactions)

| Pattern | `<xa-datasource>`, `<xa-datasource-property>` |
|---------|-----------------------------------------------|
| **File Types** | `*-ds.xml`, `standalone.xml`, `standalone-ha.xml`, `domain.xml` |
| **Indicates** | XA-capable distributed transaction datasource |
| **Schema Entity** | `JNDIDatasource` + `TransactionRequirement` |

**DSL Schema Mapping:**
```yaml
JNDIDatasource:
  name: "transactionalOrdersDataSource"
  jndi_name: "java:jboss/datasources/TransactionalOrdersXA"
  server_type: JBOSS
  datasource_type: "javax.sql.XADataSource"

TransactionRequirement:
  transaction_manager: JTA
  supports_distributed_transactions: true
  xa_datasource_detected: true
  legacy_datasource_jndi: "java:jboss/datasources/TransactionalOrdersXA"
  scope: REQUIRES_USER_FEEDBACK
```

WARNING: XA/2PC is incompatible with Cloud Foundry ephemeral instances. Flag for manual architecture review.
Spring Boot alternatives: Transactional Outbox pattern or Saga pattern with compensating actions.

**Migration Effort:** HIGH

---

#### 6c — Portable vs. Vendor-Locked JNDI Namespaces

| Pattern | `java:app/`, `java:module/`, `java:global/` (portable EE7+) vs `java:jboss/` (vendor-locked) |
|---------|----------------------------------------------------------------------------------------------|
| **File Types** | `*.java`, `*.xml` |
| **Indicates** | Resource injection namespace portability |
| **Schema Entity** | `JNDIDatasource` / Spring dependency injection |

**Spring Boot 4 Mapping:** Replace JNDI `@Resource(lookup = "...")` lookups with Spring component injection (`@Autowired private DataSource dataSource;`).

**Migration Effort:** LOW

---

#### 6d — Multiple Datasources (Multi-Tenancy)

| Pattern | Multiple `<datasource>` entries across EAR/WAR modules |
|---------|--------------------------------------------------------|
| **File Types** | `*-ds.xml`, `standalone.xml`, `jboss-web.xml` |
| **Indicates** | Multi-datasource or multi-tenant routing |
| **Schema Entity** | `MultiTenancyRequirement` + multiple `JNDIDatasource` |

**DSL Schema Mapping:**
```yaml
MultiTenancyRequirement:
  pattern: DATASOURCE_ROUTING
  datasource_routing_pattern: PER_TENANT
  module_count: 3

JNDIDatasource:
  name: "ordersDataSource"
  jndi_name: "java:jboss/datasources/OrdersDS"
  server_type: JBOSS

JNDIDatasource:
  name: "billingDataSource"
  jndi_name: "java:jboss/datasources/BillingDS"
  server_type: JBOSS

JNDIDatasource:
  name: "auditDataSource"
  jndi_name: "java:jboss/datasources/AuditDS"
  server_type: JBOSS
```

**Migration Effort:** MEDIUM

**Scaling Note:** Each CF instance opens independent pools for each datasource. Total connections = `(pool_orders + pool_billing + pool_audit) × instance_count`.

---

### Category 7: JNDI Resources (Mail, Messaging, JCA)

#### 7a — Mail Session

| Pattern | `<mail-session jndi-name="java:jboss/mail/`, `<smtp-server>` |
|---------|---------------------------------------------------------------|
| **File Types** | `standalone.xml`, `domain.xml` |
| **Indicates** | Container-managed JavaMail session |
| **Schema Entity** | `MailRequirement` |

**DSL Schema Mapping:**
```yaml
MailRequirement:
  name: "defaultMailSession"
  legacy_jndi_name: "java:jboss/mail/Default"
  server_type: JBOSS
  properties:
    smtp_host: "smtp.company.com"
    smtp_port: 587
    smtp_auth: true
    starttls_enabled: true
    from_address: "noreply@company.com"
  spring_boot_alternative: "spring.mail.* + JavaMailSender"
```

**Migration Effort:** LOW

---

#### 7b — Artemis / JBoss Messaging (JMS)

| Pattern | `<jms-queue>`, `<jms-topic>`, `<connection-factory>` in messaging subsystem |
|---------|-----------------------------------------------------------------------------|
| **File Types** | `standalone.xml`, `standalone-ha.xml`, `domain.xml`, `jboss-ejb3.xml` |
| **Indicates** | JBoss HornetQ/Artemis messaging destinations and connection factories |
| **Schema Entity** | `MessagingRequirement` |

**DSL Schema Mapping:**
```yaml
MessagingRequirement:
  name: "orderQueueMessaging"
  pattern: POINT_TO_POINT
  direction: CONSUMER
  destination_type: QUEUE
  destination_name: "OrderQueue"
  server_type: JBOSS
  legacy_config:
    connection_factory_jndi: "java:/ConnectionFactory"
    destination_jndi: "java:jboss/jms/queue/OrderQueue"
  spring_boot_recommendation: "Spring Cloud Stream with RabbitMQ/ActiveMQ binder or @JmsListener"
```

**Migration Effort:** MEDIUM

---

### Category 8: Clustering & Session Replication

#### 8a — Distributed Session Replication (Infinispan)

| Pattern | `<distributable/>` in `web.xml`, Infinispan `<distributed-cache>` in `standalone-ha.xml`, `org.infinispan:infinispan-spring-boot-starter` |
|---------|---------------------------------------------------------------------------------------------------------------------------------------------|
| **File Types** | `web.xml`, `standalone-ha.xml`, `pom.xml`, `build.gradle` |
| **Indicates** | Clustered session replication via Infinispan cache container |
| **Schema Entity** | `SessionManagementRequirement` |

**DSL Schema Mapping:**
```yaml
SessionManagementRequirement:
  persistence_type: REPLICATED
  server_type: JBOSS
  clustering_enabled: true
  replication_mechanism: INFINISPAN
  cloud_foundry_compatible: false
  spring_boot_alternative: "redis"
```

**Migration Effort:** MEDIUM

**Scaling Note:** Infinispan assumes direct node-to-node multicast/discovery. Cloud Foundry has no such cluster fabric. Externalize sessions to Spring Session + Redis.

---

#### 8b — HA Singleton Deployments

| Pattern | `<singleton>` deployment descriptor, `@Singleton` + `@Schedule`, `SingletonService`, HA-singleton Infinispan config |
|---------|-----------------------------------------------------------------------------------------------------------------------|
| **File Types** | `*.xml`, `*.java` |
| **Indicates** | Single-active-instance cluster singleton pattern |
| **Schema Entity** | `ServiceTask` / architecture review flag |

WARNING: HA Singleton assumes exactly one active node cluster-wide. Cloud Foundry is ephemeral and stateless. NOT CF COMPATIBLE.
Spring Boot alternatives: Distributed Lock (Redis/ShedLock) or dedicated single-instance worker app (`instances: 1`).

**Migration Effort:** HIGH (manual architecture review required)

---

### Category 9: Monitoring

#### 9a — JBoss Management API

| Pattern | `ModelControllerClient`, `jboss.as:` resource paths |
|---------|-----------------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | Programmatic JBoss management operations |
| **Schema Entity** | `MonitoringRequirement` |

**Spring Boot 4 Replacement:** Spring Boot Actuator + Micrometer.

**Migration Effort:** MEDIUM

---

#### 9b — JBoss JMX Monitoring

| Pattern | `org.jboss:*`, `jboss.as:subsystem=*`, `jboss.as:deployment=*` ObjectNames |
|---------|----------------------------------------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | JMX MBean monitoring of JBoss subsystems |
| **Schema Entity** | `MonitoringRequirement` |

**DSL Schema Mapping:**
```yaml
MonitoringRequirement:
  metrics_source: JMX
  server_type: JBOSS
  mbean_pattern: "jboss.as:subsystem=datasources,*"
  spring_boot_alternative: "micrometer-prometheus"
  prometheus_export: true
  per_instance_tagging: true
  requires_centralized_backend: true
```

**Migration Effort:** LOW

---

### Category 10: Security

#### 10a — Legacy JAAS Security Domain

| Pattern | `<security-domain>` in `jboss-web.xml`, `UsersRolesLoginModule`, `PicketBox` |
|---------|-----------------------------------------------------------------------------|
| **File Types** | `jboss-web.xml`, `standalone.xml` |
| **Indicates** | Legacy JAAS/PicketBox security domain authentication |
| **Schema Entity** | `SecurityRequirement` |

**DSL Schema Mapping:**
```yaml
SecurityRequirement:
  authentication_method: JAAS_LOGIN_MODULE
  realm_type: "PicketBox"
  server_type: JBOSS
```

**Migration Effort:** MEDIUM

---

#### 10b — Elytron Security (Modern WildFly)

| Pattern | `<subsystem xmlns="urn:wildfly:elytron">`, `<filesystem-realm>`, `<properties-realm>` |
|---------|--------------------------------------------------------------------------------------|
| **File Types** | `standalone.xml`, `standalone-ha.xml` |
| **Indicates** | Modern WildFly Elytron security subsystem |
| **Schema Entity** | `SecurityRequirement` |

**Migration Effort:** LOW (Elytron concepts map cleanly to Spring Security)

---

#### 10c — LDAP Realm

| Pattern | `<ldap-realm>`, `<dir-context url="ldap://` |
|---------|---------------------------------------------|
| **File Types** | `standalone.xml`, `domain.xml` |
| **Indicates** | JBoss/WildFly LDAP authentication |
| **Schema Entity** | `LdapRequirement` + `SecurityRequirement` |

**DSL Schema Mapping:**
```yaml
LdapRequirement:
  name: "corporateLdap"
  provider_url: "ldap://ldap.example.com:389"
  search_base: "dc=example,dc=com"
  bind_dn: "cn=admin,dc=example,dc=com"
  search_filter: "uid={0}"
  group_search_base: "ou=groups,dc=example,dc=com"
  server_type: JBOSS

SecurityRequirement:
  authentication_method: LDAP
  realm_type: LDAP
  server_type: JBOSS
```

**Migration Effort:** HIGH

---

### JBoss / WildFly Corner Cases

1. **PicketBox + Elytron coexisting**: Flag and recommend consolidating to Elytron / Spring Security.
2. **Conflicting datasource `jndi-name` in multiple `*-ds.xml` files**: Incomplete configuration; flag for manual resolution.
3. **`<resource-ref>` in `jboss-web.xml` without matching bound datasource**: Orphaned configuration.
4. **System-property-driven overrides** (`${jboss.server.data.dir}`): Replace with `@Value` and `application.yml`.
5. **Programmatic JNDI rebind at runtime**: Anti-pattern; refactor to Spring `@Bean`.

---

## WebLogic

### Server Identification

| Pattern | File Types | Indicates | Schema Entity |
|---------|------------|-----------|---------------|
| `weblogic.xml`, `weblogic-application.xml`, `weblogic-application-ext.xml`, `weblogic-ejb-jar.xml` | filename | WebLogic deployment descriptors | `ModernizationRequirements.source_server = WEBLOGIC` |
| `config.xml` (WebLogic domain config) | filename | WebLogic domain configuration | `ServerMigrationRequirement.deployment_descriptors` |
| `weblogic\.`, `com\.bea\.`, `com\.oracle\.weblogic\.` | `*.java` | WebLogic vendor API dependency | `ServerMigrationRequirement.vendor_dependencies` |

### Category 16: WebLogic Datasources & JNDI

#### 16a — Standard Datasource with Connection Pool

| Pattern | `<jdbc-data-source>`, `<jndi-name>jdbc/`, `<max-capacity>`, `<min-capacity>` |
|---------|-----------------------------------------------------------------------------|
| **File Types** | `*-jdbc.xml`, `config.xml`, `weblogic.xml` |
| **Indicates** | WebLogic container-managed datasource with connection pool configuration |
| **Schema Entity** | `JNDIDatasource` |

**Example (`*-jdbc.xml`):**
```xml
<jdbc-data-source xmlns="http://xmlns.oracle.com/weblogic/jdbc-data-source">
  <name>OrdersDataSource</name>
  <jdbc-driver-params>
    <url>jdbc:mysql://db.example.com:3306/orders</url>
    <driver-name>com.mysql.cj.jdbc.Driver</driver-name>
    <properties>
      <property><name>user</name><value>appuser</value></property>
    </properties>
    <password-encrypted>apppass</password-encrypted>
  </jdbc-driver-params>
  <jdbc-connection-pool-params>
    <initial-capacity>5</initial-capacity>
    <max-capacity>20</max-capacity>
    <min-capacity>5</min-capacity>
    <connection-reserve-timeout-seconds>30</connection-reserve-timeout-seconds>
    <inactive-connection-timeout-seconds>900</inactive-connection-timeout-seconds>
    <test-connections-on-reserve>true</test-connections-on-reserve>
    <test-table-name>SQL SELECT 1</test-table-name>
  </jdbc-connection-pool-params>
  <jdbc-data-source-params>
    <jndi-name>jdbc/OrdersDB</jndi-name>
    <global-transactions-protocol>None</global-transactions-protocol>
  </jdbc-data-source-params>
</jdbc-data-source>
```

**DSL Schema Mapping:**
```yaml
JNDIDatasource:
  name: "ordersDataSource"
  jndi_name: "jdbc/OrdersDB"
  server_type: WEBLOGIC
  pool_config:
    max_pool_size: 20
    min_pool_size: 5
    connection_timeout_seconds: 30
    idle_timeout_seconds: 900
    connection_test_query: "SELECT 1"
  spring_boot_migration:
    datasource_url: "jdbc:mysql://db.example.com:3306/orders"
    hikari_maximum_pool_size: 20
    hikari_minimum_idle: 5
    hikari_idle_timeout_ms: 900000
    hikari_connection_timeout_ms: 30000
```

**Migration Effort:** LOW

**Scaling Note:** `max-capacity: 20` is per instance. Sizing formula: `floor((db_max_connections × 0.8) / instance_count)`.

---

#### 16b — XA Datasource (Distributed Transactions)

| Pattern | `<jdbc-xa-data-source>`, `<xa-connection-pool-params>`, `<global-transactions-protocol>TwoPhaseCommit` |
|---------|--------------------------------------------------------------------------------------------------------|
| **File Types** | `*-jdbc.xml`, `config.xml` |
| **Indicates** | Distributed XA transaction datasource |
| **Schema Entity** | `JNDIDatasource` + `TransactionRequirement` |

**DSL Schema Mapping:**
```yaml
JNDIDatasource:
  name: "xaOrdersDataSource"
  jndi_name: "jdbc/OrdersXA"
  server_type: WEBLOGIC
  datasource_type: "javax.sql.XADataSource"

TransactionRequirement:
  transaction_manager: JTA
  supports_distributed_transactions: true
  xa_datasource_detected: true
  legacy_datasource_jndi: "jdbc/OrdersXA"
  scope: REQUIRES_USER_FEEDBACK
```

WARNING: XA/2PC is incompatible with Cloud Foundry ephemeral instances. Flag for manual architecture review.
Spring Boot alternatives: Transactional Outbox pattern or Saga pattern with compensating actions.

**Migration Effort:** HIGH

---

#### 16c — Multi-Datasource Configuration Across Applications

| Pattern | Multiple `<jdbc-data-source>` entries in `config.xml` or referenced in `weblogic-application.xml` |
|---------|----------------------------------------------------------------------------------------------------|
| **File Types** | `weblogic-application.xml`, `config.xml`, `*-jdbc.xml` |
| **Indicates** | Multi-datasource setup across EAR modules |
| **Schema Entity** | `MultiTenancyRequirement` + multiple `JNDIDatasource` |

**DSL Schema Mapping:**
```yaml
MultiTenancyRequirement:
  pattern: DATASOURCE_ROUTING
  module_count: 2

JNDIDatasource:
  name: "ordersDataSource"
  jndi_name: "jdbc/OrdersDB"
  server_type: WEBLOGIC

JNDIDatasource:
  name: "inventoryDataSource"
  jndi_name: "jdbc/InventoryDB"
  server_type: WEBLOGIC
```

**Migration Effort:** MEDIUM

---

### Category 17: Vendor-Specific JNDI Resources (Mail, Messaging, JCA)

#### 17a — Mail Session

| Pattern | `<mail-session>`, `<jndi-name>mail/` |
|---------|--------------------------------------|
| **File Types** | `config.xml`, `*-mail.xml` |
| **Indicates** | WebLogic container-managed mail session |
| **Schema Entity** | `MailRequirement` |

**DSL Schema Mapping:**
```yaml
MailRequirement:
  name: "defaultMailSession"
  legacy_jndi_name: "mail/Default"
  server_type: WEBLOGIC
  properties:
    smtp_host: "smtp.company.com"
    smtp_port: 587
    smtp_auth: true
    starttls_enabled: true
    from_address: "noreply@company.com"
  spring_boot_alternative: "spring.mail.* + JavaMailSender"
```

**Migration Effort:** LOW

---

#### 17b — Oracle AQ / JMS Messaging

| Pattern | `<jms-connection-factory>`, `<jms-queue>`, `<jms-topic>`, `<physical-name>` in `*-jms.xml` |
|---------|---------------------------------------------------------------------------------------------|
| **File Types** | `*-jms.xml`, `config.xml`, `weblogic-ejb-jar.xml` |
| **Indicates** | WebLogic JMS destinations and connection factories |
| **Schema Entity** | `MessagingRequirement` |

**DSL Schema Mapping:**
```yaml
MessagingRequirement:
  name: "orderQueueMessaging"
  pattern: POINT_TO_POINT
  direction: CONSUMER
  destination_type: QUEUE
  destination_name: "OrderQueue"
  server_type: WEBLOGIC
  legacy_config:
    connection_factory_jndi: "jms/OrderQueueCF"
    destination_jndi: "jms/OrderQueue"
  spring_boot_recommendation: "Spring Cloud Stream with RabbitMQ/ActiveMQ binder or @JmsListener"
```

**Migration Effort:** MEDIUM to HIGH

---

### Category 18: Clustering & Session Replication

#### 18a — In-Memory Session Replication (Cluster-Aware)

| Pattern | `<clustering-enabled>true`, `<session-replication-type>MEMORY`, static `<cluster>` in `config.xml` |
|---------|----------------------------------------------------------------------------------------------------|
| **File Types** | `weblogic.xml`, `config.xml` |
| **Indicates** | Static WebLogic cluster in-memory session replication |
| **Schema Entity** | `SessionManagementRequirement` |

WARNING: In-memory session replication assumes static cluster peer topology. Cloud Foundry is ephemeral and dynamic. NOT CF COMPATIBLE.

**DSL Schema Mapping:**
```yaml
SessionManagementRequirement:
  persistence_type: MEMORY_REPLICATED
  server_type: WEBLOGIC
  clustering_enabled: true
  cloud_foundry_compatible: false
  spring_boot_alternative: "redis"
```

**Migration Effort:** HIGH (externalize to Redis)

---

#### 18b — Database-Backed Session Persistence

| Pattern | `<persistent-store-type>jdbc</persistent-store-type>`, `<persistent-store-connection-pool>` |
|---------|---------------------------------------------------------------------------------------------|
| **File Types** | `weblogic.xml` |
| **Indicates** | WebLogic JDBC session persistence |
| **Schema Entity** | `SessionManagementRequirement` |

**DSL Schema Mapping:**
```yaml
SessionManagementRequirement:
  persistence_type: DATABASE
  server_type: WEBLOGIC
  persistence_mechanism: DATABASE_STORE
  cloud_foundry_compatible: true
  spring_boot_alternative: "redis"
  migration_effort: MEDIUM
```

**Migration Effort:** MEDIUM

---

### Category 19: Administration & Monitoring

#### 19a — WebLogic WLST / Runtime MBean Server

| Pattern | `com.bea:Name=RuntimeService,Type=weblogic.management.mbeanservers.runtime` |
|---------|-----------------------------------------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | Programmatic WebLogic administration / runtime querying |
| **Schema Entity** | `MonitoringRequirement` |

**Spring Boot 4 Replacement:** Spring Boot Actuator + Micrometer.

**Migration Effort:** MEDIUM

---

#### 19b — WebLogic JMX Monitoring

| Pattern | `com.bea:Type=*`, `com.oracle.weblogic:*` ObjectNames |
|---------|-------------------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | WebLogic subsystem metrics via JMX MBeans |
| **Schema Entity** | `MonitoringRequirement` |

**DSL Schema Mapping:**
```yaml
MonitoringRequirement:
  metrics_source: JMX
  server_type: WEBLOGIC
  mbean_pattern: "com.bea:*"
  spring_boot_alternative: "micrometer-prometheus"
  prometheus_export: true
  per_instance_tagging: true
  requires_centralized_backend: true
```

**Migration Effort:** LOW

---

### Category 20: Security Realms and Authentication

#### 20a — WebLogic Default Realm (File-Based)

| Pattern | `<realm>`, `DefaultAuthenticator`, `DefaultAuthorizer` in `config.xml` |
|---------|------------------------------------------------------------------------|
| **File Types** | `config.xml` |
| **Indicates** | WebLogic default embedded LDAP / file-based realm |
| **Schema Entity** | `SecurityRequirement` |

**DSL Schema Mapping:**
```yaml
SecurityRequirement:
  authentication_method: FILE_REALM
  realm_type: "DefaultRealm"
  server_type: WEBLOGIC
```

**Migration Effort:** LOW

---

#### 20b — WebLogic LDAP Realm

| Pattern | `LDAPAuthenticator`, host/port/user-base-dn/group-base-dn in `config.xml` |
|---------|---------------------------------------------------------------------------|
| **File Types** | `config.xml` |
| **Indicates** | WebLogic external LDAP authentication |
| **Schema Entity** | `LdapRequirement` + `SecurityRequirement` |

**DSL Schema Mapping:**
```yaml
LdapRequirement:
  name: "corporateLdap"
  provider_url: "ldap://ldap.example.com:389"
  search_base: "dc=example,dc=com"
  bind_dn: "cn=admin,dc=example,dc=com"
  search_filter: "(&(objectClass=user)(sAMAccountName={0}))"
  group_search_base: "ou=groups,dc=example,dc=com"
  server_type: WEBLOGIC

SecurityRequirement:
  authentication_method: LDAP
  realm_type: LDAP
  server_type: WEBLOGIC
```

**Migration Effort:** HIGH

---

#### 20c — Role-Based Access Control (RBAC)

| Pattern | `<security-role-ref>`, `<role-name>`, `<principal-name>` in `weblogic-application.xml` |
|---------|----------------------------------------------------------------------------------------|
| **File Types** | `weblogic-application.xml`, `weblogic.xml` |
| **Indicates** | WebLogic role-to-principal mappings |
| **Schema Entity** | `SecurityRequirement` |

**Migration Effort:** LOW

---

### WebLogic Corner Cases

1. **Multi-tier cluster messaging (multicast vs unicast)**: WebLogic cluster communication relies on multicast or unicast heartbeat discovery. Replace with Redis or external message brokers.
2. **WLST automated deployment scripts**: Replace deployment lifecycle scripting with Cloud Foundry CLI (`cf push`) and CI/CD pipelines.
3. **WebLogic Work Managers** (`<work-manager>`, `<min-threads-constraint>`, `<max-threads-constraint>`): Replace with Spring `@Async` and `ThreadPoolTaskExecutor`.
4. **Custom MBeans registered in WebLogic domain**: Migrate to Spring Boot Actuator `@Endpoint` or Micrometer gauges.
5. **Security realm fallback chaining**: Implement Spring Security `AuthenticationManager` with multiple `AuthenticationProvider` instances.

---

## WebSphere

### Server Identification

| Pattern | File Types | Indicates | Schema Entity |
|---------|------------|-----------|---------------|
| `ibm-web-bnd.xml`, `ibm-web-ext.xml`, `ibm-application-bnd.xml`, `ibm-application-ext.xml` | filename | Traditional WebSphere deployment descriptors | `ModernizationRequirements.source_server = WEBSPHERE` |
| `server.xml`, `server.env`, `bootstrap.properties` (Liberty) | filename | WebSphere Liberty / Open Liberty configuration | `ModernizationRequirements.source_server = WEBSPHERE_LIBERTY` |
| `com\.ibm\.websphere\.`, `com\.ibm\.ws\.` | `*.java` | WebSphere-specific code and APIs | `ServerMigrationRequirement.vendor_dependencies` |

### Category 11: WebSphere Deployment Descriptors & Datasources

#### 11a — Standard Datasource with Connection Pool

| Pattern | `<datasource jndiName="jdbc/`, `<connectionManager maxPoolSize=`, `<properties.mysql` |
|---------|----------------------------------------------------------------------------------------|
| **File Types** | `server.xml`, `ibm-web-bnd.xml`, `ibm-application-bnd.xml` |
| **Indicates** | Container-managed WebSphere datasource with connection pool configuration |
| **Schema Entity** | `JNDIDatasource` |

**Example (WebSphere Liberty `server.xml`):**
```xml
<datasource id="OrdersDataSource" jndiName="jdbc/OrdersDB">
  <jdbcDriver type="javax.sql.DataSource" libraryRef="MySQLLib">
    <properties.mysql
        serverName="db.example.com"
        portNumber="3306"
        databaseName="orders"
        user="appuser"
        password="apppass"/>
  </jdbcDriver>
  <connectionManager maxPoolSize="20" minPoolSize="5"
                    connectionTimeout="30" idleTimeout="900"/>
</datasource>
```

**DSL Schema Mapping:**
```yaml
JNDIDatasource:
  name: "ordersDataSource"
  jndi_name: "jdbc/OrdersDB"
  server_type: WEBSPHERE
  pool_config:
    max_pool_size: 20
    min_pool_size: 5
    idle_timeout_seconds: 900
  spring_boot_migration:
    datasource_url: "jdbc:mysql://db.example.com:3306/orders"
    hikari_maximum_pool_size: 20
    hikari_minimum_idle: 5
    hikari_idle_timeout_ms: 900000
    hikari_connection_timeout_ms: 30000
```

**Migration Effort:** LOW

**Scaling Note:** `maxPoolSize: 20` is a per-instance pool size. On Cloud Foundry with 4 instances, total = 80 connections against DB `max_connections` limit.

---

#### 11b — XA Datasource (Distributed Transactions)

| Pattern | `type="javax.sql.XADataSource"`, `<dataSource ... type="javax.sql.XADataSource">` |
|---------|----------------------------------------------------------------------------------|
| **File Types** | `server.xml`, WebSphere admin configuration |
| **Indicates** | Distributed XA datasource |
| **Schema Entity** | `JNDIDatasource` + `TransactionRequirement` |

**DSL Schema Mapping:**
```yaml
JNDIDatasource:
  name: "xaOrdersDataSource"
  jndi_name: "jdbc/OrdersXA"
  server_type: WEBSPHERE
  datasource_type: "javax.sql.XADataSource"

TransactionRequirement:
  transaction_manager: JTA
  supports_distributed_transactions: true
  xa_datasource_detected: true
  legacy_datasource_jndi: "jdbc/OrdersXA"
  scope: REQUIRES_USER_FEEDBACK
```

WARNING: XA/2PC is NOT supported on Cloud Foundry. Flag for manual architecture review.
Spring Boot alternatives: Transactional Outbox pattern or Saga pattern with compensating actions.

**Migration Effort:** HIGH

---

#### 11c — Multiple Datasources with EJB Binding References

| Pattern | Multiple `<resource-ref>` entries with distinct JNDI names in `ibm-application-bnd.xml` |
|---------|-----------------------------------------------------------------------------------------|
| **File Types** | `ibm-application-bnd.xml`, `ibm-web-bnd.xml` |
| **Indicates** | Multiple independent datasources across modules |
| **Schema Entity** | `MultiTenancyRequirement` + multiple `JNDIDatasource` |

**Example (`ibm-application-bnd.xml`):**
```xml
<application-bnd>
  <ejb-jar-binding ejbJarName="orders-ejb.jar">
    <session name="OrderService">
      <resource-ref name="jdbc/OrdersDB" binding-name="jdbc/OrdersDB"/>
      <resource-ref name="jdbc/AuditDB" binding-name="jdbc/AuditDB"/>
    </session>
  </ejb-jar-binding>
</application-bnd>
```

**DSL Schema Mapping:**
```yaml
MultiTenancyRequirement:
  pattern: DATASOURCE_ROUTING
  module_count: 2

JNDIDatasource:
  name: "ordersDataSource"
  jndi_name: "jdbc/OrdersDB"
  server_type: WEBSPHERE

JNDIDatasource:
  name: "auditDataSource"
  jndi_name: "jdbc/AuditDB"
  server_type: WEBSPHERE
```

**Migration Effort:** MEDIUM (requires multi-datasource `@Configuration` in Spring Boot)

---

### Category 12: Vendor-Specific JNDI Resources (Mail, Messaging, JCA)

#### 12a — Mail Session

| Pattern | `<mailSession jndiName="mail/`, `<properties.smtp` |
|---------|---------------------------------------------------|
| **File Types** | `server.xml` |
| **Indicates** | Container-managed WebSphere mail session |
| **Schema Entity** | `MailRequirement` |

**DSL Schema Mapping:**
```yaml
MailRequirement:
  name: "defaultMailSession"
  legacy_jndi_name: "mail/Default"
  server_type: WEBSPHERE
  properties:
    smtp_host: "smtp.company.com"
    smtp_port: 587
    smtp_auth: true
    starttls_enabled: true
    from_address: "noreply@company.com"
  spring_boot_alternative: "spring.mail.* + JavaMailSender"
```

**Migration Effort:** LOW

---

#### 12b — IBM MQ (WebSphere Messaging)

| Pattern | `<jmsQueueConnectionFactory>`, `<jmsTopicConnectionFactory>`, `<jmsQueue>`, `<jmsTopic>`, `<properties.wmq` |
|---------|-------------------------------------------------------------------------------------------------------------|
| **File Types** | `server.xml`, `ibm-web-bnd.xml` |
| **Indicates** | IBM MQ messaging destinations and connection factories |
| **Schema Entity** | `MessagingRequirement` |

**DSL Schema Mapping:**
```yaml
MessagingRequirement:
  name: "orderQueueMessaging"
  pattern: POINT_TO_POINT
  direction: CONSUMER
  destination_type: QUEUE
  destination_name: "OrderQueue"
  server_type: WEBSPHERE
  legacy_config:
    mq_provider: "IBM_MQ"
    queue_manager: "QM_ORDERS"
    channel: "SYSTEM.DEF.SVRCONN"
    connection_factory_jndi: "jms/OrderQueueCF"
    destination_jndi: "jms/OrderQueue"
  spring_boot_recommendation: "Spring Cloud Stream with IBM MQ binder or @JmsListener"
```

**Migration Effort:** MEDIUM to HIGH

---

#### 12c — JCA Resource Adapters

| Pattern | `<resourceAdapter>`, `<connectionFactory>` referencing RAR |
|---------|------------------------------------------------------------|
| **File Types** | `server.xml`, `ibm-application-bnd.xml`, `*.rar` |
| **Indicates** | Java EE Connector Architecture (JCA) adapter |
| **Schema Entity** | `ExternalResourceRequirement` |

**DSL Schema Mapping:**
```yaml
ExternalResourceRequirement:
  name: "customJcaConnector"
  type: CONNECTION_FACTORY
  legacy_jndi_name: "eis/CustomConnector"
  legacy_server: WEBSPHERE
  properties:
    migration_notes: "JCA adapters require architectural review; replace with REST client or Spring Integration"
```

**Migration Effort:** HIGH (requires case-by-case architectural review)

---

### Category 13: Session Management & WebSphere Clustering

#### 13a — Database-Backed Session Persistence

| Pattern | `DatabaseSessionStore`, `<property name="datasource" value="jdbc/`, `<property name="table_prefix"` |
|---------|------------------------------------------------------------------------------------------------------|
| **File Types** | `server.xml`, `ibm-web-ext.xml` |
| **Indicates** | Database-backed session persistence in WebSphere |
| **Schema Entity** | `SessionManagementRequirement` |

**DSL Schema Mapping:**
```yaml
SessionManagementRequirement:
  persistence_type: DATABASE
  server_type: WEBSPHERE
  persistence_mechanism: DATABASE_STORE
  cloud_foundry_compatible: true
  spring_boot_alternative: "redis"
  migration_effort: MEDIUM
```

**Migration Effort:** MEDIUM

---

#### 13b — Memory-to-Memory Session Replication

| Pattern | References to "Memory-to-Memory Replication" in cluster config or admin console |
|---------|--------------------------------------------------------------------------------|
| **File Types** | `server.xml`, deployment configurations |
| **Indicates** | Static cluster peer memory-to-memory replication |
| **Schema Entity** | `SessionManagementRequirement` |

WARNING: Memory-to-memory replication assumes static cluster topology. Cloud Foundry is ephemeral and dynamic. NOT CF COMPATIBLE.

**DSL Schema Mapping:**
```yaml
SessionManagementRequirement:
  persistence_type: MEMORY_REPLICATED
  server_type: WEBSPHERE
  replication_mechanism: MEMORY_TO_MEMORY
  cloud_foundry_compatible: false
  spring_boot_alternative: "redis"
```

**Migration Effort:** HIGH (flag for manual architecture review)

---

#### 13c — Secure Session Cookies

| Pattern | `secure="true"`, `httpOnly="true"`, `sameSite="Strict"` in `<session-config>` |
|---------|--------------------------------------------------------------------------------|
| **File Types** | `web.xml`, `ibm-web-ext.xml`, `server.xml` |
| **Indicates** | Cookie security settings |
| **Schema Entity** | `SessionManagementRequirement` |

**DSL Schema Mapping:**
```yaml
SessionManagementRequirement:
  session_timeout_minutes: 30
  server_type: WEBSPHERE
  cookie_secure: true
  cookie_http_only: true
  cookie_same_site: "strict"
```

**Migration Effort:** LOW

---

### Category 14: Admin Console APIs and Monitoring

#### 14a — WebSphere Admin Service Client

| Pattern | `AdminService`, `AdminServiceFactory`, `com.ibm.websphere.management.*` |
|---------|-------------------------------------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | Programmatic WebSphere administration API calls |
| **Schema Entity** | `MonitoringRequirement` |

**Spring Boot 4 Replacement:** Micrometer + Spring Boot Actuator.

**Migration Effort:** MEDIUM

---

#### 14b — WebSphere JMX Monitoring

| Pattern | `WebSphere:type=*`, `com.ibm.websphere:*` ObjectNames |
|---------|-------------------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | JMX MBean querying of WebSphere platform metrics |
| **Schema Entity** | `MonitoringRequirement` |

**DSL Schema Mapping:**
```yaml
MonitoringRequirement:
  metrics_source: JMX
  server_type: WEBSPHERE
  mbean_pattern: "WebSphere:*"
  spring_boot_alternative: "micrometer-prometheus"
  requires_centralized_backend: true
  migration_effort: LOW
```

**Migration Effort:** LOW

---

### Category 15: Security Realms and Authentication

#### 15a — File-Based Realm (BasicRegistry)

| Pattern | `<basicRegistry>`, `<user name=`, `<group name=` in Liberty `server.xml` |
|---------|--------------------------------------------------------------------------|
| **File Types** | `server.xml` |
| **Indicates** | WebSphere file-based / basic user registry |
| **Schema Entity** | `SecurityRequirement` |

**DSL Schema Mapping:**
```yaml
SecurityRequirement:
  authentication_method: FILE_REALM
  realm_type: "basicRegistry"
  server_type: WEBSPHERE
  role_mapping_present: true
```

**Migration Effort:** LOW (maps to Spring Security userDetailsService / in-memory or JDBC)

---

#### 15b — Federated Repository + LDAP

| Pattern | `<federatedRepository>`, `<ldapRepository>`, `<idsldapconfiguration>` |
|---------|-----------------------------------------------------------------------|
| **File Types** | `server.xml`, WebSphere security config |
| **Indicates** | WebSphere federated LDAP repository authentication |
| **Schema Entity** | `LdapRequirement` + `SecurityRequirement` |

**DSL Schema Mapping:**
```yaml
LdapRequirement:
  name: "corporateLdap"
  provider_url: "ldap://ldap.example.com:389"
  search_base: "dc=example,dc=com"
  bind_dn: "cn=admin,dc=example,dc=com"
  search_filter: "(&(objectClass=inetOrgPerson)(uid={0}))"
  group_search_base: "ou=groups,dc=example,dc=com"
  server_type: WEBSPHERE

SecurityRequirement:
  authentication_method: LDAP
  server_type: WEBSPHERE
```

**Migration Effort:** HIGH

---

#### 15c — Role-Based Access Control (RBAC) with Security Roles

| Pattern | `<security-role name="...">` + `<user>` / `<group>` in `ibm-application-bnd.xml` |
|---------|-----------------------------------------------------------------------------------|
| **File Types** | `ibm-application-bnd.xml`, `ibm-web-bnd.xml` |
| **Indicates** | WebSphere application role bindings |
| **Schema Entity** | `SecurityRequirement` |

**DSL Schema Mapping:**
```yaml
SecurityRequirement:
  authentication_method: FILE_REALM
  server_type: WEBSPHERE
  roles:
    - name: "admin"
      description: "Administrators"
      permissions: ["READ_ADMIN", "WRITE_ADMIN"]
    - name: "user"
      description: "Regular users"
      permissions: ["READ_USER"]
```

**Migration Effort:** LOW

---

### WebSphere Corner Cases

1. **Multiple authentication mechanisms configured** (LDAP + BasicRegistry fallback): Flag for consolidation to single Spring Security provider chain.
2. **Custom login module classes** (`com.ibm.ws.security.*`): Requires custom Spring Security AuthenticationProvider.
3. **Fine-Grained Access Control (FGA)** via WebSphere authorization policies: Map to Spring Security `@PreAuthorize` SpEL expressions.
4. **WebSphere SSL/TLS mTLS realm configuration**: Ensure Cloud Foundry container keystore contains certificate chain.
5. **Disabled user accounts in registry**: Implement disabled account checking in Spring Security `UserDetails.isEnabled()`.

---

## Tomcat

### Tomcat Context Configuration

| Pattern | `context.xml` (filename in META-INF) |
|---------|---------------------------------------|
| **File Types** | `context.xml` |
| **Indicates** | Tomcat deployment |
| **Schema Entity** | `ServerMigrationRequirement.source_server = TOMCAT` |

**Example:**
```xml
<Context>
    <Resource name="jdbc/OrdersDS"
              type="javax.sql.DataSource"
              driverClassName="com.mysql.cj.jdbc.Driver"
              url="jdbc:mysql://localhost:3306/orders"
              username="app"
              password="secret"
              maxTotal="20"/>
</Context>
```

**Extraction Notes:**
- Extract `<Resource>` definitions → `ExternalResourceRequirement`
- Resource properties map to Spring Boot datasource configuration

---

### Tomcat Package Imports

| Pattern | `org\.apache\.catalina\.` |
|---------|---------------------------|
| **File Types** | `*.java` |
| **Indicates** | Tomcat-specific code |
| **Schema Entity** | `ServerMigrationRequirement.vendor_dependencies` |

---

## Generic Java EE

### Standard web.xml

| Pattern | `web.xml` (filename) |
|---------|----------------------|
| **File Types** | `web.xml` |
| **Indicates** | Java EE web application |
| **Schema Entity** | Multiple entities |

**Key patterns to extract:**
- `<servlet>` definitions
- `<filter>` definitions
- `<listener>` definitions
- `<resource-ref>` references
- `<security-constraint>` rules
- `<welcome-file-list>` configuration

---

### Standard ejb-jar.xml

| Pattern | `ejb-jar.xml` (filename) |
|---------|--------------------------|
| **File Types** | `ejb-jar.xml` |
| **Indicates** | EJB deployment descriptor |
| **Schema Entity** | Multiple entities (EJB configuration) |

---

### Standard application.xml

| Pattern | `application.xml` (filename) |
|---------|------------------------------|
| **File Types** | `application.xml` |
| **Indicates** | Java EE EAR application |
| **Schema Entity** | Application structure |

---

## Datasource connection pool extraction

When a JNDI resource is a **datasource**, extract the connection pool settings from the server
configuration where present, and populate [`JNDIDatasource`](../technical-context-schema.md#jndidatasource)
— not `ExternalResourceRequirement`. `JNDIDatasource.pool_config` holds the vendor numbers as declared, and
`JNDIDatasource.spring_boot_migration` holds the per-instance HikariCP numbers.

**The vendor pool size is not a per-instance size.** It was tuned for one always-on server. Cloud Foundry
runs several instances of the same code, and each opens its own pool, so
`total_connections = pool_size × instance_count`. Record the vendor number unchanged and compute the
per-instance number separately — see the pool sizing rule in `JNDIDatasource`.

Keep `ExternalResourceRequirement.connection_pool` only for a datasource that is not JNDI-bound.

### GlassFish

See [glassfish.md](glassfish.md) Category 1. Pool settings are attributes of `<jdbc-connection-pool>` in
`glassfish-resources.xml`, `sun-resources.xml`, or `domain.xml`, joined to the JNDI name through
`<jdbc-resource pool-name=>`.

### JBoss / WildFly

| Pattern | `<pool>`, `<min-pool-size>`, `<max-pool-size>` |
|---------|--------------------------------------------------|
| **File Types** | `*-ds.xml`, `standalone.xml`, `domain.xml` |
| **Indicates** | Datasource connection pool configuration |
| **Schema Entity** | `JNDIDatasource.pool_config`, `JNDIDatasource.spring_boot_migration` |

**Example:**
```xml
<datasource jndi-name="java:jboss/datasources/OrdersDS" ...>
    <pool>
        <min-pool-size>5</min-pool-size>
        <max-pool-size>20</max-pool-size>
    </pool>
</datasource>
```

**Extraction Notes:**
- `max-pool-size` → `pool_config.max_pool_size` as declared, and a recomputed `spring_boot_migration.hikari_maximum_pool_size`
- `min-pool-size` → `pool_config.min_pool_size` and `spring_boot_migration.hikari_minimum_idle`
- Timeout settings in the descriptor → `pool_config.idle_timeout_seconds` and `pool_config.max_wait_time_millis`, and their millisecond equivalents in `spring_boot_migration`

### WebLogic

Connection pool settings appear in WebLogic datasource configuration (console or JNDI datasource XML). Record `Maximum Capacity`, `Initial Capacity`, and the timeout settings in `JNDIDatasource.pool_config` as declared, and the recomputed per-instance values in `JNDIDatasource.spring_boot_migration`.

### WebSphere / Tomcat

- **WebSphere**: Pool settings in datasource or connection pool configuration in the admin console or config files. Map to `JNDIDatasource.pool_config` and the per-instance `JNDIDatasource.spring_boot_migration` values.
- **Tomcat**: In `context.xml` `<Resource>`, attributes such as `maxTotal`, `maxIdle` map to `maximum_pool_size`, `minimum_idle`. Map to `ExternalResourceRequirement.connection_pool` and `spring.datasource.hikari.*`.

### Schema target

For each JNDI-bound datasource with pool configuration:
- Populate `JNDIDatasource.pool_config` with the vendor values as declared (`max_pool_size`, `min_pool_size`, `idle_timeout_seconds`, `max_wait_time_millis`, `connection_validation_method`, `validation_table`).
- Populate `JNDIDatasource.spring_boot_migration` with the per-instance HikariCP values (`hikari_maximum_pool_size`, `hikari_minimum_idle`, `hikari_idle_timeout_ms`, `hikari_connection_timeout_ms`), which map to `spring.datasource.hikari.maximum-pool-size`, `spring.datasource.hikari.minimum-idle`, `spring.datasource.hikari.idle-timeout`, `spring.datasource.hikari.connection-timeout`.
- Populate `JNDIDatasource.cloud_foundry_service` when the database can be a bound platform service.
