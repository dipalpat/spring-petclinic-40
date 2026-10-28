# MuleSoft Detection Patterns

This document covers detection patterns for legacy MuleSoft applications (Mule 3 / Mule 4 runtimes) migrating to Spring Boot 4, Spring Integration, and Spring Cloud Stream.

## Server Identification

| Pattern | File Types | Indicates | Schema Entity |
|---------|------------|-----------|---------------|
| `mule-config.xml`, `mule-artifact.json`, `mule-deploy.properties` | filename | MuleSoft application descriptor / configuration | `ModernizationRequirements.source_server = MULESOFT` |
| `<mule xmlns="http://www.mulesoft.org/schema/mule/` | `*.xml` | MuleSoft XML namespace in flow definitions | `ModernizationRequirements.source_server = MULESOFT` |
| `org\.mule\.`, `com\.mulesoft\.` | `*.java` | MuleSoft Java API / custom component dependency | `ServerMigrationRequirement.vendor_dependencies` |

---

## Category 21: MuleSoft Configuration Files & HTTP Flows

### 21a — HTTP Listener Flow (REST Endpoint)

| Pattern | `<http:listener-config`, `<http:listener`, `path=`, `method=` |
|---------|---------------------------------------------------------------|
| **File Types** | `mule-config.xml`, `*.xml` in `src/main/mule/` or `src/main/app/` |
| **Indicates** | Inbound HTTP listener endpoint exposed as a REST API |
| **Schema Entity** | `IntegrationFlowRequirement`, `ServiceEndpoint` (the listener path and method), `ServiceOperation` (the processors between the listener and the response) |

**Example:**
```xml
<mule xmlns="http://www.mulesoft.org/schema/mule/core"
      xmlns:http="http://www.mulesoft.org/schema/mule/http"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://www.mulesoft.org/schema/mule/core
                          http://www.mulesoft.org/schema/mule/core/current/mule.xsd
                          http://www.mulesoft.org/schema/mule/http
                          http://www.mulesoft.org/schema/mule/http/current/mule-http.xsd">

  <http:listener-config name="HTTP_Listener_config">
    <http:listener-connection host="0.0.0.0" port="8081"/>
  </http:listener-config>

  <flow name="getOrderFlow">
    <http:listener config-ref="HTTP_Listener_config" path="/api/orders" method="GET"/>
    <set-payload value='{"orderId": "12345", "status": "delivered"}'/>
  </flow>
</mule>
```

**DSL Schema Mapping:**
```yaml
IntegrationFlowRequirement:
  name: "getOrderFlow"
  flow_type: HTTP_REST
  http_method: GET
  endpoint_path: "/api/orders"
  server_type: MULESOFT
  spring_boot_alternative: "@RestController"
  migration_effort: LOW
```

**Spring Boot 4 Mapping:**
```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
  @GetMapping
  public ResponseEntity<Order> getOrder(@RequestParam String orderId) {
    Order order = new Order("12345", "delivered");
    return ResponseEntity.ok(order);
  }
}
```

**Extraction Notes:**
- A `<http:listener>` is an inbound endpoint. Extract `path` → `ServiceEndpoint.path`, `method` →
  `ServiceEndpoint.method`, and the payload media type → `ServiceEndpoint.response_type` (`JSON` for
  `output application/json`). Set `ServiceEndpoint.authentication: ANONYMOUS` unless the flow holds an
  authentication step or a policy.
- The processors between the listener and the response are the operation. Record one
  `ServiceOperation` per flow: `name` from the `flow name`, `type: QUERY` for a read and `COMMAND` for
  a write, `transactional: true` only when the flow holds a `<try>` around a write.
- Link the two: `ServiceEndpoint.operation` = that `ServiceOperation.name`.
- `#[attributes.uriParams.x]` is a path variable → `ServiceEndpoint.path_variables`.
  `#[attributes.queryParams.x]` is a query parameter → `ServiceEndpoint.query_parameters`.
- A `<flow>` with no inbound listener is a sub-flow. Record the `ServiceOperation` and no
  `ServiceEndpoint`.

