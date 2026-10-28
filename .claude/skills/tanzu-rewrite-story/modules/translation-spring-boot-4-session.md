# Spring Boot 4 Translation: SessionManagementRequirement

Map **SessionManagementRequirement** to Spring Session with an external store. This module also holds the
`Session State` guidance for `migration_considerations`. For the core translation guide, see
[../SKILL.md](../SKILL.md). For the service binding, see
[translation-spring-boot-4-cloud-foundry.md](translation-spring-boot-4-cloud-foundry.md).

## The rule

Keep the application **stateless** and put state in **shared storage**, so that any instance can serve any
request and a restart loses nothing.

A Cloud Foundry application runs several instances, and any instance can be replaced without warning. Every
`persistence_type` in the spec maps to the same target — an external store — because none of the legacy
options survives an instance recycle:

| `persistence_type` | Legacy behaviour | Target |
|--------------------|------------------|--------|
| `REPLICATED` | Server-to-server replication inside a cluster | Spring Session + Redis. The platform has no server-to-server replication. |
| `MEMORY_WITH_FILESTORE` | One instance's heap, written to that instance's local disk | Spring Session + Redis. Local disk is ephemeral. |
| `MEMORY_ONLY` | One instance's heap | Spring Session + Redis, or make the endpoint stateless. |
| `JDBC` | A shared database table | Spring Session JDBC. This one already externalizes state; keep the database unless Redis is preferred. |
| `HA_DEPRECATED` | GlassFish `ha`, removed in later versions | Spring Session + Redis. |

Do not generate a sticky-session route rule as a substitute. Sticky sessions stop true horizontal scaling and
still lose the session when the instance holding it is replaced. Remove any sticky-session rule found in the
legacy `manifest.yml` or router configuration.

## Spring Session with Redis

No application code change is needed. Spring Session replaces the `HttpSession` implementation
transparently, so existing `getAttribute` / `setAttribute` calls keep working.

**Dependency (Maven):**

```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
```

Add `spring-boot-starter-data-redis` as well. For `store-type: jdbc`, use `spring-session-jdbc` instead.

**Configuration (`application.yml`):**

```yaml
spring:
  session:
    store-type: redis
    timeout: 30m
    redis:
      namespace: "spring:session:orders-app"
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD:}
server:
  servlet:
    session:
      cookie:
        secure: true
        http-only: true
```

| Spec field | Spring Boot property |
|------------|----------------------|
| `spring_boot_store_type` | `spring.session.store-type` |
| `session_timeout_minutes` | `spring.session.timeout` (use the `30m` duration form) |
| `cookie_secure` | `server.servlet.session.cookie.secure` |
| `cookie_http_only` | `server.servlet.session.cookie.http-only` |

Set `spring.session.redis.namespace` per application, so two applications sharing one Redis instance do not
collide.

## How session-to-user resolution works

Sessions are keyed by **session ID**, not by user ID. Explain this when the user asks how one instance can
serve a request that another instance started:

1. The **browser** holds the session ID in a cookie (`SESSION` for Spring Session, `JSESSIONID` for a
   servlet container) and sends it on every request.
2. The **application**, through Spring Session's `SessionRepositoryFilter`, looks the session up in Redis by
   that ID.
3. **Redis** returns the serialized session with its attributes, including Spring Security's
   `SecurityContext`.
4. The **user** is resolved from the session — `SecurityContextHolder.getContext().getAuthentication()`.

Cookie (session ID) → Redis lookup → session object → user. There is no in-memory session store and no
"assign session to instance" step, which is exactly why any instance can serve any request.

## Caches on the same Redis

Application caches have the same problem as sessions: an in-process cache is invisible to the other
instances and disappears on a recycle. Use Spring Cache with Redis, and the same bound Redis instance can
serve both sessions and caches. Keep the cache keys under a separate namespace from the session keys.

## Redis on Cloud Foundry

Redis is a managed service. The application does not run it.

1. Create a service instance from the marketplace — `cf create-service <redis-service> <plan> my-redis`.
2. Bind it — `cf bind-service my-app my-redis`, or list it under `services:` in `manifest.yml`.
3. The platform injects the connection details into `VCAP_SERVICES`.
4. Let java-cfenv read `VCAP_SERVICES` and configure `spring.data.redis.*` automatically. Add
   `java-cfenv-boot` when it is not already on the classpath.

Do not write a Redis host or password into `application.yml`. The `${REDIS_HOST:localhost}` form above is
for local development; on the platform the binding supplies the values. Where java-cfenv cannot be used, map
the bound credentials to `spring.data.redis.host`, `.port` and `.password` through environment variables —
do not hand-parse `VCAP_SERVICES` in application code.

```yaml
# manifest.yml
applications:
  - name: orders-app
    instances: 4
    memory: 1G
    services:
      - my-redis
```

## Serialization (the step that is usually missed)

Spring Session writes every session attribute to Redis. Every attribute type must be serializable.

- Check each class the application puts in the session. Add `implements Serializable`, or replace the
  attribute with a small serializable DTO.
- A JPA entity in the session is a defect, not a serialization problem: a detached entity with lazy
  associations fails on access after deserialization. Store the identifier and reload it.
- Prefer JSON over Java serialization for a session that must survive a redeploy with changed classes.
  Configure a `RedisSerializer` and say so in the summary.

## Prefer stateless where the story allows it

Session state is a constraint, not a requirement. Before generating a session store, check what the
application actually keeps in the session:

- An authenticated identity → a bearer token or a signed cookie. No server-side session.
- A short multi-step form → the client holds the partial state, or a single-page flow.
- A shopping cart or another long-lived working set → this genuinely needs the store.

Say which choice was made and why.

## Checklist

- [ ] `spring.session.store-type` is set, and the matching dependency is present
- [ ] `cloud_foundry_compatible: false` in the spec did not produce a local or replicated store
- [ ] No sticky-session rule is generated, and any existing one is removed from `manifest.yml`
- [ ] A Redis service is listed under `services:` in `manifest.yml`, and no host or password is hardcoded
- [ ] Every session attribute type is serializable, and no JPA entity is stored in the session
- [ ] The Redis namespace is specific to this application
