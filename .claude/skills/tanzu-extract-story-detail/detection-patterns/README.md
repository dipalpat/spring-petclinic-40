# Detection Patterns

This folder contains grep/regex patterns for detecting legacy application constructs and mapping them to the Technical Context Schema entities. Use these patterns to analyze source code and extract requirements for Spring Boot modernization.

## Pattern Files

| File | Description |
|------|-------------|
| [spring-boot.md](spring-boot.md) | Spring Boot 2.x / 3.x / 4.x patterns (code and config only; no build files) |
| [spring-web.md](spring-web.md) | Spring Web (non-Boot) patterns for controllers, services, transactions, etc. |
| [ui-layer.md](ui-layer.md) | UI layer patterns for JSP, Tiles, JSF, SiteMesh, Thymeleaf, and Freemarker (Struts summary with cross-reference) |
| [struts.md](struts.md) | **Comprehensive Apache Struts 1.x and 2.x patterns** including actions, forms, validation, interceptors, Tiles integration, and migration guide |
| [ejb.md](ejb.md) | Enterprise JavaBeans patterns for session beans, MDBs, timers, etc. |
| [jndi-ldap.md](jndi-ldap.md) | Programmatic JNDI lookups (`InitialContext`) and direct LDAP access (`InitialLdapContext`, `DirContext`, LDAP controls, event listeners) |
| [picocli.md](picocli.md) | Picocli CLI framework patterns for commands, options, subcommands, and Spring Boot integration |
| [cobol.md](cobol.md) | COBOL patterns for program structure, data definitions, CICS, and JCL |
| [app-servers.md](app-servers.md) | **Start here for server detection.** Identifies the application server, then routes to the per-vendor file. Holds the JBoss, WebLogic, WebSphere and Tomcat patterns. |
| [glassfish.md](glassfish.md) | **Comprehensive GlassFish, Payara and Sun Application Server patterns** — datasources with vendor pools, JNDI mail and JMS resources, session replication, JMX and admin API monitoring, and security realms |
| [mulesoft.md](mulesoft.md) | **MuleSoft integration patterns** — HTTP listeners, database/JMS/mail/FTP-SFTP connectors, DataWeave transformations, and error handling policies |
| [caching.md](caching.md) | **Caching patterns** — Spring Cache (Caffeine/Simple), Ehcache, JCache and Apache Commons JCS, Infinispan and Apache Ignite, Hazelcast, app-server and proprietary caching to Redis |
| [jpa-persistence.md](jpa-persistence.md) | Javax/Jakarta JPA entity property extraction; use whenever Spring Boot, Spring Web, or EJB is in use |
| [usage-guidelines.md](usage-guidelines.md) | Guidelines for detection workflow, pattern priority, and output format |

---

## Overview

### Pattern Structure

Each pattern entry includes:

| Field | Description |
|-------|-------------|
| **Pattern** | The grep/regex pattern to search for |
| **File Types** | File extensions to search (e.g., `*.java`, `*.xml`) |
| **Indicates** | What the presence of this pattern means |
| **Schema Entity** | Which Technical Context Schema entity this maps to |
| **Example** | Sample code that matches this pattern |
| **Extraction Notes** | How to extract structured data from matches |

### Pattern Syntax

- Patterns use standard regex syntax compatible with `grep -E` or `ripgrep`
- Escape special characters as needed for your search tool
- Some patterns are for file names/paths rather than content

---

## Quick Reference

### Technology Detection Summary

| Technology | Key Patterns | Pattern File |
|------------|--------------|--------------|
| Spring Boot 2/3/4 | `@SpringBootApplication`, `SpringApplication.run`, `application.yml`, `javax`/`jakarta` | [spring-boot.md](spring-boot.md) |
| Spring Web | `DispatcherServlet`, `@Controller`, `<beans xmlns` | [spring-web.md](spring-web.md) |
| JSP | `*.jsp`, JSTL taglibs, Spring form tags | [ui-layer.md](ui-layer.md) |
| Tiles | `tiles.xml`, `TilesConfigurer` | [ui-layer.md](ui-layer.md) |
| Struts 1.x | `struts-config.xml`, `extends Action`, `extends ActionForm`, `struts-html` taglib | **[struts.md](struts.md)** |
| Struts 2.x | `struts.xml`, `extends ActionSupport`, `@Action`, `/struts-tags` taglib, `StrutsPrepareAndExecuteFilter` | **[struts.md](struts.md)** |
| JSF | `faces-config.xml`, `@ManagedBean`, `*.xhtml` | [ui-layer.md](ui-layer.md) |
| EJB | `@Stateless`, `@Stateful`, `@MessageDriven` | [ejb.md](ejb.md) |
| JNDI (programmatic) | `InitialContext(`, `CompositeName(`, `NamingEnumeration` | [jndi-ldap.md](jndi-ldap.md) |
| LDAP (direct) | `InitialLdapContext(`, `DirContext`, `LdapName(`, `*Control(` | [jndi-ldap.md](jndi-ldap.md) |
| Picocli CLI | `@Command`, `@Option`, `CommandLine.execute`, `picocli-spring-boot-starter` | [picocli.md](picocli.md) |
| JPA entities (javax/jakarta) | `@Entity`, `@Table`, `@Column` | [jpa-persistence.md](jpa-persistence.md) — always run with Spring Boot, Spring Web, or EJB |
| COBOL | `IDENTIFICATION DIVISION`, `EXEC SQL`, `EXEC CICS` | [cobol.md](cobol.md) |
| GlassFish | `glassfish-web.xml`, `glassfish-resources.xml`, `sun-web.xml`, `org.glassfish.*` / `com.sun.enterprise.*` imports | **[glassfish.md](glassfish.md)** |
| JBoss | `jboss-web.xml`, `java:jboss/` | [app-servers.md](app-servers.md) |
| WebLogic | `weblogic.xml`, `weblogic.*` imports | [app-servers.md](app-servers.md) |
| WebSphere | `ibm-web-bnd.xml`, `com.ibm.websphere.*` | [app-servers.md](app-servers.md) |
| Tomcat | `context.xml`, `org.apache.catalina.*` | [app-servers.md](app-servers.md) |
| MuleSoft | `mule-config.xml`, `<mule xmlns=`, `<flow name=`, `org.mule.*` | **[mulesoft.md](mulesoft.md)** |
| Caching | `@EnableCaching`, `spring.cache.type`, `ehcache.xml`, `javax.cache`, `JCS.getInstance(`, `infinispan`, `Ignition.start(`, `hazelcast.xml` | **[caching.md](caching.md)** |

### Related Documentation

- [Technical Context Schema](../technical-context-schema.md) - Schema definitions for extracted requirements
