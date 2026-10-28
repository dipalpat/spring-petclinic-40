# Caching Detection Patterns

This document covers detection patterns for in-process, distributed, and application-server-coupled caching solutions, mapping them to `CacheRequirement` and modernizing them to Spring Boot 4 + Redis on Cloud Foundry.

---

## Spring Cache (Caffeine / Simple)

### Detection Patterns

| Pattern | File Types | Indicates | Schema Entity |
|---------|------------|-----------|---------------|
| `@EnableCaching` | `*.java` | Spring Cache enabled | `CacheRequirement` |
| `spring.cache.type=caffeine`, `spring.cache.type=simple` | `application.yml`, `application.properties` | In-process Spring Cache provider | `CacheRequirement` |
| `CaffeineCacheManager`, `ConcurrentMapCacheManager`, `SimpleCacheManager` | `*.java` | Programmatic in-memory cache manager bean | `CacheRequirement` |
| `caffeine.spec`, `com.github.ben-manes.caffeine:caffeine` | `pom.xml`, `build.gradle`, `application.yml` | Caffeine cache library dependency / spec | `CacheRequirement` |
| `@Cacheable`, `@CachePut`, `@CacheEvict`, `@Caching` | `*.java` | Spring declarative caching annotations | `CacheRequirement` (cache names) |

### DSL Schema Mapping

```yaml
CacheRequirement:
  name: "springCache"
  source_technology: SPRING_CACHE
  source_backend: "caffeine"
  scope: INSTANCE_LOCAL
  is_distributed: false
  cache_names: ["users", "products"]
  cloud_foundry_compatible: false
  spring_boot_alternative: "spring.cache.type=redis + spring-boot-starter-data-redis"
  migration_effort: LOW
```

### Spring Boot 4 + Redis Target Mapping

**Dependencies (`pom.xml`):**
```xml
<!-- Remove: com.github.ben-manes.caffeine:caffeine -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

**Configuration (`application.yml`):**
```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 600000 # 10m in ms
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD:}
      jedis:
        pool:
          max-active: 15
```

**Cloud Foundry Considerations:**
- In-process caches lose cached items upon instance recycle and do not share entries across horizontal CF instances.
- Migrating to `spring.cache.type=redis` allows all CF instances to share the same distributed cache with zero code changes to existing `@Cacheable` annotations.

---

## Ehcache

### Detection Patterns

| Pattern | File Types | Indicates | Schema Entity |
|---------|------------|-----------|---------------|
| `org.ehcache:ehcache`, `net.sf.ehcache:ehcache` | `pom.xml`, `build.gradle` | Ehcache library dependency (v2 or v3) | `CacheRequirement` |
| `ehcache.xml`, `META-INF/ehcache.xml`, `ehcache-failsafe.xml` | filename | Ehcache XML configuration file | `CacheRequirement` |
| `spring.cache.type=ehcache`, `spring.cache.ehcache.config` | `application.yml`, `application.properties` | Spring Boot Ehcache configuration | `CacheRequirement` |
| `org.ehcache.CacheManager`, `net.sf.ehcache.CacheManager`, `EhCacheCacheManager`, `EhCacheManagerFactoryBean` | `*.java` | Programmatic Ehcache cache manager | `CacheRequirement` |
| `<cache alias="`, `<ehcache`, `<heap unit=`, `<expiry>` | `ehcache.xml` | Ehcache cache definitions, heap limits, TTL | `CacheRequirement` |

### DSL Schema Mapping

```yaml
CacheRequirement:
  name: "userCache"
  source_technology: EHCACHE
  source_backend: "ehcache"
  scope: INSTANCE_LOCAL
  is_distributed: false
  cache_names: ["userCache"]
  ttl_seconds: 1800
  max_entries: 5000
  cloud_foundry_compatible: false
  spring_boot_alternative: "spring.cache.type=redis + spring-boot-starter-data-redis"
  migration_effort: LOW
```

### Spring Boot 4 + Redis Target Mapping

