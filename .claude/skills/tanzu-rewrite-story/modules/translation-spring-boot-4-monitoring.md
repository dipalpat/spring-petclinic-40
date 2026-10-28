# Spring Boot 4 Translation: MonitoringRequirement

Map **MonitoringRequirement** to Spring Boot Actuator and Micrometer. For the core translation guide, see
[../SKILL.md](../SKILL.md).

## Patterns by `metrics_source`

| `metrics_source` | Target |
|------------------|--------|
| `JMX` — the application **reads** server MBeans | Delete the code. Server state is the platform's concern, not the application's. Do not reimplement it. |
| `JMX` — the application **registers** its own MBean | One Micrometer meter per MBean attribute. Keep the semantics, drop the MBean. |
| `VENDOR_API` (admin or management API) | Delete the code. Actuator health and info endpoints replace what is still meaningful. |
| `VENDOR_PROPERTIES` (`System.getProperty("glassfish.*")`) | Spring `Environment` and a normal application property. |
| `MANAGEMENT_API` | Same as `VENDOR_API`. |

## Configuration

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      probes:
        enabled: true
  metrics:
    tags:
      application: ${vcap.application.name:orders-app}
      cf_instance_index: ${CF_INSTANCE_INDEX:0}
      cf_instance_guid: ${CF_INSTANCE_GUID:local}
```

Dependencies: `spring-boot-starter-actuator`, plus `micrometer-registry-prometheus` when
`prometheus_export: true`.

## Per-instance tagging (mandatory when the spec sets it)

Every instance publishes its own copy of every meter. Without an instance tag, the backend merges them into
one series and the result is meaningless — a gauge shows whichever instance reported last, and a counter
undercounts. Cloud Foundry supplies two variables to every container:

- `CF_INSTANCE_INDEX` — the ordinal, stable for the life of the instance
- `CF_INSTANCE_GUID` — unique per instance, changes on every recycle

Use both: the index for dashboards, the GUID to tell a restarted instance from the one it replaced.

The `management.metrics.tags.*` properties above cover it. A programmatic equivalent:

```java
@Bean
MeterRegistryCustomizer<MeterRegistry> cfInstanceTags(Environment env) {
    return registry -> registry.config().commonTags(
            "cf_instance_index", env.getProperty("CF_INSTANCE_INDEX", "0"),
            "cf_instance_guid", env.getProperty("CF_INSTANCE_GUID", "local"));
}
```

## Replacing a registered MBean

```java
// Before: an MBean exposing an attribute
public interface ServerMonitorMBean {
    long getProcessedOrderCount();
}

// After: a Micrometer counter
@Service
class OrderService {

    private final Counter processedOrders;

    OrderService(MeterRegistry registry) {
        this.processedOrders = Counter.builder("orders.processed")
                .description("Orders processed by this instance")
                .register(registry);
    }

    void process(Order order) {
        // ...
        processedOrders.increment();
    }
}
```

- A read-only attribute becomes a `Gauge`. A monotonically increasing total becomes a `Counter`. A duration
  becomes a `Timer`. An MBean operation is not a metric — expose it as an endpoint or delete it.
- Name meters in the dotted Micrometer convention (`orders.processed`), not the MBean `ObjectName`.

## Replacing vendor system properties

```java
// Before
String version = System.getProperty("glassfish.version");

// After
@Component
class RequestLogger {

    private final Environment environment;

    RequestLogger(Environment environment) {
        this.environment = environment;
    }

    void log() {
        String version = environment.getProperty("app.version", "unknown");
    }
}
```

`com.sun.aas.instanceRoot` and `com.sun.aas.installRoot` are paths into the server installation. There is no
equivalent. Do not substitute a container path — report each use to the user.

## Centralized backend

Set when `requires_centralized_backend: true`: instance-local metrics disappear with the instance, so the
data must be scraped or pushed to a backend outside the application. Name the requirement in the summary;
choosing and provisioning the backend is a platform task, not something to generate.

## Checklist

- [ ] No `javax.management` import, no `ObjectName`, and no `MBeanServer` use remains
- [ ] No `org.glassfish` / `com.sun.enterprise` / `com.sun.appserv` import remains
- [ ] Every exported metric carries `CF_INSTANCE_INDEX` and `CF_INSTANCE_GUID` when the spec asked for it
- [ ] Health probes are enabled, so the platform can restart an unhealthy instance
- [ ] Every unmappable vendor property is reported, not silently replaced