**Migration Effort:** LOW

---

### 21b — Flow with Request/Response Transformation

| Pattern | `<flow name="...">`, `<http:listener method="POST"`, `<set-payload>` |
|---------|----------------------------------------------------------------------|
| **File Types** | `*.xml` |
| **Indicates** | Inbound HTTP flow with request handling and response payload generation |
| **Schema Entity** | `IntegrationFlowRequirement`, `ServiceEndpoint`, `ServiceOperation`, `DataStructure` (the request DTO and the response DTO) |

**Example:**
```xml
<flow name="transformOrderFlow">
  <http:listener config-ref="HTTP_Listener_config" path="/api/orders" method="POST"/>
  <logger message="Received order: #[payload]" level="INFO"/>
  <set-payload value='{"orderId": "generated_id", "status": "created"}'/>
</flow>
```

**DSL Schema Mapping:**
```yaml
IntegrationFlowRequirement:
  name: "transformOrderFlow"
  flow_type: HTTP_REST
  http_method: POST
  endpoint_path: "/api/orders"
  server_type: MULESOFT
  spring_boot_alternative: "@RestController + @PostMapping"
  migration_effort: MEDIUM
```

**Extraction Notes:**
- The request body and the response body are two structures, not one. Record a `DataStructure` with
  `persistence_type: DTO` for each. Take `fields` from the JSON keys in `<set-payload>`, in the
  DataWeave output, or from the `#[payload.x]` references the flow reads.
- Reference them from the endpoint: `ServiceEndpoint.request_body` and `ServiceEndpoint.response_body`.
- A field has no declared type in Mule XML. Use `String` unless the value is plainly numeric or
  boolean, and say which fields you inferred.

**Migration Effort:** MEDIUM

---

### 21c — Outbound HTTP Request (REST Client)

| Pattern | `<http:request`, `<http:request-config`, `<http:request-connection`, `url=` |
|---------|-----------------------------------------------------------------------------|
| **File Types** | `*.xml` |
| **Indicates** | Outbound call from a flow to an external HTTP or REST API |
| **Schema Entity** | `ConnectorRequirement`, `IntegrationPoint` (type `REST_API`, direction `OUTBOUND`) |

**Example:**
```xml
<http:request-config name="Supplier_Request_config">
  <http:request-connection host="supplier-a.internal" port="80"/>
</http:request-config>

<flow name="fetchSupplierQuoteFlow">
  <http:listener config-ref="HTTP_Listener_config" path="/api/quotes" method="GET"/>
  <http:request method="GET" url="http://supplier-a.internal/quote"/>
</flow>
```

**DSL Schema Mapping:**
```yaml
ConnectorRequirement:
  name: "Supplier_Request_config"
  connector_type: HTTP_CLIENT
  server_type: MULESOFT
  operations: [SEND]
  config_ref: "Supplier_Request_config"
  spring_boot_alternative: "@HttpExchange interface with RestClient"
  migration_effort: LOW
```

**Extraction Notes:**
- Record one `IntegrationPoint` per distinct target. Put the `url`, or the `host` and `path` joined,
  into `IntegrationPoint.endpoint`. Set `type: REST_API` and `direction: OUTBOUND`, and `protocol`
  from the scheme.
- An `<http:request>` is the outbound half of integration. Do not confuse it with `<http:listener>`,
  which is inbound and belongs to 21a.
- Record a `<reconnect>` or a `responseTimeout` into `IntegrationPoint.retry_policy` and
  `IntegrationPoint.timeout_ms`.
- Never copy a `username` or a `password` found on the connection. Record the field name only, and
  set `IntegrationPoint.authentication.credentials_source`.

**Migration Effort:** LOW

---

### 21d — Routing and Scatter-Gather

| Pattern | `<scatter-gather>`, `<choice>`, `<when`, `<otherwise>`, `<round-robin>`, `<first-successful>` |
|---------|-----------------------------------------------------------------------------------------------|
| **File Types** | `*.xml` |
| **Indicates** | A flow that splits into routes and joins the results, or picks one route |
| **Schema Entity** | `IntegrationFlowRequirement` (`flow_type: SCATTER_GATHER` or `ROUTING`) |

