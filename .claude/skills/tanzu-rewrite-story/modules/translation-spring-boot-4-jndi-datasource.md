# Spring Boot 4 Translation: JNDIDatasource, MultiTenancyRequirement

Map **JNDIDatasource** to `spring.datasource.*` with HikariCP, and **MultiTenancyRequirement** to an
`AbstractRoutingDataSource`. For the core translation guide, see [../SKILL.md](../SKILL.md). For the service
binding, see [translation-spring-boot-4-cloud-foundry.md](translation-spring-boot-4-cloud-foundry.md).

## Patterns

- **Remove the lookup.** Delete every `InitialContext`, `lookup(...)`, and `javax.naming` import. Inject
  `DataSource` through the constructor. Delete the `<resource-ref>` block and the vendor resource file.
- **Configuration:** `spring.datasource.url`, `.username`, `.password`, `.driver-class-name`. On Cloud
  Foundry, prefer the bound service over literal values; java-cfenv reads `VCAP_SERVICES`.
- **Pool:** `spring.datasource.hikari.maximum-pool-size`, `.minimum-idle`, `.idle-timeout`,
  `.connection-timeout`. Take the values from `spring_boot_migration`, never from `pool_config`.
- **Repository:** with JPA, `DataSource` needs no direct use. Inject a Spring Data repository instead.

## Pool sizing (mandatory — do not skip)

The vendor number in `pool_config.max_pool_size` was tuned for one always-on server. Cloud Foundry runs
`instances:` copies of the same code, and each opens its own pool:

```
total_connections = maximum-pool-size × instance_count
```

Compute the per-instance number:

```
safe_per_instance_pool_size = floor((db_max_connections × 0.8) / max_planned_instances)
```

Reserve the remaining 20% for migrations, admin sessions, and other applications on the same database. When
an autoscaler is in use, `max_planned_instances` is the autoscaler ceiling, not the current count.

**Worked example.** GlassFish declared `max-pool-size="50"`. The target runs `instances: 4` against a
database with `max_connections = 200`:

```
naive port:  50 × 4 = 200 connections  → the database limit is reached with zero headroom
safe size:   floor((200 × 0.8) / 4) = 40
```

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 40   # NOT the legacy 50
      minimum-idle: 10
      idle-timeout: 900000
      connection-timeout: 6000
```

When `db_max_connections` or the instance ceiling is unknown, do not guess. Emit the legacy value with a
`TODO` comment naming both missing numbers, and say so in the summary to the user.

**Alternative when the numbers cannot be reduced:** front the database with a connection-pooling proxy
(PgBouncer, or a managed database proxy) so `instances × pool_size` no longer maps 1:1 to real database
connections. Recommend this instead of an unsafe pool size.

## Vendor settings with no HikariCP equivalent

| Vendor setting | Action |
|----------------|--------|
| `fail-all-connections="true"` | No equivalent. HikariCP evicts the single broken connection. Add a `TODO` comment; do not emulate it. |
| `connection-validation-method` + `validation-classname` | Replace with `spring.datasource.hikari.connection-test-query`, or leave it out — HikariCP uses `Connection.isValid()` by default, which is better. |
| `statement-cache-size` | Move it to the JDBC driver properties (e.g. `spring.datasource.hikari.data-source-properties.cachePrepStmts`). |
| `pool-resize-quantity`, `steady-pool-size` | `steady-pool-size` → `minimum-idle`. `pool-resize-quantity` has no equivalent; drop it. |

## MultiTenancyRequirement

One `spring.datasource.<name>.*` block per tenant, one `DataSource` bean per block, and one
`AbstractRoutingDataSource` in front of them.

```java
@Configuration
class TenantDataSourceConfig {

    @Bean
    DataSource routingDataSource(
            @Qualifier("tenantADataSource") DataSource tenantA,
            @Qualifier("tenantBDataSource") DataSource tenantB) {
        AbstractRoutingDataSource routing = new AbstractRoutingDataSource() {
            @Override
            protected Object determineCurrentLookupKey() {
                return TenantContext.currentTenant();
            }
        };
        routing.setTargetDataSources(Map.of("tenantA", tenantA, "tenantB", tenantB));
        routing.setDefaultTargetDataSource(tenantA);
        return routing;
    }
}
```

- Set the tenant from `tenant_identifier_source`: `HEADER` → a `Filter` or `HandlerInterceptor` reading the
  header; `URL_PATH` → a path variable; `PRINCIPAL` → the authenticated user's claim.
- `THREADLOCAL` needs care with virtual threads and reactive code. Prefer a request-scoped bean.
- **Budget the pools together.** Apply the sizing formula to the sum of all tenant pools, not to each pool
  in isolation. Three tenant pools of 10 on 4 instances is 120 connections.
- Several unrelated datasources are not multi-tenancy. Without a `MultiTenancyRequirement` in the spec,
  generate plain named `DataSource` beans with `@Qualifier`, not a routing datasource.

## Checklist

- [ ] No `javax.naming` / `jakarta.naming` import remains, and no `lookup(` call remains
- [ ] `maximum-pool-size` is the per-instance number, or carries a `TODO` naming the missing inputs
- [ ] Every vendor setting with no equivalent is either mapped or noted in a comment
- [ ] The vendor resource file and the `<resource-ref>` block are deleted
