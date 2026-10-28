# Spring Boot 4 Translation: CacheRequirement (Redis Distributed Caching)

Map **CacheRequirement** to Spring Cache with Redis backend on Cloud Foundry. For the core translation guide, see [../SKILL.md](../SKILL.md). For Cloud Foundry service bindings, see [translation-spring-boot-4-cloud-foundry.md](translation-spring-boot-4-cloud-foundry.md).

## The Rule

In-process caches (Caffeine, Simple, Ehcache in-memory), clustered embedded caches (Infinispan, Hazelcast, JGroups), and application server cache managers (WebSphere Dynamic Cache, JBoss HA Cache) are **not suitable for Cloud Foundry**.

On Cloud Foundry:
1. **Multiple instances run concurrently**: In-process caches lead to cache inconsistency across instances.
2. **Instances are ephemeral**: Instance restart/recycle purges local memory.

All caching solutions must be migrated to **Spring Cache with Redis** (or Spring Session + Redis for HTTP sessions).

---

## Spring Cache to Redis

Existing `@Cacheable`, `@CachePut`, and `@CacheEvict` annotations remain **unchanged**.

### Dependencies (Maven)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

### Configuration (`application.yml`)

```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 600000 # TTL in milliseconds
      cache-null-values: false
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD:}
```

### Serialization Requirement

All cached domain objects must implement `java.io.Serializable` or be Jackson-serializable to enable Redis serialization.

---

## Ehcache to Redis

### Migration Actions

1. **Remove Ehcache configuration**: Delete `ehcache.xml` or `META-INF/ehcache.xml`.
2. **Remove Ehcache dependencies**: Delete `org.ehcache:ehcache` or `net.sf.ehcache:ehcache`.
3. **Remove programmatic beans**: Delete custom `EhCacheCacheManager` or `CacheManager` beans configured for Ehcache.
4. **Add Redis starters**: Add `spring-boot-starter-cache` and `spring-boot-starter-data-redis`.
5. **Configure Redis caching in `application.yml`**:
   ```yaml
   spring:
     cache:
       type: redis
       redis:
         time-to-live: 1800000 # Map TTL from ehcache.xml
   ```
6. **Preserve annotations**: `@Cacheable`, `@CachePut`, and `@CacheEvict` work with Spring Boot's auto-configured `RedisCacheManager`.

---

## JCache (JSR-107) and Apache Commons JCS to Redis

### Migration Actions

1. **Convert annotations**:
   - `@CacheResult` → `@Cacheable`
   - `@CachePut` → `@CachePut`
   - `@CacheRemove` → `@CacheEvict`
   - `@CacheRemoveAll` → `@CacheEvict(allEntries = true)`
   - `@CacheKey param` → SpEL `#param` in `key` attribute
2. **Remove dependencies**: Delete `javax.cache:cache-api`, `jakarta.cache:cache-api`, and underlying provider jars.
3. **Delete configuration**: Remove `cache-config.xml` or `jcache.xml`.
   For Apache Commons JCS (`source_technology: APACHE_JCS`), remove `commons-jcs3-core` / `commons-jcs3-jcache` / `jcs`, delete `cache.ccf` and `jcs.properties`, and replace every `JCS.getInstance("region")` and `CacheAccess.get` / `.put` / `.remove` call with `@Cacheable` / `@CachePut` / `@CacheEvict` on the method that owns the lookup. A JCS region name becomes a Spring cache name.
4. **Add Redis starters**: Add `spring-boot-starter-cache` and `spring-boot-starter-data-redis`.
5. **Set `spring.cache.type=redis`** in `application.yml`.

---

## Infinispan and Apache Ignite to Redis

### Migration Actions

1. **Remove peer-to-peer clustering**: Delete `jgroups.xml`, `infinispan.xml`, and any JGroups discovery protocols (`TCPPING`, `JDBC_PING`).
2. **Remove Infinispan dependencies**: Delete `infinispan-core`, `infinispan-spring-boot-starter`, and `jgroups`.
3. **Remove `EmbeddedCacheManager` beans**: Delete custom Infinispan manager classes and `@Bean` definitions.
4. **Convert programmatic API**: Convert `EmbeddedCacheManager.getCache().put()` / `get()` calls to Spring `@CachePut` / `@Cacheable` or Spring Data `RedisTemplate`.
5. **Add Redis starters**: Add `spring-boot-starter-cache` and `spring-boot-starter-data-redis`.
6. **Set `spring.cache.type=redis`** in `application.yml`.
7. **Apache Ignite** (`source_technology: APACHE_IGNITE`) takes the same route: remove `ignite-core` / `ignite-spring`, delete `ignite-config.xml` and the `TcpDiscoveryVmIpFinder` peer list, drop the `Ignition.start(...)` node, and convert `IgniteCache.get` / `.put` calls to Spring Cache annotations. An embedded Ignite node needs peer-to-peer discovery, which the platform does not provide.

---

## HTTP Session State to Redis

### Migration Actions

1. **Add Spring Session Redis dependency**: Add `spring-session-data-redis` and `spring-boot-starter-data-redis`.
2. **Configure `application.yml`**:
   ```yaml
   spring:
     session:
       store-type: redis
       timeout: 30m
       redis:
         namespace: "myapp:session"
   ```
3. **Delete sticky-session from `manifest.yml`**: Remove `sticky-session: true` to enable true horizontal autoscaling.
4. **Transparent code execution**: Do not change any `HttpSession` Java code.
5. **Serialization verification**: Ensure all objects saved in session implement `java.io.Serializable`.

---

## App Server & Proprietary Caching (WebSphere, JBoss, Spymemcached) to Redis

### Migration Actions

1. **Spring Boot baseline prerequisite**: Complete Spring Boot migration first (convert EJBs, servlets, and deployment descriptors).
2. **Remove proprietary caching jars**: Delete WebSphere DynaCache (`com.ibm.websphere.cache.*`), JBoss TreeCache/Cache (`org.jboss.cache.*`), and Spymemcached dependencies.
3. **Remove deployment descriptors**: Delete `cachespec.xml`, `objectgrid.xml`, `jboss-web.xml`, and `ibm-web-ext.xml`.
4. **Convert cache access logic**: Replace proprietary `DistributedMap` / `TreeCache` API calls with Spring `@Cacheable`, `@CachePut`, and `@CacheEvict` annotations.
5. **Add Redis starters**: Add `spring-boot-starter-cache` and `spring-boot-starter-data-redis`.
6. **Set `spring.cache.type=redis`** in `application.yml`.
7. **Ensure `Serializable`**: Verify all cached entity/DTO classes implement `java.io.Serializable`.
