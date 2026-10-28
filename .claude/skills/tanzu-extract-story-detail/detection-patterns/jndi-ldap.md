# JNDI and LDAP Patterns

This document defines detection patterns for programmatic JNDI lookups and direct LDAP directory access in Java EE applications. These patterns identify classes that must be replaced with Spring Boot 4 configuration, Spring LDAP, and Spring Security LDAP during modernization.

Patterns are organized into four tiers by usage severity and migration complexity.

## Table of Contents

- [Tier 1 — Basic JNDI Lookups](#tier-1--basic-jndi-lookups)
- [Tier 2 — LDAP Directory Operations](#tier-2--ldap-directory-operations)
- [Tier 3 — Advanced LDAP Controls](#tier-3--advanced-ldap-controls)
- [Tier 4 — LDAP Events and Listeners](#tier-4--ldap-events-and-listeners)

---

## Tier 1 — Basic JNDI Lookups

These patterns indicate **programmatic JNDI lookups** that bypass `@Resource` injection. Each resolved name is an `ExternalResourceRequirement` that must be explicitly mapped in `application.yml`.

### InitialContext

| Pattern | `InitialContext\s*\(` |
|---------|----------------------|
| **File Types** | `*.java` |
| **Indicates** | Programmatic JNDI lookup — primary entry point for all Java EE resource lookups |
| **Schema Entity** | `ExternalResourceRequirement` |

**Example:**
```java
Context ctx = new InitialContext();
DataSource ds = (DataSource) ctx.lookup("java:comp/env/jdbc/OrdersDS");
```

**Extraction Notes:**
- Extract the string passed to `lookup()` → `legacy_jndi_name`
- Infer `type` from the JNDI name pattern and cast target type:
  - `jdbc/` or `datasources/` and cast to `DataSource` → `DATASOURCE`
  - `jms/` and cast to `Queue`, `Topic`, or `ConnectionFactory` → `QUEUE` / `TOPIC` / `CONNECTION_FACTORY`
  - `mail/` and cast to `Session` → `MAIL_SESSION`
  - Cast to `EJBHome` or `EJBLocalHome` → document in `ServerMigrationRequirement.vendor_dependencies`
- For `DataSource` lookups, attempt to resolve pool configuration from server descriptors (see [app-servers.md](app-servers.md))
- Add to `ServerMigrationRequirement.vendor_dependencies`:
  - `package_pattern: "javax.naming.*"`, `replacement: "Remove — use Spring dependency injection and application.yml"`

---

### CompositeName / CompoundName

| Pattern | `CompositeName\s*\(\|CompoundName\s*\(` |
|---------|-----------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | Programmatic JNDI name construction (multi-namespace or single-namespace name parsing) |
| **Schema Entity** | `ServerMigrationRequirement.vendor_dependencies` |

**Example:**
```java
Name name = new CompositeName("java:comp/env/jdbc/OrdersDS");
```

**Extraction Notes:**
- Extract the string argument → record as a candidate `legacy_jndi_name`
- These classes have no Spring Boot equivalent; document as vendor dependency:
  - `package_pattern: "javax.naming.CompositeName\|javax.naming.CompoundName"`, `replacement: "Remove — use Spring @Value or Environment to resolve names"`

---

### NamingEnumeration / NameClassPair / NameParser

| Pattern | `NamingEnumeration\|NameClassPair\s*\(\|NameParser` |
|---------|-----------------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | JNDI namespace enumeration — the application iterates the JNDI tree to discover resources dynamically |
| **Schema Entity** | `ServerMigrationRequirement`, `migration_considerations` |

**Example:**
```java
NamingEnumeration<NameClassPair> list = ctx.list("java:comp/env");
while (list.hasMore()) {
    NameClassPair pair = list.next();
}
```

**Extraction Notes:**
- Dynamic JNDI enumeration has **no Spring Boot equivalent**
- Add `"JNDI Enumeration"` to `ModernizationRequirements.migration_considerations`
- Document in `ServerMigrationRequirement`:
  - `vendor_dependencies`: `package_pattern: "javax.naming.NamingEnumeration\|javax.naming.NameClassPair"`, `usage_description: "Dynamic JNDI namespace enumeration"`, `replacement: "Re-architect using explicit Spring bean wiring or service locator pattern"`
- Flag for manual review: each `list()` or `listBindings()` call is a resource discovery mechanism that must be converted to explicit Spring configuration

---

### RefAddr / StringRefAddr / BinaryRefAddr / LinkRef

| Pattern | `RefAddr\|StringRefAddr\s*\(\|BinaryRefAddr\s*\(\|LinkRef\s*\(` |
|---------|-------------------------------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | JNDI reference address construction (custom resource factory references) |
| **Schema Entity** | `ServerMigrationRequirement.vendor_dependencies` |

**Extraction Notes:**
- These are used to build `javax.naming.Reference` objects for custom resource factories
- Document as vendor dependency:
  - `package_pattern: "javax.naming.RefAddr\|javax.naming.StringRefAddr\|javax.naming.BinaryRefAddr\|javax.naming.LinkRef"`, `replacement: "Remove — replace custom resource factories with Spring @Bean configuration"`

---

## Tier 2 — LDAP Directory Operations

These patterns indicate **direct LDAP server communication**, typically for user authentication, authorization group lookup, or directory-based profile retrieval. They map to the new `LdapRequirement` schema entity and `IntegrationPoint type: LDAP`.

### InitialLdapContext

| Pattern | `InitialLdapContext\s*\(` |
|---------|--------------------------|
| **File Types** | `*.java` |
| **Indicates** | Direct LDAP connection — primary entry point for all LDAP operations |
| **Schema Entity** | `LdapRequirement`, `IntegrationPoint (type: LDAP)` |

**Example:**
```java
Hashtable<String, String> env = new Hashtable<>();
env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.ldap.LdapCtxFactory");
env.put(Context.PROVIDER_URL, "ldap://ldap.example.com:389");
env.put(Context.SECURITY_PRINCIPAL, "cn=admin,dc=example,dc=com");
env.put(Context.SECURITY_CREDENTIALS, "secret");
LdapContext ctx = new InitialLdapContext(env, null);
```

**Extraction Notes:**
- Extract `Context.PROVIDER_URL` value → `LdapRequirement.provider_url`
- Extract `Context.SECURITY_PRINCIPAL` → `LdapRequirement.bind_dn`
- Extract `Context.SECURITY_CREDENTIALS` source → `LdapRequirement.credentials_source` (prefer `"environment"` if value is a placeholder or system property)
- If URL starts with `ldaps://` → `LdapRequirement.tls_mode = LDAPS`
- If `StartTlsRequest` is also detected → `LdapRequirement.tls_mode = STARTTLS` (see Tier 3)
- Add to `ServerMigrationRequirement.vendor_dependencies`:
  - `package_pattern: "javax.naming.ldap.InitialLdapContext"`, `replacement: "Replace with Spring Boot auto-configured LdapContextSource (spring-boot-starter-data-ldap)"`

---

### DirContext

| Pattern | `DirContext` |
|---------|-------------|
| **File Types** | `*.java` |
| **Indicates** | LDAP directory search, modify, or attribute lookup |
| **Schema Entity** | `LdapRequirement` (search details) |

**Example:**
```java
DirContext ctx = new InitialDirContext(env);
SearchControls controls = new SearchControls();
controls.setSearchScope(SearchControls.SUBTREE_SCOPE);
controls.setReturningAttributes(new String[]{"uid", "cn", "mail", "memberOf"});
NamingEnumeration<SearchResult> results = ctx.search(
    "ou=people,dc=example,dc=com", "(uid={0})", new Object[]{username}, controls);
```

**Extraction Notes:**
- Extract first `search()` argument → `LdapRequirement.search_base`
- Extract filter string → `LddapRequirement.search_filter`
- Extract `setReturningAttributes()` array → `LdapRequirement.returning_attributes`
- `SearchControls.SUBTREE_SCOPE` is the default Spring LDAP scope; note if `ONELEVEL_SCOPE` is used
- Calls to `modifyAttributes()` or `bind()` on `DirContext` indicate write operations on the directory — capture in `LdapRequirement.spring_boot_recommendation` as requiring `LdapTemplate.modify()` / `LdapTemplate.bind()`

---

### LdapName / Rdn

| Pattern | `LdapName\s*\(\|new Rdn\s*\(` |
|---------|-------------------------------|
| **File Types** | `*.java` |
| **Indicates** | LDAP distinguished name (DN) construction or parsing |
| **Schema Entity** | `LdapRequirement.search_base` (if used to build base DN) |

**Example:**
```java
LdapName base = new LdapName("dc=example,dc=com");
Rdn ou = new Rdn("ou", "people");
base.add(ou);
```

**Extraction Notes:**
- Extract the base DN string → `LdapRequirement.search_base` when used as a search root
- `LdapName` and `Rdn` are replaced by Spring LDAP's `LdapNameBuilder` and `LdapUtils.newLdapName()`:
  - `LdapNameBuilder.newInstance("dc=example,dc=com").add("ou", "people").build()`

---

### ManageReferralControl

| Pattern | `ManageReferralControl\s*\(` |
|---------|------------------------------|
| **File Types** | `*.java` |
| **Indicates** | LDAP referral handling strategy |
| **Schema Entity** | `LdapRequirement.referral_handling` |

**Example:**
```java
ctx.setRequestControls(new Control[]{new ManageReferralControl()});
```

**Extraction Notes:**
- Presence of `ManageReferralControl` → `LdapRequirement.referral_handling = IGNORE`
- Maps to `spring.ldap.base-environment.java.naming.referral: ignore` in `application.yml`
- If absent and LDAP tree uses referrals, default behavior is `follow`

---

## Tier 3 — Advanced LDAP Controls

These patterns indicate **non-trivial LDAP operations** that require explicit Spring LDAP `DirContextProcessor` configuration beyond basic `LdapTemplate` usage.

### PagedResultsControl

| Pattern | `PagedResultsControl\s*\(` |
|---------|---------------------------|
| **File Types** | `*.java` |
| **Indicates** | Server-side paging of large LDAP result sets |
| **Schema Entity** | `LdapRequirement.paging_enabled`, `LdapRequirement.page_size` |

**Example:**
```java
int pageSize = 100;
ctx.setRequestControls(new Control[]{new PagedResultsControl(pageSize, Control.CRITICAL)});
```

**Extraction Notes:**
- Extract `pageSize` argument → `LdapRequirement.page_size`
- Set `LdapRequirement.paging_enabled = true`
- Maps to Spring LDAP `PagedResultsDirContextProcessor`:
  ```java
  PagedResultsDirContextProcessor processor = new PagedResultsDirContextProcessor(100);
  ldapTemplate.search(query, mapper, processor);
  ```
- Also detect `PagedResultsResponseControl` — indicates the code reads the paging cookie for multi-page iteration; this pattern requires a loop in the Spring LDAP translation

---

### SortControl / SortKey / SortResponseControl

| Pattern | `SortControl\s*\(\|new SortKey\s*\(\|SortResponseControl\s*\(` |
|---------|----------------------------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | Server-side sorting of LDAP results |
| **Schema Entity** | `LdapRequirement.sort_enabled`, `LdapRequirement.sort_attributes` |

**Example:**
```java
SortKey[] keys = {new SortKey("cn"), new SortKey("uid", false, null)};
ctx.setRequestControls(new Control[]{new SortControl(keys, Control.CRITICAL)});
```

**Extraction Notes:**
- Set `LdapRequirement.sort_enabled = true`
- Extract `SortKey` attribute names → `LdapRequirement.sort_attributes`
- Maps to Spring LDAP `SortControlDirContextProcessor`:
  ```java
  SortControlDirContextProcessor processor = new SortControlDirContextProcessor("cn");
  ldapTemplate.search(query, mapper, processor);
  ```

---

### StartTlsRequest / StartTlsResponse

| Pattern | `StartTlsRequest\s*\(\|StartTlsResponse` |
|---------|------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | TLS upgrade on an existing plaintext LDAP connection (STARTTLS extension) |
| **Schema Entity** | `LdapRequirement.tls_mode = STARTTLS` |

**Example:**
```java
StartTlsResponse tls = (StartTlsResponse) ctx.extendedOperation(new StartTlsRequest());
tls.negotiate();
```

**Extraction Notes:**
- Set `LdapRequirement.tls_mode = STARTTLS`
- Maps to Spring LDAP `DefaultTlsDirContextAuthenticationStrategy`:
  ```java
  @Bean
  public LdapContextSource ldapContextSource() {
      LdapContextSource source = new LdapContextSource();
      source.setUrl("ldap://ldap.example.com:389");
      source.setAuthenticationStrategy(new DefaultTlsDirContextAuthenticationStrategy());
      return source;
  }
  ```
- Note: STARTTLS requires the LDAP server to support the extension; verify during migration

---

### BasicControl / ControlFactory

| Pattern | `BasicControl\s*\(\|ControlFactory` |
|---------|-------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | Custom LDAP controls or control factory |
| **Schema Entity** | `LdapRequirement.ldap_controls` |

**Extraction Notes:**
- Extract subclass names → `LdapRequirement.ldap_controls` array
- Custom controls require manual migration; add to `ServerMigrationRequirement.vendor_dependencies`:
  - `package_pattern: "javax.naming.ldap.BasicControl\|javax.naming.ldap.ControlFactory"`, `usage_description: "Custom LDAP controls"`, `replacement: "Implement as Spring LDAP DirContextProcessor or use spring-security-ldap RequestControlDirContextProcessor"`

---

## Tier 4 — LDAP Events and Listeners

These patterns indicate **directory change notification** subscriptions. They have **no direct Spring LDAP equivalent** and require manual re-architecture.

### NamingEvent / NamingExceptionEvent

| Pattern | `NamingEvent\s*\(\|NamingExceptionEvent\s*\(` |
|---------|-----------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | LDAP directory change notifications (entry added, removed, or renamed) |
| **Schema Entity** | `LdapRequirement.event_listeners`, `migration_considerations` |

**Example:**
```java
EventDirContext eventCtx = (EventDirContext) ctx.lookup("ou=people,dc=example,dc=com");
eventCtx.addNamingListener("", EventContext.SUBTREE_SCOPE, new NamingListener() {
    public void objectChanged(NamingEvent evt) { ... }
    public void namingExceptionThrown(NamingExceptionEvent evt) { ... }
});
```

**Extraction Notes:**
- Extract `NamingListener` implementation class names → `LdapRequirement.event_listeners`
- Add `"LDAP Events"` to `ModernizationRequirements.migration_considerations`
- Spring LDAP has no built-in `NamingListener` equivalent
- Document in `ServerMigrationRequirement.vendor_dependencies`:
  - `package_pattern: "javax.naming.event.NamingListener\|javax.naming.event.NamingEvent"`, `usage_description: "LDAP directory change listener"`, `replacement: "Re-architect: implement polling via @Scheduled + LdapTemplate.search(), or use LDAP persistent search if server supports it"`

---

### UnsolicitedNotificationEvent

| Pattern | `UnsolicitedNotificationEvent` |
|---------|-------------------------------|
| **File Types** | `*.java` |
| **Indicates** | Server-initiated (push) notification from the LDAP server |
| **Schema Entity** | `LdapRequirement.event_listeners`, `migration_considerations` |

**Example:**
```java
EventContext eventCtx = (EventContext) ctx;
eventCtx.addNamingListener("", EventContext.SUBTREE_SCOPE, new UnsolicitedNotificationListener() {
    public void notificationReceived(UnsolicitedNotificationEvent evt) { ... }
    public void namingExceptionThrown(NamingExceptionEvent evt) { ... }
});
```

**Extraction Notes:**
- Add `"LDAP Events"` to `ModernizationRequirements.migration_considerations` (same tag as `NamingEvent`)
- Document as vendor dependency requiring re-architecture:
  - `package_pattern: "javax.naming.ldap.UnsolicitedNotificationEvent\|javax.naming.ldap.UnsolicitedNotificationListener"`, `replacement: "Re-architect: no Spring LDAP equivalent for server-push; consider LDAP persistent search extension or application-level polling"`

---

## Schema Entity Summary

| Pattern Group | Primary Schema Entity | Secondary Entity |
|---|---|---|
| `InitialContext` | `ExternalResourceRequirement` | `ServerMigrationRequirement.vendor_dependencies` |
| `CompositeName`, `CompoundName` | `ServerMigrationRequirement.vendor_dependencies` | — |
| `NamingEnumeration`, `NameClassPair` | `ServerMigrationRequirement` | `migration_considerations: "JNDI Enumeration"` |
| `RefAddr`, `StringRefAddr`, `BinaryRefAddr`, `LinkRef` | `ServerMigrationRequirement.vendor_dependencies` | — |
| `InitialLdapContext` | `LdapRequirement` | `IntegrationPoint (type: LDAP)` |
| `DirContext` | `LdapRequirement` (search_base, search_filter, returning_attributes) | — |
| `LdapName`, `Rdn` | `LdapRequirement.search_base` | — |
| `ManageReferralControl` | `LdapRequirement.referral_handling` | — |
| `PagedResultsControl`, `PagedResultsResponseControl` | `LdapRequirement.paging_enabled`, `page_size` | — |
| `SortControl`, `SortKey`, `SortResponseControl` | `LdapRequirement.sort_enabled`, `sort_attributes` | — |
| `StartTlsRequest`, `StartTlsResponse` | `LdapRequirement.tls_mode = STARTTLS` | — |
| `BasicControl`, `ControlFactory` | `LdapRequirement.ldap_controls` | `ServerMigrationRequirement.vendor_dependencies` |
| `NamingEvent`, `NamingExceptionEvent` | `LdapRequirement.event_listeners` | `migration_considerations: "LDAP Events"` |
| `UnsolicitedNotificationEvent` | `LdapRequirement.event_listeners` | `migration_considerations: "LDAP Events"` |