**Example:**
```xml
<flow name="scatterGatherFlow">
  <http:listener config-ref="HTTP_Listener_config" path="/api/quotes/aggregate" method="POST"/>
  <scatter-gather>
    <route>
      <http:request method="GET" url="http://supplier-a.internal/quote"/>
    </route>
    <route>
      <http:request method="GET" url="http://supplier-b.internal/quote"/>
    </route>
  </scatter-gather>
</flow>
```

**DSL Schema Mapping:**
```yaml
IntegrationFlowRequirement:
  name: "scatterGatherFlow"
  flow_type: SCATTER_GATHER
  http_method: POST
  endpoint_path: "/api/quotes/aggregate"
  server_type: MULESOFT
  spring_boot_alternative: "Parallel RestClient calls joined in a @RestController, or Spring Integration"
  migration_effort: HIGH
```

**Extraction Notes:**
- `<scatter-gather>` runs every `<route>` at the same time and joins the results. Use
  `flow_type: SCATTER_GATHER`. The target needs concurrent calls and a join step, not a chain.
- `<choice>`, `<round-robin>` and `<first-successful>` pick one route. Use `flow_type: ROUTING`.
  Record each `<when>` condition, because it becomes a branch in the target.
- Record every route. A route that holds an `<http:request>` also produces the entities in 21c.
- The joined result is a new structure. Record a `DataStructure` with `persistence_type: DTO` that
  holds one field per route.

**Migration Effort:** HIGH — a parallel split and join has no single Spring annotation

---

## Category 22: Connectors (Database, Mail, Messaging, FTP)

### 22a — Database Connector (SQL)

| Pattern | `<db:config>`, `<db:select>`, `<db:insert>`, `<db:update>`, `<db:delete>`, `<db:sql>` |
|---------|---------------------------------------------------------------------------------------|
| **File Types** | `*.xml` |
| **Indicates** | Relational database operations via Mule DB connector |
| **Schema Entity** | `ConnectorRequirement`, `IntegrationPoint` (type `DATABASE`, direction `OUTBOUND`), `DataStructure` (the row shape the SQL reads or writes) |

**Example:**
```xml
<db:config name="Database_Config">
  <db:mysql-connection host="db.example.com" port="3306" user="dbuser" password="dbpass" database="orders"/>
</db:config>

<flow name="queryOrderFromDB">
  <http:listener config-ref="HTTP_Listener_config" path="/api/orders/{id}" method="GET"/>
  <db:select config-ref="Database_Config">
    <db:sql>SELECT * FROM orders WHERE id = #[attributes.uriParams.id]</db:sql>
  </db:select>
</flow>
```

**DSL Schema Mapping:**
```yaml
ConnectorRequirement:
  name: "DatabaseConnector"
  connector_type: DATABASE
  database_type: MYSQL
  server_type: MULESOFT
  operations: [SELECT, INSERT, UPDATE, DELETE]
  spring_boot_alternative: "JdbcTemplate or Spring Data JPA"
  migration_effort: MEDIUM
```

**Extraction Notes:**
- A database is an external system. Record an `IntegrationPoint` with `type: DATABASE`,
  `direction: OUTBOUND`, `protocol: JDBC`, and the `host`, `port` and `database` joined into
  `endpoint`. Never copy the `user` or the `password`.
- The table in the `<db:sql>` is a data model. Record a `DataStructure` named after the table with
  `persistence_type: ENTITY`, `table_name` from the SQL, and one `Field` per named column. When the
  SQL is `SELECT *`, say that the column list is unknown rather than inventing fields.
- The columns in an `INSERT` or an `UPDATE` are the fields the target must be able to write.
- `#[attributes.uriParams.id]` inside `<db:sql>` binds a path variable to a query parameter. Record
  it in `ServiceOperation.inputs`.

**Migration Effort:** MEDIUM

---

### 22b — JMS Connector (Messaging)

