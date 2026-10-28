# Spring Boot 4 Translation: EJB Remote Interfaces

Map **EJB remote interface artifacts** (`@Remote`, `@RemoteHome`, `EJBObject`, `EJBHome`, `EJBLocalObject`, `EJBLocalHome`) detected during extraction to Spring Boot 4 equivalents. For the core translation guide, see [../SKILL.md](../SKILL.md).

Load this module when the story's `technical-context.yaml` contains any of:
- `migration_considerations` includes `"Remote EJB Interface"`, `"Remote EJB Home"`, `"EJB 2.x Remote Interface"`, or `"EJB 2.x Remote Home"`
- A `ServiceOperation` with `remote_interface: true`
- An `IntegrationPoint` with `type: RMI_EJB`
- `ServerMigrationRequirement.vendor_dependencies` referencing `javax.ejb.Remote`, `javax.ejb.EJBObject`, or `javax.ejb.EJBHome`

---

## Decision: REST vs Messaging

Before generating code, determine how the remote interface is consumed:

| If callers are... | Replace with |
|---|---|
| HTTP clients (web apps, microservices over network) | `@RestController` exposing the same operations |
| Internal JVM callers within the same application | `@Service` interface (no network transport needed) |
| Async/fire-and-forget callers | `@JmsListener` + `JmsClient` (see [translation-spring-boot-4-messaging.md](translation-spring-boot-4-messaging.md)) |
| Scheduled or batch consumers | `@Scheduled` or Spring Batch step |

When the spec does not specify caller type, **default to REST** (`@RestController`) and add a note in the generated code.

---

## Pattern 1: `@Remote` Interface → `@RestController` + `@Service`

### Source (EJB)
```java
@Remote
public interface OrderService {
    Order findById(Long id);
    Order placeOrder(OrderRequest request);
}

@Stateless
@Remote(OrderService.class)
public class OrderServiceBean implements OrderService {
    @PersistenceContext
    private EntityManager em;

    public Order findById(Long id) { return em.find(Order.class, id); }
    public Order placeOrder(OrderRequest request) { /* ... */ return order; }
}
```

### Target (Spring Boot 4)

**Service interface** (replaces the `@Remote` interface contract):
```java
public interface OrderService {
    Order findById(Long id);
    Order placeOrder(OrderRequest request);
}
```

**Service implementation** (replaces `@Stateless` bean):
```java
@Service
@Transactional(readOnly = true)
public class OrderServiceImpl implements OrderService {

    private final OrderRepository orderRepository;

    public OrderServiceImpl(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Override
    public Order findById(Long id) {
        return orderRepository.findById(id)
                .orElseThrow(() -> new EntityNotFoundException("Order not found: " + id));
    }

    @Override
    @Transactional
    public Order placeOrder(OrderRequest request) {
        Order order = new Order(request);
        return orderRepository.save(order);
    }
}
```

**REST controller** (exposes the formerly-remote interface over HTTP):
```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @GetMapping("/{id}")
    public ResponseEntity<Order> findById(@PathVariable Long id) {
        return ResponseEntity.ok(orderService.findById(id));
    }

    @PostMapping
    public ResponseEntity<Order> placeOrder(@RequestBody @Valid OrderRequest request) {
        Order order = orderService.placeOrder(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(order);
    }
}
```

**Extraction notes:**
- Each method on the `@Remote` interface maps 1:1 to a controller method + service method
- `throws RemoteException` is removed (network errors are now HTTP 5xx responses)
- `@Transactional(readOnly = true)` on the class; override with `@Transactional` on mutating methods

---

## Pattern 2: `@RemoteHome` + `EJBHome` → Remove Entirely

EJB home interfaces (`EJBHome`, `EJBLocalHome`) are factory objects that have **no Spring Boot equivalent**. Spring Boot uses constructor injection — the container manages lifecycle.

### Source (EJB 2.x)
```java
public interface OrderServiceHome extends EJBHome {
    OrderServiceRemote create() throws CreateException, RemoteException;
    OrderServiceRemote findByPrimaryKey(Long pk) throws FinderException, RemoteException;
}
```

### Target (Spring Boot 4)

| EJB Home method | Spring Boot replacement |
|---|---|
| `create()` | Constructor injection via `@Autowired` / `@Service` |
| `findByPrimaryKey(pk)` | `repository.findById(pk)` (Spring Data JPA) |
| `remove()` | `repository.delete(entity)` |

**Delete** the home interface class — it has no direct replacement. Callers that used `home.create()` simply autowire `OrderService` directly:

```java
// Before (EJB client)
InitialContext ctx = new InitialContext();
OrderServiceHome home = (OrderServiceHome) ctx.lookup("java:comp/env/ejb/OrderService");
OrderServiceRemote svc = home.create();
svc.placeOrder(request);

// After (Spring Boot)
@Autowired
private OrderService orderService;  // injected by Spring — no lookup needed

orderService.placeOrder(request);
```