**Dependencies (`pom.xml`):**
```xml
<!-- Remove: org.ehcache:ehcache, net.sf.ehcache:ehcache -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

**Configuration (`application.yml`):**
```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 1800000 # 30m TTL from Ehcache XML
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD:}
```

**Migration Steps:**
1. Remove `ehcache.xml` and Ehcache dependencies from `pom.xml`.
2. Add `spring-boot-starter-data-redis` and `spring-boot-starter-cache`.
3. Configure `spring.cache.type=redis` and map `<expiry><ttl>` settings to `spring.cache.redis.time-to-live`.
4. Ensure all cached object types implement `java.io.Serializable`.
5. Remove any custom `EhCacheCacheManager` `@Bean` methods.

---

## JCache (JSR-107) and Apache Commons JCS

### Detection Patterns

| Pattern | File Types | Indicates | Schema Entity |
|---------|------------|-----------|---------------|
| `javax.cache:cache-api`, `jakarta.cache:cache-api` | `pom.xml`, `build.gradle` | JSR-107 API dependency | `CacheRequirement` |
| `@CacheResult`, `@CachePut`, `@CacheRemove`, `@CacheRemoveAll`, `@CacheKey`, `@CacheValue` | `*.java` | JSR-107 annotations (`javax.cache.annotation.*` / `jakarta.cache.annotation.*`) | `CacheRequirement` |
| `javax.cache.CacheManager`, `javax.cache.Caching` | `*.java` | Programmatic JSR-107 cache usage | `CacheRequirement` |
| `spring.cache.type=jcache`, `spring.cache.jcache.config` | `application.yml`, `application.properties` | Spring JCache provider config | `CacheRequirement` |
| `cache-config.xml`, `jcache.xml` | filename | JCache provider XML configuration | `CacheRequirement` |
| `org.apache.commons:commons-jcs3-core`, `org.apache.commons:commons-jcs3-jcache`, `org.apache.jcs:jcs` | `pom.xml`, `build.gradle` | Apache Commons JCS dependencies | `CacheRequirement` |
| `JCS.getInstance(`, `CacheAccess`, `org.apache.commons.jcs3.*`, `org.apache.jcs.*` | `*.java` | Apache Commons JCS cache access | `CacheRequirement` |
| `cache.ccf`, `jcs.properties`, `jcs.default=` | filename, `*.properties` | Apache Commons JCS configuration | `CacheRequirement` |

### Annotation Translation Reference

| JSR-107 Annotation | Spring Cache Annotation |
|--------------------|-------------------------|
| `@CacheResult(cacheName = "orders")` | `@Cacheable(cacheNames = "orders")` |
| `@CachePut(cacheName = "orders")` | `@CachePut(cacheNames = "orders")` |
| `@CacheRemove(cacheName = "orders")` | `@CacheEvict(cacheNames = "orders")` |
| `@CacheRemoveAll(cacheName = "orders")` | `@CacheEvict(cacheNames = "orders", allEntries = true)` |
| `@CacheKey String orderId` | `@Cacheable(..., key = "#orderId")` |

### DSL Schema Mapping

```yaml
CacheRequirement:
  name: "orderCache"
  source_technology: JCACHE # or APACHE_JCS
  source_backend: "jcache"
  scope: INSTANCE_LOCAL
  is_distributed: false
  cache_names: ["orders"]
  cloud_foundry_compatible: false
  spring_boot_alternative: "spring.cache.type=redis + spring-boot-starter-data-redis"
  migration_effort: LOW
```

### Spring Boot 4 + Redis Target Mapping

**Dependencies (`pom.xml`):**
```xml
<!-- Remove: javax.cache:cache-api, jakarta.cache:cache-api, provider libs -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

**Configuration (`application.yml`):**
```yaml
spring:
  cache:
    type: redis
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD:}
```

---

## Infinispan and Apache Ignite (Embedded Data Grids)

### Detection Patterns

| Pattern | File Types | Indicates | Schema Entity |
|---------|------------|-----------|---------------|
| `org.infinispan:infinispan-core`, `infinispan-spring-boot-starter`, `org.jgroups:jgroups` | `pom.xml`, `build.gradle` | Infinispan / JGroups clustering dependencies | `CacheRequirement` |
| `EmbeddedCacheManager`, `DefaultCacheManager`, `InfinispanCacheConfigurer` | `*.java` | Embedded Infinispan cluster manager bean | `CacheRequirement` |
| `CacheMode.REPL_SYNC`, `CacheMode.DIST_SYNC`, `CacheMode.LOCAL` | `*.java`, `*.xml` | Infinispan clustering cache mode | `CacheRequirement` |
| `infinispan.xml`, `jgroups.xml`, `jgroups-tcp.xml` | filename | Infinispan / JGroups cluster configuration | `CacheRequirement` |
| `infinispan.embedded.enabled=true`, `infinispan.jgroups.config` | `application.yml`, `application.properties` | Infinispan embedded Spring Boot config | `CacheRequirement` |
| `org.apache.ignite:ignite-core`, `ignite-spring`, `ignite-spring-data` | `pom.xml`, `build.gradle` | Apache Ignite data grid dependencies | `CacheRequirement` |
| `Ignition.start(`, `IgniteCache`, `IgniteConfiguration`, `SpringCacheManager` | `*.java` | Embedded Apache Ignite node and cache access | `CacheRequirement` |
| `ignite-config.xml`, `default-config.xml`, `TcpDiscoveryVmIpFinder` | filename, `*.xml` | Apache Ignite cluster discovery configuration | `CacheRequirement` |

### DSL Schema Mapping

```yaml
CacheRequirement:
  name: "infinispanCache"
  source_technology: INFINISPAN # or APACHE_IGNITE
  source_backend: "infinispan"
  scope: CLUSTERED
  is_distributed: true
  cloud_foundry_compatible: false
  spring_boot_alternative: "spring.cache.type=redis + spring-boot-starter-data-redis"
  migration_effort: HIGH
  migration_notes: "Infinispan embedded data grid relies on JGroups peer-to-peer clustering which is incompatible with Cloud Foundry ephemeral containers. Migrate to Spring Cache with external Redis."
```

### Spring Boot 4 + Redis Target Mapping

**Dependencies (`pom.xml`):**
```xml
<!-- Remove: org.infinispan:*, org.jgroups:* -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

**Configuration (`application.yml`):**
```yaml
spring:
  cache:
    type: redis
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD:}
```

**Migration Steps:**
1. Remove all Infinispan (`infinispan-core`, `infinispan-spring-boot-starter`) and JGroups dependencies.
2. Delete `jgroups.xml` and `infinispan.xml` clustering files.
3. Remove custom `EmbeddedCacheManager` and `DefaultCacheManager` configuration classes.
4. Replace programmatic cache API calls (`cache.put()`, `cache.get()`) with standard `@Cacheable`, `@CachePut`, and `@CacheEvict` annotations.
5. Add `spring-boot-starter-data-redis` and `spring-boot-starter-cache`.
6. Configure `spring.cache.type=redis` in `application.yml`.

---

## HTTP Session State to Redis

### Detection Patterns

| Pattern | File Types | Indicates | Schema Entity |
|---------|------------|-----------|---------------|
| `HttpSession`, `request.getSession()`, `session.setAttribute(`, `session.getAttribute(` | `*.java` | Direct HTTP session usage | `SessionManagementRequirement` / `CacheRequirement` |
| `@SessionAttributes`, `@SessionAttribute`, `SessionStatus` | `*.java` | Spring MVC session attributes | `SessionManagementRequirement` / `CacheRequirement` |
| `sticky-session: true` | `manifest.yml` | Cloud Foundry sticky-session load balancing (anti-pattern) | `SessionManagementRequirement` |
| `server.servlet.session.timeout`, `<session-timeout>` | `application.yml`, `web.xml` | Session timeout configuration | `SessionManagementRequirement` |

### DSL Schema Mapping

```yaml
SessionManagementRequirement:
  persistence_type: MEMORY_ONLY
  clustering_enabled: false
  cloud_foundry_compatible: false
  session_timeout_minutes: 30
  spring_boot_store_type: redis
  spring_properties:
    spring.session.store-type: "redis"
    spring.session.redis.namespace: "spring:session:app"
    server.servlet.session.timeout: "30m"
  migration_effort: LOW

CacheRequirement:
  name: "httpSessionState"
  source_technology: SESSION_STORAGE
  source_backend: "http-session"
  scope: SESSION_STORAGE
  is_distributed: false
  cloud_foundry_compatible: false
  spring_boot_alternative: "spring-session-data-redis + spring.session.store-type=redis"
  migration_effort: LOW
```

### Spring Boot 4 + Redis Target Mapping

**Dependencies (`pom.xml`):**
```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

**Configuration (`application.yml`):**
```yaml
spring:
  session:
    store-type: redis
    timeout: 30m
    redis:
      namespace: "myapp:session"
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD:}
```

**Cloud Foundry Manifest (`manifest.yml`):**
- **Delete `sticky-session: true`** from `manifest.yml`.
- Bind the Redis service instance under `services:`.
- Application code using `HttpSession` requires **zero code modifications** because Spring Session transparently intercepts session management.
- All session attribute values must implement `java.io.Serializable`.

---

## Application Server & Proprietary Caching (WebSphere, JBoss, Spymemcached)

### Detection Patterns

| Pattern | File Types | Indicates | Schema Entity |
|---------|------------|-----------|---------------|
| `com.ibm.websphere.cache.*`, `DistributedObjectCache`, `DistributedMap`, `cachespec.xml` | `*.java`, `*.xml` | WebSphere Dynamic Cache (DynaCache) | `CacheRequirement` |
| `com.ibm.websphere.objectgrid.*`, `objectgrid.xml`, `objectgrid-deployment.xml` | `*.java`, `*.xml` | WebSphere eXtreme Scale / ObjectGrid | `CacheRequirement` |
| `org.jboss.cache.*`, `TreeCache`, `PojoCache`, `JBossCacheManager` | `*.java`, `*.xml` | JBoss Cache / TreeCache clustering | `CacheRequirement` |
| `<replication-config>`, `<cache-name>websession</cache-name>` | `jboss-web.xml`, `ibm-web-ext.xml` | App-server clustered session/cache replication | `CacheRequirement` / `SessionManagementRequirement` |
| `net.spy.spymemcached.*`, `MemcachedClient` | `*.java`, `pom.xml` | Spymemcached memcached client | `CacheRequirement` |
| `com.hazelcast.core.HazelcastInstance`, `hazelcast.xml` | `*.java`, `*.xml` | Hazelcast embedded cluster cache | `CacheRequirement` |

### DSL Schema Mapping

```yaml
CacheRequirement:
  name: "appServerCache"
  source_technology: WEBSPHERE_CACHE # or JBOSS_CACHE, SPYMEMCACHED, WEBSPHERE_OBJECT_GRID, HAZELCAST
  source_backend: "com.ibm.websphere.cache.DistributedObjectCache"
  scope: CLUSTERED
  is_distributed: true
  server_type: WEBSPHERE
  cloud_foundry_compatible: false
  spring_boot_alternative: "spring.cache.type=redis + spring-boot-starter-data-redis"
  migration_effort: HIGH
  migration_notes: "App server cache APIs and deployment descriptors must be removed during Spring Boot migration. Migrate proprietary cache put/get operations to Spring Cache annotations backed by Redis."
```

### Spring Boot 4 + Redis Target Mapping

**Dependencies (`pom.xml`):**
```xml
<!-- Remove: com.ibm.websphere:cache, org.jboss:jboss-cache, net.spy:spymemcached, etc. -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

**Configuration (`application.yml`):**
```yaml
spring:
  cache:
    type: redis
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD:}
```

**Migration Steps:**
1. **Prerequisite:** Perform Spring Boot baseline migration first to eliminate application server container dependencies.
2. Remove vendor caching libraries (`com.ibm.websphere:cache`, `org.jboss:jboss-cache`, `spymemcached`) from build descriptors.
3. Remove deployment descriptors (`cachespec.xml`, `objectgrid.xml`, `jboss-web.xml`, `ibm-web-ext.xml`).
4. Replace proprietary programmatic cache access (`cache.get()`, `cache.put()`, `cache.remove()`) with Spring Cache annotations (`@Cacheable`, `@CachePut`, `@CacheEvict`) or `RedisTemplate`.
5. Add `spring-boot-starter-cache` and `spring-boot-starter-data-redis`.
6. Configure `spring.cache.type=redis` in `application.yml`.
7. Verify all cached domain models implement `java.io.Serializable`.
