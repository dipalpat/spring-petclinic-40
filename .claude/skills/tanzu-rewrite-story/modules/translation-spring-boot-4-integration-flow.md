# Spring Boot 4 Translation: IntegrationFlowRequirement, ConnectorRequirement, TransformationRequirement, ErrorHandlingRequirement

Map the four MuleSoft entities to plain Spring Boot components. For the core translation guide, see
[../SKILL.md](../SKILL.md). For the service bindings, see
[translation-spring-boot-4-cloud-foundry.md](translation-spring-boot-4-cloud-foundry.md).

## The rule

A Mule flow is not a unit of code. It is XML that wires a listener, a connector, a transform and an error
handler together. Translate each of those four parts into the Spring component that already does that job.
Do not build a flow engine.

Reach for Spring Integration only when the spec records a `flow_type` that a controller cannot express —
`SCATTER_GATHER`, or an `EVENT_DRIVEN` flow with a poller. A `HTTP_REST` flow is a `@RestController`.

| Entity | Target |
|--------|--------|
| `IntegrationFlowRequirement` | `@RestController`, or a Spring Integration flow |
| `ConnectorRequirement` | `JdbcTemplate`/Spring Data, `JmsTemplate`, `JavaMailSender`, Spring Integration adapter |
| `TransformationRequirement` | `ObjectMapper`, or a hand-written mapper class |
| `ErrorHandlingRequirement` | `@RestControllerAdvice` with `@ExceptionHandler` |

## IntegrationFlowRequirement

| Field | Use it for |
|-------|------------|
| `name` | The method name. `getOrderFlow` becomes `getOrder`. |
| `flow_type` | Picks the target, see the table below. |
| `http_method` | The mapping annotation: `@GetMapping`, `@PostMapping`, and so on. |
| `endpoint_path` | The `@RequestMapping` value. Keep the path unchanged so callers do not break. |

| `flow_type` | Target |
|-------------|--------|
| `HTTP_REST` | `@RestController` method |
| `SUB_FLOW` | A `@Service` method called by the controller. A sub-flow is not an endpoint. |
| `ROUTING` | A method with a `switch`, or a `Map<String, Handler>`. Not a routing framework. |
| `EVENT_DRIVEN` | `@JmsListener`, or a Spring Cloud Stream `Consumer<T>` |
| `SCATTER_GATHER` | `CompletableFuture.allOf(...)`, or a Spring Integration scatter-gather flow |
| `BATCH` | Spring Batch. See [translation-spring-boot-4-batch.md](translation-spring-boot-4-batch.md). |

```java
@RestController
@RequestMapping("/api/orders")
class OrderController {

    private final OrderService orders;

    OrderController(OrderService orders) {
        this.orders = orders;
    }

    @GetMapping("/{id}")
    ResponseEntity<OrderResponse> getOrder(@PathVariable String id) {
        return ResponseEntity.ok(orders.findById(id));
    }
}
```

A Mule `<set-payload>` at the end of a flow is the return value. It is not a separate step.

## ConnectorRequirement

| `connector_type` | Target | Dependency |
|------------------|--------|------------|
| `DATABASE` | `JdbcClient`, or a Spring Data repository | `spring-boot-starter-data-jpa` or `-jdbc` |
| `JMS` | `@JmsListener` and `JmsTemplate` | `spring-boot-starter-artemis` |
| `KAFKA` | Spring Cloud Stream `Function`/`Consumer`/`Supplier` | `spring-cloud-stream-binder-kafka` |
| `MAIL` | `JavaMailSender`. See [translation-spring-boot-4-mail.md](translation-spring-boot-4-mail.md). | `spring-boot-starter-mail` |
| `FTP`, `SFTP` | Spring Integration inbound or outbound adapter | `spring-integration-ftp` or `-sftp` |
| `HTTP_CLIENT` | `RestClient`. See [translation-spring-boot-4-integration.md](translation-spring-boot-4-integration.md). | `spring-boot-starter-web` |
| `CUSTOM` | Report it. Do not guess a replacement. | — |

`operations` says which methods the class needs. `[SELECT, INSERT]` means a read method and a write method,
not a full CRUD repository.

`config_ref` is the name of a Mule global element. Several flows share one. Create one bean, not one per
flow. Never carry over a host, user or password found in the Mule XML: read every one from the environment.

```java
@Repository
class OrderRepository {

    private final JdbcClient jdbc;

    OrderRepository(JdbcClient jdbc) {
        this.jdbc = jdbc;
    }

    Optional<Order> findById(String id) {
        return jdbc.sql("SELECT id, amount FROM orders WHERE id = :id")
                .param("id", id)
                .query(Order.class)
                .optional();
    }
}
```