| Pattern | `<jms:config>`, `<jms:publish>`, `<jms:listener>`, `destinationType="QUEUE|TOPIC"` |
|---------|------------------------------------------------------------------------------------|
| **File Types** | `*.xml` |
| **Indicates** | Message publishing or subscription over JMS / message broker |
| **Schema Entity** | `MessagingRequirement`, `ConnectorRequirement`, `IntegrationPoint` (type `MESSAGE_QUEUE`) |

**Example:**
```xml
<jms:config name="JMS_Config">
  <jms:active-mq-connection brokerURL="tcp://mq.example.com:61616"/>
</jms:config>

<flow name="publishOrderEventFlow">
  <http:listener config-ref="HTTP_Listener_config" path="/api/orders" method="POST"/>
  <jms:publish config-ref="JMS_Config" destination="ORDER_QUEUE" destinationType="QUEUE">
    <jms:message>
      <jms:body>#[payload]</jms:body>
    </jms:message>
  </jms:publish>
</flow>

<flow name="consumeOrderEventFlow">
  <jms:listener config-ref="JMS_Config" destination="ORDER_QUEUE" destinationType="QUEUE"/>
  <logger message="Processing order event: #[payload]" level="INFO"/>
</flow>
```

**DSL Schema Mapping:**
```yaml
MessagingRequirement:
  name: "orderQueueMessaging"
  pattern: POINT_TO_POINT
  direction: BIDIRECTIONAL
  destination_type: QUEUE
  destination_name: "ORDER_QUEUE"
  server_type: MULESOFT
  spring_boot_recommendation: "Spring Cloud Stream with RabbitMQ/ActiveMQ binder or @JmsListener"
  migration_effort: MEDIUM
```

```yaml
ConnectorRequirement:
  name: "JMS_Config"
  connector_type: JMS
  server_type: MULESOFT
  operations: [SEND, CONSUME]
  config_ref: "JMS_Config"
  spring_boot_alternative: "JmsTemplate and @JmsListener, or Spring Cloud Stream"
  migration_effort: MEDIUM
```

**Extraction Notes:**
- The broker is an external system. Record an `IntegrationPoint` with `type: MESSAGE_QUEUE`,
  `protocol: JMS`, and the `brokerURL` into `endpoint`. Set `direction: OUTBOUND` for a
  `<jms:publish>` only, `INBOUND` for a `<jms:listener>` only, and `BIDIRECTIONAL` when the
  application does both.
- The message body is a structure. Record a `DataStructure` with `persistence_type: DTO` for it, and
  reference it from `MessagingRequirement`.
- A `<jms:listener>` runs on every instance. Record the consumer group, or say that there is none.

**Migration Effort:** MEDIUM

**Scaling Note:** When scaling across multiple Cloud Foundry instances, ensure message consumer groups are configured to avoid duplicate message processing.

---

### 22c — Mail Connector (SMTP)

| Pattern | `<email:smtp-config>`, `<email:send>` |
|---------|---------------------------------------|
| **File Types** | `*.xml` |
| **Indicates** | Email sending via Mule email connector |
| **Schema Entity** | `MailRequirement`, `IntegrationPoint` (type `EMAIL`, direction `OUTBOUND`) |

**Example:**
```xml
<email:smtp-config name="Gmail_SMTP" from="noreply@company.com" host="smtp.gmail.com" port="587" user="email@gmail.com" password="app_password"/>

<flow name="sendOrderConfirmationFlow">
  <http:listener config-ref="HTTP_Listener_config" path="/api/orders/confirm" method="POST"/>
  <email:send config-ref="Gmail_SMTP" to="#[payload.customerEmail]" subject="Order Confirmed">
    <email:body contentType="text/plain">
      <![CDATA[
        Order #[payload.orderId] has been confirmed.
        Estimated delivery: #[payload.deliveryDate]
      ]]>
    </email:body>
  </email:send>
</flow>
```

**DSL Schema Mapping:**
```yaml
MailRequirement:
  name: "sendOrderConfirmationFlow"
  server_type: MULESOFT
  properties:
    smtp_host: "smtp.gmail.com"
    smtp_port: 587
    smtp_auth: true
    from_address: "noreply@company.com"
  spring_boot_alternative: "JavaMailSender"
  migration_effort: LOW
```

