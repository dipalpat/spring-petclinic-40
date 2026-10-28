# EJB Patterns

This document defines detection patterns for Enterprise JavaBeans (EJB) applications. These patterns help identify EJB session beans, message-driven beans, timers, interceptors, and other components that need modernization to Spring Boot.

## Table of Contents

- [Session Bean Detection](#session-bean-detection)
- [Remote Interface Detection](#remote-interface-detection)
- [Message-Driven Beans](#message-driven-beans)
- [Persistence Context](#persistence-context)
- [EJB Timer Service](#ejb-timer-service)
- [EJB Interceptors](#ejb-interceptors)
- [EJB Security](#ejb-security)
- [EJB Transaction Management](#ejb-transaction-management)
- [EJB Resource Injection](#ejb-resource-injection)
- [ejb-jar.xml Descriptor](#ejb-jarxml-descriptor)

---

## Session Bean Detection

### @Stateless Annotation

| Pattern | `@Stateless` |
|---------|--------------|
| **File Types** | `*.java` |
| **Indicates** | Stateless session bean (business service) |
| **Schema Entity** | `ServiceOperation` |

**Example:**
```java
@Stateless
@Local(OrderService.class)
public class OrderServiceBean implements OrderService {
    
    @PersistenceContext
    private EntityManager em;
    
    public Order findById(Long id) {
        return em.find(Order.class, id);
    }
}
```

**Extraction Notes:**
- Map to `@Service` in Spring Boot
- Extract `@Local` interfaces for service contracts; extract `@Remote` interfaces using the [Remote Interface Detection](#remote-interface-detection) patterns below
- Public methods become `ServiceOperation` entries

---

### @Stateful Annotation

| Pattern | `@Stateful` |
|---------|-------------|
| **File Types** | `*.java` |
| **Indicates** | Stateful session bean (conversational state) |
| **Schema Entity** | `ServiceOperation` with migration notes |

**Example:**
```java
@Stateful
public class ShoppingCartBean implements ShoppingCart {
    private List<CartItem> items = new ArrayList<>();
    
    public void addItem(Product product, int quantity) {
        items.add(new CartItem(product, quantity));
    }
    
    @Remove
    public Order checkout() {
        // Process order and cleanup
    }
}
```

**Extraction Notes:**
- Flag for architectural decision: convert to stateless with external session storage or `@SessionScope` bean
- When `@Stateful` is detected, ensure `ModernizationRequirements.migration_considerations` includes `"Session State"` (conversational state → externalize)
- Extract `@Remove` methods as lifecycle markers
- Document state management requirements

---

### @Singleton Annotation

| Pattern | `@Singleton` |
|---------|--------------|
| **File Types** | `*.java` |
| **Indicates** | Singleton session bean |
| **Schema Entity** | `ServiceOperation` (maps to default Spring singleton scope) |

**Example:**
```java
@Singleton
@Startup
public class ApplicationConfigBean {
    
    @PostConstruct
    public void init() {
        // Initialization logic
    }
}
```

**Extraction Notes:**
- Maps naturally to Spring `@Service` (singleton by default)
- Extract `@Startup` → Spring `@PostConstruct` or `ApplicationRunner`
- Extract `@Lock` annotations for concurrency requirements

---

## Remote Interface Detection

These patterns identify EJB remote interfaces and their home objects. They indicate **distributed RMI-IIOP communication** that must be replaced with REST endpoints, Spring Integration, or messaging during modernization. Each detected remote interface is a strong signal that inter-application coupling exists.

### @Remote Annotation

| Pattern | `@Remote` |
|---------|-----------|
| **File Types** | `*.java` |
| **Indicates** | EJB remote business interface — exposed over RMI-IIOP to remote clients |
| **Schema Entity** | `ServiceOperation` with `remote_interface: true`, `IntegrationPoint (type: RMI_EJB)`, `ServerMigrationRequirement.vendor_dependencies` |

**Example:**
```java
@Remote
public interface OrderService {
    Order findById(Long id);
    Order placeOrder(OrderRequest request);
}

@Stateless
@Remote(OrderService.class)
public class OrderServiceBean implements OrderService {
    // ...
}
```

**Extraction Notes:**
- When `@Remote` is on the **interface**: mark all its methods as `ServiceOperation` entries with `remote_interface: true`
- When `@Remote(SomeInterface.class)` is on the **bean class**: resolve `SomeInterface`, extract its methods as `ServiceOperation` entries with `remote_interface: true`
- Add an `IntegrationPoint` with `type: RMI_EJB`, `direction: OUTBOUND` for each remote interface (it is a coupling point visible to remote callers)
- Add to `ModernizationRequirements.migration_considerations`: `"Remote EJB Interface"` (RMI-IIOP transport → replace with REST or messaging)
- Add to `ServerMigrationRequirement.vendor_dependencies`:
  - `package_pattern: "javax.ejb.Remote"`, `replacement: "Remove — expose via @RestController (REST) or @JmsListener (messaging)"`

---

### @RemoteHome Annotation

| Pattern | `@RemoteHome\s*\(` |
|---------|---------------------|
| **File Types** | `*.java` |
| **Indicates** | EJB 2.x-style remote home interface — used by clients to create/find EJB instances over RMI-IIOP |
| **Schema Entity** | `ServerMigrationRequirement.vendor_dependencies`, `migration_considerations` |

**Example:**
```java
@RemoteHome(OrderServiceHome.class)
@Stateless
public class OrderServiceBean implements OrderServiceRemote {
    // ...
}

public interface OrderServiceHome extends EJBHome {
    OrderServiceRemote create() throws CreateException, RemoteException;
}
```

**Extraction Notes:**
- Extract the home interface class referenced in `@RemoteHome(...)` → document as a legacy coupling point
- EJB 2.x home interfaces (`EJBHome`) have **no Spring Boot equivalent**; they are replaced entirely by Spring's dependency injection and REST/messaging
- Add `"Remote EJB Home"` to `ModernizationRequirements.migration_considerations`
- Add to `ServerMigrationRequirement.vendor_dependencies`:
  - `package_pattern: "javax.ejb.RemoteHome"`, `usage_description: "EJB 2.x remote home interface"`, `replacement: "Remove — replace EJB home/create pattern with Spring @Service and REST controller"`

---

### EJBObject Interface

| Pattern | `extends EJBObject\|implements EJBObject` |
|---------|-------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | EJB 2.x remote component interface — the client-side stub for remote EJB calls |
| **Schema Entity** | `ServerMigrationRequirement.vendor_dependencies`, `migration_considerations` |

**Example:**
```java
public interface OrderServiceRemote extends EJBObject {
    Order findById(Long id) throws RemoteException;
    Order placeOrder(OrderRequest request) throws RemoteException;
}
```

**Extraction Notes:**
- Extract all methods declared on the `EJBObject` subinterface → create `ServiceOperation` entries with `remote_interface: true` and note each `throws RemoteException` as a legacy remote-call error-handling concern
- `EJBObject` is an EJB 2.x artifact; it is superseded by `@Remote` annotations in EJB 3.x
- Add `"EJB 2.x Remote Interface"` to `ModernizationRequirements.migration_considerations`
- Add to `ServerMigrationRequirement.vendor_dependencies`:
  - `package_pattern: "javax.ejb.EJBObject"`, `usage_description: "EJB 2.x remote component interface"`, `replacement: "Remove — replace with Spring @RestController methods (REST) or @Service interface (local)"`

---

### EJBHome Interface

| Pattern | `extends EJBHome\|implements EJBHome` |
|---------|---------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | EJB 2.x remote home interface — factory used by remote clients to create/find/remove EJB instances |
| **Schema Entity** | `ServerMigrationRequirement.vendor_dependencies`, `migration_considerations` |

**Example:**
```java
public interface OrderServiceHome extends EJBHome {
    OrderServiceRemote create() throws CreateException, RemoteException;
    OrderServiceRemote findByPrimaryKey(Long pk) throws FinderException, RemoteException;
}
```

**Extraction Notes:**
- `create()` methods → document as factory pattern; in Spring Boot this is replaced by `@Service` autowiring
- `findByPrimaryKey()` methods → map to Spring Data repository `findById()`
- `remove()` methods → map to repository `delete()`
- Add `"EJB 2.x Remote Home"` to `ModernizationRequirements.migration_considerations`
- Add to `ServerMigrationRequirement.vendor_dependencies`:
  - `package_pattern: "javax.ejb.EJBHome"`, `usage_description: "EJB 2.x remote home (factory)"`, `replacement: "Remove — Spring Boot uses constructor injection; expose create/find via @RestController or repository"`
- Also check for `EJBLocalHome` (same pattern, local variant) — same extraction notes apply but scope is intra-JVM

---

### EJBLocalObject / EJBLocalHome Interfaces

| Pattern | `extends EJBLocalObject\|extends EJBLocalHome\|implements EJBLocalObject\|implements EJBLocalHome` |
|---------|------------------------------------------------------------------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | EJB 2.x local component/home interface — intra-JVM variant of remote EJBObject/EJBHome |
| **Schema Entity** | `ServerMigrationRequirement.vendor_dependencies` |

**Extraction Notes:**
- Same extraction logic as `EJBObject` / `EJBHome` above, but no `RemoteException` — coupling is local to the JVM
- Add to `ServerMigrationRequirement.vendor_dependencies`:
  - `package_pattern: "javax.ejb.EJBLocalObject\|javax.ejb.EJBLocalHome"`, `replacement: "Remove — replace with Spring @Service interface and constructor injection"`

---

## Message-Driven Beans

### @MessageDriven Annotation

| Pattern | `@MessageDriven` |
|---------|------------------|
| **File Types** | `*.java` |
| **Indicates** | JMS message consumer |
| **Schema Entity** | `MessagingRequirement` with `direction = CONSUMER` |

**Example:**
```java
@MessageDriven(activationConfig = {
    @ActivationConfigProperty(propertyName = "destinationType", propertyValue = "javax.jms.Queue"),
    @ActivationConfigProperty(propertyName = "destination", propertyValue = "java:jboss/jms/queue/OrderQueue")
})
public class OrderMessageBean implements MessageListener {
    
    @Override
    public void onMessage(Message message) {
        // Process message
    }
}
```

**Extraction Notes:**
- Extract `destination` from activation config → `destination_name`
- Extract `destinationType` → `destination_type`
- Maps to `@JmsListener` or Spring Cloud Stream in Spring Boot

---

## Persistence Context

### @PersistenceContext Annotation

| Pattern | `@PersistenceContext` |
|---------|----------------------|
| **File Types** | `*.java` |
| **Indicates** | JPA EntityManager injection |
| **Schema Entity** | `TransactionRequirement.transaction_manager_type = JPA` |

**Example:**
```java
@PersistenceContext(unitName = "ordersPU")
private EntityManager entityManager;
```

**Extraction Notes:**
- Extract `unitName` to identify persistence unit
- Look for `persistence.xml` for datasource configuration
- When entity classes are used with `@PersistenceContext`, extract their **structure** (fields and relationships) using the [JPA persistence patterns](jpa-persistence.md). Always apply JPA persistence patterns when EJB is in use so entity properties are fully populated.

---

### persistence.xml File

| Pattern | `persistence.xml` (filename) |
|---------|-------------------------------|
| **File Types** | `persistence.xml` |
| **Indicates** | JPA configuration |
| **Schema Entity** | `ExternalResourceRequirement`, `DataStructure` |

**Example:**
```xml
<persistence-unit name="ordersPU" transaction-type="JTA">
    <jta-data-source>java:jboss/datasources/OrdersDS</jta-data-source>
    <class>com.example.model.Order</class>
    <properties>
        <property name="hibernate.dialect" value="org.hibernate.dialect.MySQL8Dialect"/>
    </properties>
</persistence-unit>
```

**Extraction Notes:**
- Extract `jta-data-source` or `non-jta-data-source` → `ExternalResourceRequirement`
- Extract `<class>` entries → `DataStructure` references
- When entity classes are listed in `persistence.xml`, extract their **structure** (fields and relationships) using the [JPA persistence patterns](jpa-persistence.md). Always apply JPA persistence patterns when EJB is in use so entity properties are fully populated.
- Extract properties for Spring Boot `spring.jpa.*` configuration

---

## EJB Timer Service

### @Schedule Annotation

| Pattern | `@Schedule\s*\(` |
|---------|------------------|
| **File Types** | `*.java` |
| **Indicates** | EJB timer-based scheduled task |
| **Schema Entity** | `ServiceTask` |

**Example:**
```java
@Singleton
public class ReportScheduler {
    
    @Schedule(hour = "2", minute = "0", persistent = false)
    public void generateNightlyReport() {
        // Generate report
    }
}
```

**Extraction Notes:**
- Convert EJB `@Schedule` format to Spring `@Scheduled` cron expression
- Extract `persistent` flag for clustering considerations

---

### TimerService Usage

| Pattern | `TimerService\|@Timeout` |
|---------|--------------------------|
| **File Types** | `*.java` |
| **Indicates** | Programmatic EJB timer |
| **Schema Entity** | `ServiceTask` |

**Example:**
```java
@Resource
private TimerService timerService;

public void scheduleProcessing(Date when) {
    timerService.createTimer(when, "processOrder");
}

@Timeout
public void handleTimeout(Timer timer) {
    // Handle timer expiration
}
```

---

## EJB Interceptors

### @Interceptors Annotation

| Pattern | `@Interceptors` |
|---------|-----------------|
| **File Types** | `*.java` |
| **Indicates** | Cross-cutting concerns (AOP-like) |
| **Schema Entity** | Migration note for Spring AOP `@Aspect` |

**Example:**
```java
@Stateless
@Interceptors({LoggingInterceptor.class, SecurityInterceptor.class})
public class OrderServiceBean {
    // ...
}
```

**Extraction Notes:**
- Map to Spring AOP `@Aspect` and `@Around` advice
- Extract interceptor logic for migration

---

### @AroundInvoke Annotation

| Pattern | `@AroundInvoke` |
|---------|-----------------|
| **File Types** | `*.java` |
| **Indicates** | Interceptor method |
| **Schema Entity** | Migration note for Spring AOP |

---

## EJB Security

### @RolesAllowed Annotation

| Pattern | `@RolesAllowed\s*\(` |
|---------|----------------------|
| **File Types** | `*.java` |
| **Indicates** | Method/class-level authorization |
| **Schema Entity** | `SecurityRequirement`, `ServiceOperation.authorization` |

**Example:**
```java
@RolesAllowed({"ADMIN", "MANAGER"})
public void approveOrder(Long orderId) {
    // ...
}
```

**Extraction Notes:**
- Extract role names → `SecurityRequirement.roles`
- Maps to Spring Security `@PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")`

---

### @PermitAll / @DenyAll Annotations

| Pattern | `@PermitAll\|@DenyAll` |
|---------|------------------------|
| **File Types** | `*.java` |
| **Indicates** | Security policy |
| **Schema Entity** | `SecurityRequirement.protected_endpoints` |

---

## EJB Transaction Management

### @TransactionAttribute Annotation

| Pattern | `@TransactionAttribute\s*\(` |
|---------|------------------------------|
| **File Types** | `*.java` |
| **Indicates** | Container-managed transaction settings |
| **Schema Entity** | `ServiceOperation.transaction_propagation` |

**Example:**
```java
@TransactionAttribute(TransactionAttributeType.REQUIRES_NEW)
public void processPayment(Payment payment) {
    // ...
}
```

**Extraction Notes:**
- Map EJB transaction types to Spring propagation:
  - `REQUIRED` → `Propagation.REQUIRED`
  - `REQUIRES_NEW` → `Propagation.REQUIRES_NEW`
  - `MANDATORY` → `Propagation.MANDATORY`
  - `NOT_SUPPORTED` → `Propagation.NOT_SUPPORTED`
  - `SUPPORTS` → `Propagation.SUPPORTS`
  - `NEVER` → `Propagation.NEVER`

---

## EJB Resource Injection

### @Resource Annotation

| Pattern | `@Resource\s*\(` |
|---------|------------------|
| **File Types** | `*.java` |
| **Indicates** | JNDI resource injection |
| **Schema Entity** | `ExternalResourceRequirement` |

**Example:**
```java
@Resource(lookup = "java:jboss/datasources/OrdersDS")
private DataSource dataSource;

@Resource(lookup = "java:jboss/jms/queue/NotificationQueue")
private Queue notificationQueue;
```

**Extraction Notes:**
- Extract `lookup` or `name` → `legacy_jndi_name`
- Infer resource type from variable type
- **For DataSource**: When the injected resource is a `DataSource`, look up connection pool configuration in server descriptors (JBoss, WebLogic, WebSphere, etc.—see [app-servers.md](app-servers.md)). Populate `ExternalResourceRequirement.connection_pool` and Spring Boot HikariCP properties in `spring_boot_config.properties` (e.g. `spring.datasource.hikari.maximum-pool-size`, `minimum-idle`, `connection-timeout`, `idle-timeout`, `max-lifetime`, `auto-commit`).

---

### @EJB Annotation

| Pattern | `@EJB` |
|---------|--------|
| **File Types** | `*.java` |
| **Indicates** | EJB injection (inter-bean dependency) |
| **Schema Entity** | `ServiceOperation.dependencies` |

**Example:**
```java
@EJB
private InventoryService inventoryService;
```

**Extraction Notes:**
- Maps to Spring `@Autowired`
- Track dependencies for service layer design

---

## ejb-jar.xml Descriptor

| Pattern | `ejb-jar.xml` (filename) |
|---------|--------------------------|
| **File Types** | `ejb-jar.xml` |
| **Indicates** | EJB deployment descriptor |
| **Schema Entity** | Multiple entities |

**Extraction Notes:**
- Parse for additional EJB configurations not in annotations
- Extract security role mappings
- Extract transaction settings
- Extract interceptor bindings
