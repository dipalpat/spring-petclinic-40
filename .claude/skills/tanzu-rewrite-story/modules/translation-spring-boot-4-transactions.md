# Spring Boot 4 Translation: TransactionRequirement

Map **TransactionRequirement** to `@Transactional`. When the spec reports XA, the pattern needs an
architectural change, not a property change. For the core translation guide, see [../SKILL.md](../SKILL.md).

## Patterns (no XA)

- `transaction_manager_type: JPA` or `JDBC` → `@Transactional` on the service method. Spring Boot
  auto-configures the manager; declare no `PlatformTransactionManager` bean.
- `default_propagation`, `default_isolation`, `default_timeout` → `@Transactional(propagation = ...,
  isolation = ..., timeout = ...)`. Set them only where they differ from the default.
- `rollback_rules` → `@Transactional(noRollbackFor = BusinessException.class)` for `rollback: false`, and
  `rollbackFor` for a checked exception that must roll back.
- `transaction_manager_type: CONTAINER_MANAGED` (EJB `@TransactionAttribute`) → `@Transactional` with the
  matching propagation. `REQUIRES_NEW` → `Propagation.REQUIRES_NEW`, `NOT_SUPPORTED` →
  `Propagation.NOT_SUPPORTED`, `MANDATORY` → `Propagation.MANDATORY`.
- Remove every `UserTransaction`, `begin()`, `commit()`, `rollback()` call. Declarative transactions replace
  them.

## XA: stop and check the scope first

When `xa_datasource_detected: true`, read `scope`:

| `scope` | What to do |
|---------|------------|
| `REQUIRES_USER_FEEDBACK` | **Stop.** Ask the user: *"Is XA / distributed transaction support critical across your application, or is it used only in specific components?"* Do not translate the pattern until the answer is `GLOBAL` or `LOCAL`. |
| `LOCAL` | Refactor only the named components. The rest of the application uses plain `@Transactional`. |
| `GLOBAL` | The application needs a redesign. Present both options below and let the user choose. Do not pick one silently. |

**Why XA cannot be carried over.** Two-phase commit needs a durable recovery log and a transaction
coordinator that outlives any single process. A Cloud Foundry instance has ephemeral local disk and is
replaced without warning on a rolling deploy, a `cf restage`, a crash restart, or an autoscaler scale-down.
The platform supplies no XA coordinator. An in-process coordinator writing its log to the container
filesystem loses in-doubt transactions on every recycle, which is worse than no XA at all.

### Option 1 — Transactional Outbox (one database, plus a message)

Use this when the XA transaction spans a database and a message broker.

```java
@Transactional
public void placeOrder(Order order) {
    orderRepository.save(order);
    outboxRepository.save(OutboxEvent.of("OrderPlaced", order.toPayload()));
}
```

The state change and the event are written in one local transaction. A separate poller publishes unsent
outbox rows and marks them sent. The publish is at-least-once, so the consumer must be idempotent.

- Add an `outbox` table with `id`, `type`, `payload`, `created_at`, `published_at`.
- Add a `@Scheduled` poller, or use a change-data-capture connector.
- Point out that the poller must be safe with several instances: use `SELECT ... FOR UPDATE SKIP LOCKED`, or
  run it as a separate single-instance application. Do not leave this to chance.

### Option 2 — Saga with compensating actions (more than one system)

Use this when the XA transaction spans two or more remote systems.

```java
public void reserveInventoryThenCharge(Order order) {
    inventoryClient.reserve(order.getItems());
    try {
        paymentClient.charge(order.getPaymentDetails());
    } catch (PaymentFailedException e) {
        inventoryClient.release(order.getItems());   // compensating action
        throw e;
    }
}
```

- Every forward step needs a compensating step.
- The compensation can itself fail. Log it, and surface the inconsistency for an operator; do not swallow it.
- Both the forward step and the compensation must be idempotent, because a retry can repeat either.

### What to say to the user

Consistency changes. XA gave atomic all-or-nothing across systems; the outbox and the saga give eventual
consistency with a visible window where the systems disagree. State this plainly, and name the window. Do not
present either option as a drop-in replacement.

## Checklist

- [ ] No `UserTransaction` and no `javax.transaction` / `jakarta.transaction` import remains
- [ ] No `JtaTransactionManager` bean and no XA datasource is generated
- [ ] When `scope` was `REQUIRES_USER_FEEDBACK`, the user was asked before any code was written
- [ ] The chosen alternative names its consistency trade-off in the summary to the user
- [ ] Any outbox poller or saga step is safe to run on more than one instance