**Extraction Notes:**
- An SMTP server is an external system. Record an `IntegrationPoint` with `type: EMAIL`,
  `direction: OUTBOUND`, `protocol: SMTP`, and the `host` and `port` joined into `endpoint`.
- Never copy the `user` or the `password` on `<email:smtp-config>`. Record the field name and set
  `IntegrationPoint.authentication.credentials_source`.
- `to="#[payload.customerEmail]"` means the recipient comes from the payload, so that field must
  exist on the request `DataStructure`.

**Migration Effort:** LOW

---

### 22d — File Transfer Connector (FTP, SFTP, local file)

| Pattern | `<ftp:config>`, `<sftp:config>`, `<file:config>`, `<ftp:read>`, `<ftp:write>`, `<sftp:read>`, `<sftp:write>`, `<ftp:listener>`, `<sftp:listener>`, `<file:listener>`, `<file:read>`, `<file:write>` |
|---------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **File Types** | `*.xml` |
| **Indicates** | File transfer or file polling via Mule FTP, SFTP or File connector |
| **Schema Entity** | `ConnectorRequirement`, `IntegrationPoint` (type `FTP`, or `FILE_SYSTEM` for a local path) |

**Example:**
```xml
<sftp:config name="SFTP_Config">
  <sftp:connection host="sftp.example.com" port="22" username="etl" password="etlpass"/>
</sftp:config>

<flow name="ingestDailySalesFlow">
  <sftp:listener config-ref="SFTP_Config" directory="/inbound/sales">
    <scheduling-strategy>
      <fixed-frequency frequency="60000"/>
    </scheduling-strategy>
  </sftp:listener>
  <file:write config-ref="File_Config" path="/archive/#[attributes.fileName]"/>
</flow>
```

**DSL Schema Mapping:**
```yaml
ConnectorRequirement:
  name: "SFTP_Config"
  connector_type: SFTP # or FTP for <ftp:config>, FTP for <file:config> with a local path
  server_type: MULESOFT
  operations: [CONSUME, SEND]
  config_ref: "SFTP_Config"
  spring_boot_alternative: "Spring Integration SFTP inbound channel adapter"
  migration_effort: HIGH
```

**Extraction Notes:**
- A `<...:listener>` polls. Record the `<fixed-frequency>` or `<cron>` value, because the target needs an
  equivalent schedule and a guard against every instance polling the same directory.
- A `<...:read>` or `<...:write>` inside a flow is a one-off operation, not a poller. Record it as `SEND` or
  `CONSUME` without a schedule.
- `<file:config>` with a local directory is not a transfer. Record it, and note that local disk on the
  platform is empty after a restart.
- Never carry over a `host`, `username` or `password` found in the connector: record the field name only.
- Record an `IntegrationPoint` for the remote server: `type: FTP` for `<ftp:config>` and
  `<sftp:config>`, `type: FILE_SYSTEM` for `<file:config>` with a local path. Set `direction` from
  the operations found, and put the `host` and `directory` into `endpoint`.
- The file layout is a structure. When the flow parses the file, record a `DataStructure` with
  `persistence_type: DTO` and `data_format` on the integration point (`CSV`, `FIXED_WIDTH`, `JSON`).

**Migration Effort:** HIGH — polling and local file state both need a rethink for the platform

---

## Category 23: DataWeave Transformations

### 23a — DataWeave Mapping (DW 2.0 / DW 1.0)

| Pattern | `<ee:transform>`, `<ee:set-payload>`, `%dw 2.0`, `%dw 1.0` |
|---------|-----------------------------------------------------------|
| **File Types** | `*.xml`, `*.dwl` |
| **Indicates** | Data transformation logic using DataWeave |
| **Schema Entity** | `TransformationRequirement`, `DataStructure` (the input shape and the output shape) |