A Mule expression such as `#[attributes.uriParams.id]` becomes a `@PathVariable`. `#[payload.field]` becomes
a field of the request record. Never build SQL by joining strings around a Mule expression.

## TransformationRequirement

DataWeave has no Spring equivalent, and it needs no runtime. Read the script, work out the shape of the
input and the output, then write both as records and one mapper.

| `transformation_type` | Target |
|-----------------------|--------|
| `DATA_WEAVE` | Two records and a mapper method. Use `stream().map(...)` for a DataWeave `map`. |
| `XSLT` | Keep the XSLT and run it with `javax.xml.transform`, or write a mapper. State which you chose. |
| `CUSTOM_MAPPER` | A mapper method. |
| `EXPRESSION` | Inline Java. A one-line expression does not need a class. |

`input_format` and `output_format` set the record types and the content type:

| Format | In the controller |
|--------|-------------------|
| `APPLICATION_JSON` | A record. Jackson handles it, no annotation needed. |
| `APPLICATION_XML` | A record plus `jackson-dataformat-xml`, and `consumes`/`produces` set to `application/xml`. |
| `TEXT_PLAIN` | `String`. |
| `MULTIPART` | `MultipartFile`. |

```java
@Component
class OrderMapper {

    OrderDto toDto(OrderRequest request) {
        return new OrderDto(
                request.orderId(),
                request.totalPrice(),
                LocalDate.now(),
                request.lineItems().stream()
                        .map(item -> new ItemDto(item.id(), item.quantity(),
                                item.unitPrice().multiply(BigDecimal.valueOf(item.quantity()))))
                        .toList());
    }
}
```

Two rules that DataWeave hides and Java does not:

- DataWeave `now()` makes the output change on every call. That breaks a byte-for-byte output test. Take the
  clock from an injected `Clock` bean so a test can fix it.
- DataWeave arithmetic on money is loose. Use `BigDecimal`, not `double`.

## ErrorHandlingRequirement

One `@RestControllerAdvice` per application, one `@ExceptionHandler` per `error_type`. Do not put a
`try`/`catch` in the controller to imitate the Mule `<try>` block.

| `error_strategy` | Target |
|------------------|--------|
| `PROPAGATE` | Return the error status from the handler. This is `<on-error-propagate>`. |
| `CONTINUE` | Return `200` with a fallback body. This is `<on-error-continue>`. |
| `RETRY` | `@Retryable` from Spring Retry, or the `RestClient` retry. State the attempt count and the backoff. |
| `FALLBACK` | Return a default value from the handler. |

`error_type` picks the exception class. `http_status_code` sets the status.

| Mule `error_type` | Exception |
|-------------------|-----------|
| `DB_ERROR` | `DataAccessException` |
| `HTTP:CONNECTIVITY`, `HTTP:TIMEOUT` | `ResourceAccessException` |
| `VALIDATION`, `EXPRESSION` | `MethodArgumentNotValidException` |
| `ANY` | `Exception`. Put it last: Spring picks the most specific handler. |

```java
@RestControllerAdvice
class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(DataAccessException.class)
    ResponseEntity<ErrorResponse> handleDatabase(DataAccessException ex) {
        log.error("Database operation failed", ex);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(new ErrorResponse("Database operation failed"));
    }

    @ExceptionHandler(Exception.class)
    ResponseEntity<ErrorResponse> handleAny(Exception ex) {
        log.error("Unexpected error", ex);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(new ErrorResponse("Internal server error"));
    }
}
```

A Mule error handler often returns the message from the exception to the caller. Do not. Log the exception
and return a fixed message, so the reply carries no stack trace and no SQL.

## Cloud Foundry notes

- A Mule `<logger>` is not monitoring. Map it to SLF4J and add `spring-boot-starter-actuator`. Every
  instance logs to the same stream, so include an identifier in the message when a request spans instances.
- A `<jms:listener>` runs on every instance. Set a Spring Cloud Stream consumer group, or a shared JMS
  subscription, or every instance handles the same message.
- A Mule poller (`FTP`, `SFTP`, `EVENT_DRIVEN` with a schedule) also runs on every instance. Take a
  distributed lock, or deploy the poller as its own single-instance application. Say which you chose.
- Mule reads and writes files on local disk. Local disk on the platform is empty after a restart. Move any
  file that must survive to object storage or a database.

## Checklist

- [ ] Every `endpoint_path` is unchanged, so existing callers still work
- [ ] No connector host, user or password from the Mule XML is in `application.yml`
- [ ] Every `config_ref` is one bean, not one bean per flow
- [ ] Money uses `BigDecimal`, and the clock is injected
- [ ] One `@RestControllerAdvice`, with the `Exception` handler last
- [ ] Every listener or poller states how it avoids running on all instances
- [ ] A `CUSTOM` connector is reported, not guessed