---

## Pattern 3: `EJBObject` Interface → `@RestController` Methods

### Source (EJB 2.x)
```java
public interface OrderServiceRemote extends EJBObject {
    Order findById(Long id) throws RemoteException;
    Order placeOrder(OrderRequest request) throws RemoteException;
}
```

### Target (Spring Boot 4)

Treat exactly as Pattern 1 (`@Remote` interface):
1. Extract all declared methods
2. Create a plain Java service interface (no `extends EJBObject`)
3. Implement with `@Service`
4. Expose via `@RestController`
5. Remove all `throws RemoteException` declarations

---

## Pattern 4: `EJBLocalObject` / `EJBLocalHome` → `@Service` Interface Only

Local EJB interfaces communicate within the same JVM — no network transport was involved. **Do not create a REST controller** for these.

### Source (EJB 2.x local)
```java
public interface OrderServiceLocal extends EJBLocalObject {
    Order findById(Long id);
}

public interface OrderServiceLocalHome extends EJBLocalHome {
    OrderServiceLocal create() throws CreateException;
}
```

### Target (Spring Boot 4)

Replace with a plain Spring service interface:
```java
public interface OrderService {
    Order findById(Long id);
}

@Service
public class OrderServiceImpl implements OrderService {
    // ...
}
```

Callers that previously looked up the local home now use `@Autowired OrderService`.

---

## Removing `javax.ejb` Imports

All `javax.ejb.*` imports are replaced. Use this table to find the Spring Boot 4 equivalent:

| Legacy import | Spring Boot 4 replacement |
|---|---|
| `jakarta.ejb.Remote` | Remove — use `@RestController` for network exposure |
| `jakarta.ejb.RemoteHome` | Remove entirely |
| `jakarta.ejb.EJBObject` | Remove — replace interface `extends` with plain Java interface |
| `jakarta.ejb.EJBHome` | Remove entirely |
| `jakarta.ejb.EJBLocalObject` | Remove — replace with plain Java interface |
| `jakarta.ejb.EJBLocalHome` | Remove entirely |
| `jakarta.ejb.CreateException` | Remove — Spring throws `DataIntegrityViolationException` or custom exceptions |
| `jakarta.ejb.FinderException` | Remove — Spring Data throws `EmptyResultDataAccessException` or use `Optional` |
| `java.rmi.RemoteException` | Remove — map to HTTP 500 or custom `@ResponseStatus` exception |

> **Note:** EJB annotations migrated from `javax.ejb.*` to `jakarta.ejb.*` in Jakarta EE 9. You may encounter either prefix in the source; treat them identically.

---

## `application.yml` — No JNDI Configuration Needed

EJB remote lookup via `InitialContext` is fully replaced by Spring dependency injection. Remove all JNDI entries for EJB remote references:

```yaml
# REMOVE any entries like:
# java:comp/env/ejb/OrderService
# java:global/app/OrderServiceBean!com.example.OrderService

# Spring Boot 4 — no EJB JNDI configuration required
# Services are injected via @Autowired / constructor injection
```

---

## POM Dependencies

EJB remote interfaces require **no additional Spring Boot starter**. Remove any EJB API dependencies:

```xml
<!-- REMOVE -->
<dependency>
    <groupId>jakarta.ejb</groupId>
    <artifactId>jakarta.ejb-api</artifactId>
</dependency>

<!-- REMOVE (app server provided scope) -->
<dependency>
    <groupId>org.jboss.spec.javax.ejb</groupId>
    <artifactId>jboss-ejb-api_3.2_spec</artifactId>
    <scope>provided</scope>
</dependency>
```

The Spring Boot starters already on the classpath (`spring-boot-starter-web`, `spring-boot-starter-data-jpa`) provide all needed replacements.

---

## Schema Entity Summary

| Source Pattern | Detected Schema Entity | Spring Boot 4 Target |
|---|---|---|
| `@Remote` interface / `@Remote(X.class)` on bean | `ServiceOperation` with `remote_interface: true`, `IntegrationPoint (type: RMI_EJB)` | `@RestController` + `@Service` |
| `@RemoteHome(X.class)` on bean, `extends EJBHome` | `ServerMigrationRequirement.vendor_dependencies` | Delete home interface; use constructor injection |
| `extends EJBObject` | `ServiceOperation` with `remote_interface: true` | `@RestController` + `@Service` |
| `extends EJBLocalObject` | `ServiceOperation` | `@Service` interface only (no controller) |
| `extends EJBLocalHome` | `ServerMigrationRequirement.vendor_dependencies` | Delete; use constructor injection |