**Example:**
```xml
<flow name="transformOrderFlow">
  <http:listener config-ref="HTTP_Listener_config" path="/api/orders/transform" method="POST"/>
  <ee:transform>
    <ee:message>
      <ee:set-payload>
        <![CDATA[
          %dw 2.0
          output application/json
          ---
          {
            id: payload.orderId,
            amount: payload.totalPrice,
            date: now() as String {format: "yyyy-MM-dd"},
            items: payload.lineItems map {
              productId: $.id,
              qty: $.quantity,
              price: $.unitPrice * $.quantity
            }
          }
        ]]>
      </ee:set-payload>
    </ee:message>
  </ee:transform>
</flow>
```

**DSL Schema Mapping:**
```yaml
TransformationRequirement:
  name: "transformOrder"
  transformation_type: DATA_WEAVE
  input_format: APPLICATION_JSON
  output_format: APPLICATION_JSON
  server_type: MULESOFT
  spring_boot_alternative: "ObjectMapper or custom mapper"
  migration_effort: MEDIUM
```

**Extraction Notes:**
- A DataWeave script names both sides of the mapping. Record two `DataStructure` entries with
  `persistence_type: DTO`: the input, from every `payload.x` the script reads, and the output, from
  every key the script writes.
- A nested `map { ... }` is a list of a second structure. Record that structure too, and join the two
  with a `Relationship` of `type: ONE_TO_MANY`.
- An expression such as `$.unitPrice * $.quantity` is a derived field. Set `Field.derived: true` and
  put the expression in `Field.derivation_rule`.
- `now()` is wall-clock time. Record it, and note that the target must take a `Clock` so the output
  stays reproducible.

**Migration Effort:** MEDIUM

---

## Category 24: Error Handling & Policies

### 24a — Error Handler (Try / Catch / On Error Propagate / Continue)

| Pattern | `<try>`, `<error-handler>`, `<on-error-propagate>`, `<on-error-continue>` |
|---------|--------------------------------------------------------------------------|
| **File Types** | `*.xml` |
| **Indicates** | Flow-level or application-wide error handling strategies |
| **Schema Entity** | `ErrorHandlingRequirement` |

**Example:**
```xml
<flow name="orderFlow">
  <http:listener config-ref="HTTP_Listener_config" path="/api/orders" method="POST"/>
  <try>
    <db:insert config-ref="Database_Config">
      <db:sql>INSERT INTO orders VALUES (#[payload.id], #[payload.amount])</db:sql>
    </db:insert>
    <set-payload value='{"status": "success"}'/>
  </try>
  <error-handler>
    <on-error-propagate type="DB_ERROR">
      <logger message="Database error occurred" level="ERROR"/>
      <set-payload value='{"error": "Database operation failed"}'/>
      <http:response statusCode="500"/>
    </on-error-propagate>
    <on-error-continue type="ANY">
      <logger message="Unexpected error" level="ERROR"/>
      <set-payload value='{"error": "Internal server error"}'/>
    </on-error-continue>
  </error-handler>
</flow>
```

**DSL Schema Mapping:**
```yaml
ErrorHandlingRequirement:
  name: "orderFlow"
  error_type: "DB_ERROR"
  error_strategy: PROPAGATE
  server_type: MULESOFT
  http_status_code: 500
  spring_boot_alternative: "@ExceptionHandler"
  migration_effort: MEDIUM
```

**Migration Effort:** MEDIUM

---

## Category 25: Clustering & Monitoring

### 25a — Flow with Monitoring and Logging

| Pattern | `<logger message=`, `level="INFO|DEBUG|ERROR|WARN"`, `#[sizeOf(payload)]` |
|---------|---------------------------------------------------------------------------|
| **File Types** | `*.xml` |
| **Indicates** | Flow logging and instrumentation |
| **Schema Entity** | `MonitoringRequirement` |

**DSL Schema Mapping:**
```yaml
MonitoringRequirement:
  name: "orderFlow"
  metrics_source: VENDOR_API
  server_type: MULESOFT
  spring_boot_alternative: "SLF4J + Logback"
  migration_effort: LOW
```

**Migration Effort:** LOW
