# Technical Context Schema

This document defines the schema for extracting technical requirements from legacy applications (Spring Web, EJB, COBOL) to enable translation to Spring Boot for Cloud Foundry deployment.

## Table of Contents

- [Overview](#overview)
- [Core Entities](#core-entities)
  - [ServiceEndpoint](#serviceendpoint)
  - [ServiceCommand](#servicecommand)
  - [ServiceOperation](#serviceoperation)
  - [DataStructure](#datastructure)
  - [IntegrationPoint](#integrationpoint)
  - [ExternalResourceRequirement](#externalresourcerequirement)
  - [JNDIDatasource](#jndidatasource)
  - [MailRequirement](#mailrequirement)
  - [MessagingRequirement](#messagingrequirement)
  - [SecurityRequirement](#securityrequirement)
  - [TransactionRequirement](#transactionrequirement)
  - [SessionManagementRequirement](#sessionmanagementrequirement)
  - [CacheRequirement](#cacherequirement)
  - [MonitoringRequirement](#monitoringrequirement)
  - [MultiTenancyRequirement](#multitenancyrequirement)
  - [ServiceTask](#servicetask)
  - [ServiceBatchProcess](#servicebatchprocess)
  - [ServerMigrationRequirement](#servermigrationrequirement)
  - [ExternalComponentDependency](#externalcomponentdependency)
  - [LdapRequirement](#ldaprequirement)
- [Integration Framework Entities (MuleSoft)](#integration-framework-entities-mulesoft)
  - [IntegrationFlowRequirement](#integrationflowrequirement)
  - [ConnectorRequirement](#connectorrequirement)
  - [TransformationRequirement](#transformationrequirement)
  - [ErrorHandlingRequirement](#errorhandlingrequirement)
- [Server Types](#server-types)
- [UI Layer Entities](#ui-layer-entities)
  - [ViewRequirement](#viewrequirement)
  - [ViewComponent](#viewcomponent)
  - [NavigationFlow](#navigationflow)
  - [FormDefinition](#formdefinition)
  - [TilesDefinition](#tilesdefinition)
  - [StrutsActionMapping](#strutsactionmapping)
  - [JSFManagedBean](#jsfmanagedbean)
- [Composition Structure](#composition-structure)
  - [ModernizationRequirements](#modernizationrequirements)
  - [UserStory](#userstory)
- [Complete Schema Reference](#complete-schema-reference)

---

## Overview

The Technical Context Schema provides a structured way to capture the technical aspects of legacy applications during requirements extraction. This schema is technology-agnostic at the requirement level, enabling consistent translation to Spring Boot regardless of the source technology (Spring Web, EJB, or COBOL).

### Purpose

1. **Requirements Extraction**: Guide the LLM to extract structured technical information from legacy code
2. **Translation Mapping**: Provide clear mappings from legacy constructs to Spring Boot equivalents
3. **Cloud Foundry Readiness**: Capture information needed for CF deployment (service bindings, environment config)

---

## Core Entities

### ServiceEndpoint

Represents a web endpoint exposed by the application (REST API, web controller, or SOAP service).

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `path` | string | Yes | The URL path pattern (e.g., `/orders/{id}`) |
| `method` | enum | Yes | HTTP method: `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `HEAD`, `OPTIONS` |
| `path_variables` | array[PathVariable] | No | Variables extracted from the URL path |
| `query_parameters` | array[QueryParameter] | No | Query string parameters |
| `request_body` | DataStructure | No | Request payload structure (for POST/PUT/PATCH) |
| `response_type` | enum | Yes | Response type: `JSON`, `XML`, `VIEW`, `REDIRECT`, `FILE`, `VOID` |
| `response_body` | DataStructure | No | Response payload structure. When `response_type` is `VIEW`, **required**: must reference the view model (DataStructure or ViewComponent) that is passed to the view, capturing the structure of data available to the UI. |
| `authentication` | enum | Yes | Authentication requirement: `ANONYMOUS`, `AUTHENTICATED`, `API_KEY` |
| `authorization` | array[string] | No | Required roles (e.g., `["ROLE_ADMIN", "ROLE_USER"]`) |
| `consumes` | array[string] | No | Accepted content types (e.g., `["application/json"]`) |
| `produces` | array[string] | No | Produced content types |
| `error_responses` | array[ErrorResponse] | No | Expected error responses with HTTP status codes |
| `operation` | string | No | Reference to the ServiceOperation name that this endpoint invokes (when it stores or retrieves data). The endpoint is responsible for mapping `path_variables`, `query_parameters`, and `request_body` to that operation's `inputs`; parameter binding from HTTP is an endpoint concern, not part of the operation definition. |
| `invoked_by_ui` | array[string] | No | References to the ViewComponent names (or `"viewName/formName"`) that trigger this endpoint (form action, view request, link, or AJAX). Omit when the endpoint is only called by non-UI callers. |

**Extraction rule:** When an endpoint stores or retrieves data (has `request_body`, `response_body`, or delegates to a backend service/repository), it must have a corresponding ServiceOperation; reference it in `operation` when applicable. When an endpoint references an operation, ensure the endpoint's HTTP parameters (path, query, body) can be mapped to the operation's `inputs`; the operation itself remains defined only in terms of data.

**VIEW response_body rule:** Every ServiceEndpoint with `response_type: VIEW` MUST have `response_body` populated with the view model (DataStructure or reference) passed to the view; this captures the UI data contract and must align with the backing ServiceOperation's `output` and the UI Specification (ViewComponent / model_attributes).

**Invoking UI rule:** When an endpoint is triggered from the UI (form submission, page request, link, or AJAX call), trace view/form → request path/method → this endpoint and populate `invoked_by_ui` with references to those ViewComponent names and, when applicable, form names (e.g. `["orderList"]` for a view request, `["orderList/orderForm"]` for form submit). Leave `invoked_by_ui` empty or omitted when the endpoint is only called by non-UI callers (scheduled task, message handler, batch, etc.).

**PathVariable Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Variable name |
| `type` | string | Yes | Data type (e.g., `Long`, `String`, `UUID`) |
| `description` | string | No | Purpose of this variable |

**QueryParameter Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Parameter name |
| `type` | string | Yes | Data type |
| `required` | boolean | Yes | Whether the parameter is mandatory |
| `default_value` | string | No | Default value if not provided |

**ErrorResponse Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `status_code` | integer | Yes | HTTP status code (e.g., `404`, `500`) |
| `condition` | string | Yes | When this error occurs |
| `response_body` | DataStructure | No | Error response structure |

**Example:**

```yaml
ServiceEndpoint:
  path: "/orders/{id}"
  method: GET
  path_variables:
    - name: id
      type: Long
      description: "Unique order identifier"
  response_type: JSON
  response_body: Order
  authentication: AUTHENTICATED
  authorization: ["ROLE_USER", "ROLE_ADMIN"]
  produces: ["application/json"]
  error_responses:
    - status_code: 404
      condition: "Order not found"
```

**Example (endpoint invoked by UI):**

```yaml
ServiceEndpoint:
  path: "/orders/{id}/cancel"
  method: POST
  path_variables:
    - name: id
      type: Long
      description: "Order identifier"
  response_type: JSON
  response_body: Order
  authentication: AUTHENTICATED
  authorization: ["ROLE_USER", "ROLE_ADMIN"]
  operation: cancelOrder
  invoked_by_ui: ["orderDetail/cancelOrderForm"]
  error_responses:
    - status_code: 404
      condition: "Order not found"
    - status_code: 400
      condition: "Order already shipped or delivered"
```

---

### ServiceCommand

Represents a command-line interface command exposed by the application.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Command identifier (for reference) |
| `command_key` | string | Yes | Shell command key (e.g., "export-data", "greet") |
| `description` | string | No | Command purpose and help text |
| `command_aliases` | array[string] | No | Alternative command names (e.g., ["export", "dump-data"]) |
| `parent_command` | string | No | Parent command for subcommands (e.g., "orders" for "orders list") |
| `options` | array[CommandOption] | No | Command-line options/flags |
| `positional_parameters` | array[CommandParameter] | No | Positional parameters |
| `operation` | string | Yes | Reference to ServiceOperation.name that implements command logic |
| `output_format` | enum | No | Display format: `TEXT`, `JSON`, `TABLE`, `NONE`. Default: TEXT |
| `interactive_features` | CommandInteractiveFeatures | No | Interactive shell features |
| `legacy_config` | LegacyCLIConfig | No | Original CLI framework info |

**CommandOption Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Option name - maps to ServiceOperation input parameter name |
| `short_name` | string | No | Short flag (e.g., "f", "t") |
| `long_name` | string | Yes | Long flag (e.g., "file", "type") |
| `type` | string | Yes | Data type (must match operation input type) |
| `required` | boolean | Yes | Whether option is required |
| `default_value` | string | No | Default value if not provided |
| `description` | string | Yes | Help text for this option |

**CommandParameter Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Parameter name - maps to ServiceOperation input parameter name |
| `type` | string | Yes | Data type (must match operation input type) |
| `index` | integer | Yes | Position index (0-based) |
| `required` | boolean | Yes | Whether parameter is required |
| `description` | string | Yes | Help text for this parameter |

**CommandInteractiveFeatures Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `completion_support` | boolean | No | Tab completion available |
| `conditional_availability` | string | No | Condition for command availability (e.g., "user authenticated") |
| `prompt_customization` | string | No | Custom prompt text |

**LegacyCLIConfig Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `framework` | enum | Yes | Source CLI framework: `PICOCLI`, `COMMONS_CLI`, `JCOMMANDER`, `SPRING_SHELL_1_X` |
| `command_class` | string | No | Original command class name |
| `runner_type` | string | No | Runner type: `COMMAND_LINE_RUNNER`, `APPLICATION_RUNNER`, or null |
| `main_class` | string | No | Main class for standalone CLI |

**Example:**

```yaml
ServiceCommand:
  name: "exportDataCommand"
  command_key: "export-data"
  description: "Export order data to a file"
  command_aliases: ["export", "dump"]
  options:
    - name: "filename"
      short_name: "f"
      long_name: "file"
      type: String
      required: true
      description: "Output file path"
    - name: "format"
      short_name: "t"
      long_name: "type"
      type: ExportFormat
      required: false
      default_value: "CSV"
      description: "Export format (CSV, JSON, XML)"
  operation: "exportOrderData"
  output_format: TEXT
  legacy_config:
    framework: PICOCLI
    command_class: "com.example.ExportCommand"
    runner_type: "COMMAND_LINE_RUNNER"
```

**Example (subcommand):**

```yaml
ServiceCommand:
  name: "ordersCancelCommand"
  command_key: "orders cancel"
  description: "Cancel an order"
  parent_command: "orders"
  options:
    - name: "orderId"
      short_name: "i"
      long_name: "id"
      type: Long
      required: true
      description: "Order ID to cancel"
  operation: "cancelOrder"
  output_format: TEXT
```

---

### ServiceOperation

Represents a business logic or data operation (service method), independent of transport.

ServiceOperation corresponds only to data operations (business logic, persistence, external systems). It does not depend on HTTP (or any transport) for reading parameters. Its `inputs` are data parameters; how those values are supplied (e.g., from an HTTP endpoint, message, or scheduler) is the responsibility of the caller (e.g., ServiceEndpoint, ServiceTask, message handler).

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Operation name (method name) |
| `description` | string | No | Business purpose of the operation |
| `type` | enum | Yes | Operation type: `QUERY`, `COMMAND`, `EVENT_HANDLER`, `SCHEDULED`, `BATCH_STEP` |
| `inputs` | array[OperationParameter] | No | Input parameters |
| `output` | DataStructure | No | Return type (DataStructure reference or primitive, e.g. `void`). When the operation returns domain data (QUERY or COMMAND with return value), prefer referencing a DTO (DataStructure with explicit fields) so the returned shape is clear. When the return type is an entity or other complex type, use `output_shape` to list what is loaded. |
| `output_shape` | array[string] | No | When `output` references an entity or complex type (and the exact shape is not already fully defined by a DTO), list the property paths and relationship names that are loaded/returned (e.g. `["id", "orderNumber", "status", "customer.id", "customer.name", "lineItems"]`). Omit when `output` is `void`, a primitive, or a DTO whose fields already define the shape. |
| `transactional` | boolean | Yes | Whether the operation requires transaction management |
| `transaction_propagation` | enum | No | Propagation: `REQUIRED`, `REQUIRES_NEW`, `SUPPORTS`, `MANDATORY`, `NOT_SUPPORTED`, `NEVER`, `NESTED` |
| `read_only` | boolean | No | Whether the transaction is read-only (for queries) |
| `preconditions` | array[Precondition] | No | Business rules that must hold before the operation proceeds. Validation is designed here; do not repeat precondition checks in `steps`. |
| `steps` | array[string] | No | Ordered list of operations performed. Use variables in **`$name`** format (e.g. `$reservation`, `$order`). For loads use "Get *Entity* as *$name* (*criteria*)" and **always** describe the criteria (e.g. "by orderId", "by idPrenotazione"). Later steps reference the variable (e.g. "Update $reservation's visitors from form"). **Steps must not contain** phrases like "Validate X", "Check X", or "Verify X"—replace any such requirement with one or more `preconditions` with a specific `description`; steps list only non-validation actions (Get, Add, Update, Return, Send). **Steps must not contain** UI or presentation actions (e.g. "Return for view", "Redirect to view", "Show view", "Render page"); the operation returns data via `output`; how that data is used (e.g. rendered in a view) is the responsibility of the endpoint or controller. Do **not** include a step that only indicates returning the result (e.g. "Return output"). Do **not** add a separate "Load X for $var" step; do **not** combine multiple output assignments in one step. For operations that return data, use **one step per output property**: each field of the operation's `output` DataStructure gets exactly one step of the form **Set output.\<field\> = \<source\>** and, when the assignment implies loading (e.g. associations), add ** to load the \<description\>** (e.g. "Set output.visitatori = $reservation.visitatori to load the visitors"). Use the exact field names of the operation's `output` type and the exact relationship/field names from the extracted entity model; steps must be valid for the DTOs and entities in the technical context. Always refer to **output**. |
| `structural_events` | array[StructuralEvent] | No | Conceptual changes to the domain when the operation succeeds: entity created/updated/deleted, links created/deleted, messages sent, etc. This is the single source of "what changes"; translators map these to persistence and messaging. |
| `exceptions` | array[ExceptionInfo] | No | Exceptions that may be thrown |
| `dependencies` | array[string] | No | Other services/operations this depends on |

**Completeness rule:** When the operation reads or writes data (QUERY, COMMAND, or any operation that touches persistence/external systems), extraction MUST populate `inputs` (all parameters) and `output` (return type or view model). For operations that feed a VIEW, `output` (or the linked ViewComponent.model_attributes) must describe the structure passed to the view. `inputs` are data parameters only; they must not be defined in terms of HTTP (path, query, or body)—that mapping belongs to the endpoint or other invoker.

**Output shape rule:** For every ServiceOperation that returns data (QUERY or COMMAND with a return value), the properties that need to be loaded must be explicit. **Preferred:** Reference a DTO (DataStructure with `persistence_type: DTO`) as `output` and define that DTO with explicit fields (and nested structures) so the returned shape is clear. **When the return type is an entity or complex type:** Populate `output_shape` with the list of property paths and relationship names that are actually loaded (e.g. scalar fields plus paths like `customer.id`, `customer.name`, `lineItems`). Omit `output_shape` when `output` is `void`, a primitive, or a DTO whose fields already define the shape.

**Preconditions rule:** When the operation validates input or entity state (e.g. "order can be cancelled", "time slot valid", "visitor data valid"), design those as `preconditions` entries with a **specific** `description`, optional `condition`, `inputs_used`, and `failure_exception`. Mandatory/not-null input rules (e.g. "Order identifier must be present") should be expressed as preconditions rather than only as parameter validation. If a step would say "Validate X" or "Check X", do **not** leave it as a step—replace it with one or more preconditions whose descriptions state what must hold (e.g. "Name must not be blank", "Document number must not already exist in the visitor list", "Number of visitors must not exceed N"). Precondition descriptions must be specific enough that a product manager can understand the rule without reading code (e.g. "not empty", "unique in list", "within limit"), not only "Validate field X". Do **not** repeat precondition checks in `steps`; steps list only non-precondition actions (Get, Add, Update, Return, Send).

**Steps and variables rule:** Variables are introduced only by Get steps in `steps` (e.g. "Get Order as $order (by orderId)"); there is no separate `loaded_entities` section. Every Get step must describe the criteria to obtain the element(s). Use `$name` in steps and in `structural_events.instance_variable` when referring to an instance obtained by a Get step.

**Steps (no UI) rule:** Steps describe only data operations (Get, Add, Update, Return [data], Send). They must not contain UI or presentation actions such as "Return for view", "Redirect to view", "Show view", or "Render page"; the operation's `output` describes returned data; the endpoint or controller decides whether that data is rendered in a view, returned as JSON, etc.

**Steps (no redundant return) rule:** Every step must correspond to a concrete data action (Get, Add, Update, or Send). Do not include a step that merely says the operation returns (e.g. "Return output"). Do not include a standalone "Load X for $var" step when output assignments already express the loading (e.g. "Set output.visitatori = $reservation.visitatori to load the visitors").

**Steps (one per output property) rule:** For operations with an `output` type, use **one step per field** of the output DataStructure, each of the form **Set output.\<fieldName\> = \<source\>** [** to load the \<description\>**]. Do not combine multiple output assignments in one step.

**Steps (valid for extracted model) rule:** Output field names must match the operation's output DataStructure; source variables and paths (e.g. $var.relationshipName) must use the exact names from the extracted entities/DataStructures. Do not invent property or relationship names—derive step text from the actual DTO and entity definitions in the technical context.

**Precondition Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `id` | string | No | Short id for reference (e.g. `TimeSlotOrder`, `NoOverlap`) |
| `description` | string | Yes | Human-readable rule; must be specific enough that a product manager can understand the rule without reading code (e.g. "Name must not be blank", "Document number must not already exist in the visitor list", "Start time must be before end time"). State what must hold (e.g. not empty, unique in list, within limit), not only "Validate X". |
| `condition` | string | No | Formal or semi-formal condition (e.g. "oraInizio < oraFine", "status ∉ {SHIPPED, DELIVERED}") |
| `inputs_used` | array[string] | No | Operation input names involved (e.g. `["oraInizio", "oraFine"]`, `["orderId"]`) |
| `failure_exception` | string | No | Exception type when violated (e.g. `InvalidTimeSlotException`, `InvalidOrderStateException`) |
| `failure_message` | string | No | Message or i18n key shown when violated |

**OperationParameter Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Parameter name |
| `type` | string | Yes | Data type or DataStructure reference |
| `required` | boolean | Yes | Whether the parameter is mandatory |
| `validation` | array[string] | No | Validation rules (e.g., `["NotNull", "Size(min=1, max=100)"]`). Do not duplicate rules that are expressed as ServiceOperation preconditions; preconditions are the preferred source for input validation. |

**StructuralEvent Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `kind` | enum | Yes | `ENTITY_CREATED`, `ENTITY_UPDATED`, `ENTITY_DELETED`, `LINK_CREATED`, `LINK_DELETED`; optionally `MESSAGE_SENT`, `EXTERNAL_CALL` for non-persistence effects |
| `entity` | string | Yes | DataStructure name (e.g. `Reservation`, `Order`) |
| `instance_variable` | string | No | When the effect applies to an entity instance obtained by a Get step, the variable in **`$name`** format (e.g. `$reservation`, `$order`). |
| `keyed_by` | string | No | When not using a variable: which input keys this instance (e.g. `orderId`) |
| `attributes` | array[string] | No | For CREATE/UPDATE: which attributes are set (e.g. `["dataOra", "stato", "oraInizio", "oraFine"]`) |
| `relationship` | string | No | For LINK_*: relationship name or target entity (e.g. `TimeSlot`, `reservation.visitors`). For effects on related entities, the DataStructure they are related to (e.g. `Prenotazione`). |
| `value_source` | string | No | Where values come from (e.g. `prenotazioneForm`, `inputs`) |
| `description` | string | No | One-line summary (e.g. "Order status set to CANCELLED and cancellation timestamp set"; "List from prenotazioneForm.visitors replaces $reservation.visitors") |

**ExceptionInfo Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `type` | string | Yes | Exception class name |
| `condition` | string | Yes | When this exception is thrown |
| `handling` | string | No | How it should be handled |

**Example:**

```yaml
ServiceOperation:
  name: "cancelOrder"
  description: "Cancels an existing order and notifies the customer"
  type: COMMAND
  inputs:
    - name: orderId
      type: Long
      required: true
  output: void
  transactional: true
  transaction_propagation: REQUIRED
  preconditions:
    - id: OrderIdPresent
      description: "Order identifier must be present"
      inputs_used: ["orderId"]
      failure_exception: IllegalArgumentException
    - id: OrderExists
      description: "Order must exist"
      inputs_used: ["orderId"]
      failure_exception: OrderNotFoundException
    - id: OrderCancellable
      description: "Order must not be SHIPPED or DELIVERED"
      inputs_used: ["orderId"]
      failure_exception: InvalidOrderStateException
  steps:
    - "Get Order as $order (by orderId)"
    - "Update $order status to CANCELLED and set cancellation timestamp"
    - "Send cancellation notification to customer"
  structural_events:
    - kind: ENTITY_UPDATED
      entity: Order
      instance_variable: "$order"
      attributes: ["status", "cancelledAt"]
      value_source: "(operation: status=CANCELLED, cancelledAt=now)"
      description: "Order status set to CANCELLED and cancellation timestamp set"
    - kind: MESSAGE_SENT
      description: "Cancellation email sent to customer"
  exceptions:
    - type: OrderNotFoundException
      condition: "Order with given ID does not exist"
    - type: InvalidOrderStateException
      condition: "Order is already shipped or delivered"
```

**Example (update with related entities and value mapping):**

```yaml
ServiceOperation:
  name: "updateReservationAdmin"
  description: "Updates or deletes reservation (admin)"
  type: COMMAND
  inputs:
    - name: idPrenotazione
      type: Integer
      required: true
    - name: prenotazioneForm
      type: PrenotazioneForm
      required: false
  output: void
  transactional: true
  preconditions:
    - id: ReservationExists
      description: "Reservation must exist"
      inputs_used: ["idPrenotazione"]
      failure_exception: ReservationNotFoundException
  steps:
    - "Get Prenotazione as $reservation (by idPrenotazione)"
    - "Update $reservation's visitors, vehicles, and contact from form or delete $reservation"
  structural_events:
    - kind: ENTITY_UPDATED
      entity: Prenotazione
      instance_variable: "$reservation"
      attributes: ["dataOra", "stato", "note"]
      value_source: prenotazioneForm
      description: "Reservation header and status from prenotazioneForm; or reservation is deleted when form indicates delete."
    - kind: ENTITY_UPDATED
      entity: Visitatore
      relationship: Prenotazione
      instance_variable: "$reservation"
      value_source: prenotazioneForm.visitors
      attributes: ["nome", "documento", "tipoDocumento"]
      description: "List from prenotazioneForm.visitors replaces $reservation.visitors; each item maps to Visitatore."
    - kind: ENTITY_UPDATED
      entity: VeicoliPrenotazioneAss
      relationship: Prenotazione
      instance_variable: "$reservation"
      value_source: prenotazioneForm.vehicles
      attributes: ["targa", "tipoVeicolo"]
      description: "List from prenotazioneForm.vehicles replaces $reservation.vehicles; each item maps to VeicoliPrenotazioneAss."
  exceptions:
    - type: ReservationNotFoundException
      condition: "Reservation with given idPrenotazione does not exist"
    - type: (validation)
      condition: "Invalid data in form"
```

**Example (input validation as preconditions—add visitor to list):**

When an operation validates form/input data (e.g. "validate visitor data", "check duplicate and max visitors"), do **not** put "Validate X" or "Check X" in steps. Define each rule as a precondition with a specific description so a PM can understand what is validated without reading code.

```yaml
ServiceOperation:
  name: "addVisitatore"
  description: "Add visitor to current visitor list after validation (returns updated list)"
  type: COMMAND
  inputs:
    - name: prenotazioneForm
      type: PrenotazioneForm
      required: true
    - name: listaVisitatori
      type: List<MdzVisitatore>
      required: true
      description: "Current visitor list (controller provides from session)"
  output: "List<MdzVisitatore>"
  output_shape: ["id", "nome", "cognome", "nDocumento", "dataNascita", "mdzTipoVisitatore.id"]
  transactional: false
  preconditions:
    - id: NameNotBlank
      description: "Name (nome) must not be blank"
      inputs_used: ["prenotazioneForm"]
      failure_exception: ValidationException
    - id: SurnameNotBlank
      description: "Surname (cognome) must not be blank"
      inputs_used: ["prenotazioneForm"]
      failure_exception: ValidationException
    - id: TypeSelected
      description: "Visitor type (mdzTipoVisitatore) must be selected"
      inputs_used: ["prenotazioneForm"]
      failure_exception: ValidationException
    - id: DocumentNotBlank
      description: "Document number (nDocumento) must not be blank"
      inputs_used: ["prenotazioneForm"]
      failure_exception: ValidationException
    - id: NoDuplicateDocument
      description: "Document number must not already exist in the current visitor list"
      inputs_used: ["prenotazioneForm", "listaVisitatori"]
      failure_exception: DuplicateDocumentException
    - id: MaxVisitorsNotExceeded
      description: "Adding one more visitor must not exceed the maximum visitors per reservation"
      inputs_used: ["listaVisitatori"]
      condition: "listaVisitatori.size() + 1 <= maxVisitors"
      failure_exception: MaxVisitorsExceededException
  steps:
    - "Add visitor from prenotazioneForm to listaVisitatori and return updated list"
  structural_events:
    - kind: ENTITY_CREATED
      entity: MdzVisitatore
      value_source: prenotazioneForm
      description: "New visitor built from prenotazioneForm and appended to list; returned list is previous list plus new visitor"
  exceptions:
    - type: ValidationException
      condition: "Required field missing or invalid (name, surname, type, document)"
    - type: DuplicateDocumentException
      condition: "Document number already in listaVisitatori"
    - type: MaxVisitorsExceededException
      condition: "Maximum visitors per reservation would be exceeded"
```

**Example (QUERY with output; one step per output property):**

For operations that return a DTO, use one step per field of the output DataStructure. Do not use "Return output" or a separate "Load X for $var" step; do not combine assignments in one step. Use exact field names from the output DTO and exact relationship names from the extracted entity model.

```yaml
ServiceOperation:
  name: "getReservationById"
  description: "Loads reservation by ID with visitors and vehicles for admin edit"
  type: QUERY
  inputs:
    - name: idPrenotazione
      type: Integer
      required: true
  output: ReservationEditViewModel
  transactional: true
  read_only: true
  preconditions:
    - id: IdPrenotazioneNotNull
      description: "Reservation ID (idPrenotazione) must not be null"
      inputs_used: ["idPrenotazione"]
      failure_exception: IllegalArgumentException
    - id: ReservationExists
      description: "Reservation must exist"
      inputs_used: ["idPrenotazione"]
      failure_exception: ReservationNotFoundException
  steps:
    - "Get MdzPrenotazione as $reservation (by idPrenotazione)"
    - "Set output.prenotazione = $reservation"
    - "Set output.visitatori = $reservation.visitatori to load the visitors"
    - "Set output.veicoli = $reservation.veicoli to load the vehicle associations"
  exceptions:
    - type: ReservationNotFoundException
      condition: "Reservation ID does not exist"
```

---

### DataStructure

Represents a data model, entity, DTO, or value object.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Structure name (class name) |
| `description` | string | No | Purpose of this structure |
| `persistence_type` | enum | Yes | Type: `ENTITY`, `VALUE_OBJECT`, `DTO`, `EMBEDDABLE`, `NONE` |
| `table_name` | string | No | Database table name (for entities) |
| `primary_key_fields` | array[string] | No | Ordered list of field names for composite primary keys. For single PK, optional if `primary_key: true` is set on the field. |
| `unique_constraints` | array[UniqueConstraint] | No | Composite unique constraints (single-column uniqueness uses Field.`unique`) |
| `fields` | array[Field] | Yes | Field definitions |
| `relationships` | array[Relationship] | No | Relationships to other structures |
| `validations` | array[string] | No | Class-level validations |
| `auditing` | AuditingConfig | No | Audit field configuration |

**Field Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Field name |
| `type` | string | Yes | Data type (primitive, wrapper, or reference to another DataStructure) |
| `column_name` | string | No | Database column name (if different from field name) |
| `primary_key` | boolean | No | `true` when the field is part of the primary key. Entities must have at least one primary key field (or composite via `primary_key_fields`). |
| `nullable` | boolean | No | Whether the field can be null (default: true) |
| `unique` | boolean | No | Whether the field must be unique (single-column uniqueness only) |
| `length` | integer | No | Maximum length (for strings) |
| `precision` | integer | No | Numeric precision |
| `scale` | integer | No | Numeric scale |
| `min_value` | number/string | No | Valid minimum value for numeric types (use string for BigDecimal if needed) |
| `max_value` | number/string | No | Valid maximum value for numeric types (use string for BigDecimal if needed) |
| `default_value` | string | No | Default value |
| `derived` | boolean | No | `true` when the field is computed/derived (not stored) |
| `derivation_rule` | string | No | How the value is derived (e.g. formula, expression, or reference like "sum of line items") |
| `validations` | array[string] | No | Field-level validations |
| `description` | string | No | Field purpose |
| `loaded_shape` | array[string] | No | When `type` references an entity or collection of entity (e.g. `MdzVisitatore`, `List<MdzVisitatore>`), list the property paths and relationship names that are loaded for that entity in this DTO (e.g. `["id", "nome", "documento"]`). Omit for primitive types, value objects, or when the field type is a view DTO that already defines the shape. |

**DTO entity-typed fields rule:** When a DTO field's type is an entity (or collection of entity), the loaded shape for that entity must be explicit—either set `loaded_shape` on the field or use a nested DataStructure (view DTO) with explicit fields instead of referencing the entity type.

**UniqueConstraint Schema** (for composite uniqueness):

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Constraint name |
| `fields` | array[string] | Yes | Ordered list of field names in the constraint |

**Relationship Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `type` | enum | Yes | Relationship type: `ONE_TO_ONE`, `ONE_TO_MANY`, `MANY_TO_ONE`, `MANY_TO_MANY` |
| `target` | string | Yes | Target DataStructure name |
| `field_name` | string | Yes | Field name for this relationship |
| `multiplicity` | string | No | Cardinality on the target side: `1` (exactly one), `0..1` (optional one), `*` or `0..*` (many), `1..*` (one or many) |
| `optional` | boolean | No | Whether the FK/relationship can be null |
| `mapped_by` | string | No | Owning side field (for bidirectional) |
| `cascade` | array[string] | No | Cascade operations: `["PERSIST", "MERGE", "REMOVE", "REFRESH", "ALL"]` |
| `fetch` | enum | No | Fetch strategy: `LAZY`, `EAGER` |
| `join_column` | string | No | Foreign key column name |
| `join_table` | string | No | Join table name (for many-to-many) |

**AuditingConfig Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `created_date_field` | string | No | Field for creation timestamp |
| `modified_date_field` | string | No | Field for last modification timestamp |
| `created_by_field` | string | No | Field for creator user |
| `modified_by_field` | string | No | Field for last modifier user |

**Example:**

```yaml
DataStructure:
  name: "Order"
  description: "Represents a customer order"
  persistence_type: ENTITY
  table_name: "orders"
  primary_key_fields: ["id"]
  unique_constraints:
    - name: "uk_order_number"
      fields: ["orderNumber"]
  fields:
    - name: id
      type: Long
      primary_key: true
      nullable: false
      description: "Primary key"
    - name: orderNumber
      type: String
      column_name: "order_number"
      length: 50
      unique: true
      validations: ["NotBlank", "Size(max=50)"]
    - name: status
      type: OrderStatus
      nullable: false
      validations: ["NotNull"]
    - name: totalAmount
      type: BigDecimal
      column_name: "total_amount"
      precision: 10
      scale: 2
      derived: true
      derivation_rule: "Sum of order line amounts"
    - name: quantity
      type: Integer
      min_value: 0
      max_value: 9999
      validations: ["NotNull", "Min(0)", "Max(9999)"]
  relationships:
    - type: MANY_TO_ONE
      target: Customer
      field_name: customer
      join_column: "customer_id"
      multiplicity: "1"
      optional: false
      fetch: LAZY
    - type: ONE_TO_MANY
      target: OrderItem
      field_name: items
      mapped_by: "order"
      multiplicity: "0..*"
      cascade: ["ALL"]
  auditing:
    created_date_field: createdAt
    modified_date_field: updatedAt
```

**DTO with entity-typed fields (loaded_shape):** When a DTO wraps entities, make the loaded shape explicit per field:

```yaml
DataStructure:
  name: "ReservationEditViewModel"
  description: "View model for editing a reservation with its visitors and vehicles"
  persistence_type: DTO
  fields:
    - name: prenotazione
      type: MdzPrenotazione
      description: "Reservation entity (loaded shape per output_shape or root entity)"
      loaded_shape: ["id", "dataOra", "stato", "note"]
    - name: visitatori
      type: "List<MdzVisitatore>"
      description: "Visitors for this reservation"
      loaded_shape: ["id", "nome", "documento"]
    - name: veicoli
      type: "List<MdzVeicoliPrenotazioneAss>"
      description: "Vehicle associations"
      loaded_shape: ["id", "veicolo.id", "targa"]
```

---

### IntegrationPoint

Represents an external system integration or communication channel.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Integration name/identifier |
| `type` | enum | Yes | Integration type: `DATABASE`, `MESSAGE_QUEUE`, `REST_API`, `SOAP_API`, `FILE_SYSTEM`, `FTP`, `EMAIL`, `LDAP`, `CACHE` |
| `direction` | enum | Yes | Communication direction: `INBOUND`, `OUTBOUND`, `BIDIRECTIONAL` |
| `protocol` | string | No | Protocol used (e.g., `HTTP`, `AMQP`, `JMS`, `JDBC`) |
| `endpoint` | string | No | Connection endpoint (URL, hostname, queue name) |
| `authentication` | IntegrationAuth | No | Authentication configuration |
| `data_format` | enum | No | Data format: `JSON`, `XML`, `CSV`, `BINARY`, `FIXED_WIDTH` |
| `retry_policy` | RetryPolicy | No | Retry configuration for failures |
| `circuit_breaker` | boolean | No | Whether circuit breaker pattern is needed |
| `timeout_ms` | integer | No | Connection/read timeout in milliseconds |

**IntegrationAuth Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `type` | enum | Yes | Auth type: `NONE`, `BASIC`, `BEARER_TOKEN`, `API_KEY`, `OAUTH2`, `CERTIFICATE` |
| `credentials_source` | string | No | Where credentials come from (e.g., `environment`, `vault`, `config`) |

**RetryPolicy Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `max_attempts` | integer | Yes | Maximum retry attempts |
| `backoff_ms` | integer | No | Initial backoff delay in milliseconds |
| `backoff_multiplier` | number | No | Multiplier for exponential backoff |
| `retryable_exceptions` | array[string] | No | Exceptions that trigger retry |

**Example:**

```yaml
IntegrationPoint:
  name: "payment-gateway"
  type: REST_API
  direction: OUTBOUND
  protocol: HTTPS
  endpoint: "https://api.payments.example.com/v2"
  authentication:
    type: BEARER_TOKEN
    credentials_source: "environment"
  data_format: JSON
  retry_policy:
    max_attempts: 3
    backoff_ms: 1000
    backoff_multiplier: 2.0
  circuit_breaker: true
  timeout_ms: 5000
```

---

### ExternalResourceRequirement

Represents external resources that must be configured (datasources, connection factories, etc.).

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Resource identifier |
| `type` | enum | Yes | Resource type: `DATASOURCE`, `CONNECTION_FACTORY`, `QUEUE`, `TOPIC`, `MAIL_SESSION`, `CACHE`. Use [JNDIDatasource](#jndidatasource) for a JNDI-bound datasource and [MailRequirement](#mailrequirement) for a JNDI mail session, not this entity. |
| `legacy_jndi_name` | string | No | Original JNDI lookup name |
| `legacy_server` | enum | No | Source application server. See [Server Types](#server-types). |
| `connection_pool` | ConnectionPoolConfig | No | Connection pool settings (for `type: DATASOURCE`). Maps to Spring Boot HikariCP properties. |
| `properties` | object | No | Resource-specific properties |
| `spring_boot_config` | SpringBootConfig | No | Recommended Spring Boot configuration |
| `cloud_foundry_service` | CFServiceBinding | No | Cloud Foundry service binding recommendation |

**ConnectionPoolConfig Schema** (for datasources; maps to HikariCP in Spring Boot):

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `maximum_pool_size` | integer | Yes | Maximum number of connections in the pool |
| `minimum_idle` | integer | No | Minimum number of idle connections |
| `connection_timeout_ms` | integer | No | Max milliseconds to wait for a connection |
| `idle_timeout_ms` | integer | No | Max time a connection can sit idle before being closed |
| `max_lifetime_ms` | integer | No | Max lifetime of a connection in the pool |
| `auto_commit` | boolean | No | Default auto-commit behavior for connections |

For Spring Boot, these map to: `spring.datasource.hikari.maximum-pool-size`, `spring.datasource.hikari.minimum-idle`, `spring.datasource.hikari.connection-timeout`, `spring.datasource.hikari.idle-timeout`, `spring.datasource.hikari.max-lifetime`, `spring.datasource.hikari.auto-commit`.

**SpringBootConfig Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `properties` | object | Yes | Spring Boot property mappings (key: property path, value: description/default). For datasources, include HikariCP keys when pool settings are known. |
| `auto_configuration` | string | No | Spring Boot auto-configuration class used |
| `starter_dependency` | string | No | Required Spring Boot starter |

**CFServiceBinding Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `service_type` | string | Yes | CF service type (e.g., `p.mysql`, `p-rabbitmq`) |
| `service_plan` | string | No | Recommended service plan |
| `binding_name` | string | No | Service binding name |
| `cfenv_support` | boolean | No | Whether java-cfenv auto-configures this |

**Example:**

```yaml
ExternalResourceRequirement:
  name: "ordersDataSource"
  type: DATASOURCE
  legacy_jndi_name: "java:jboss/datasources/OrdersDS"
  legacy_server: JBOSS
  connection_pool:
    maximum_pool_size: 20
    minimum_idle: 5
    connection_timeout_ms: 30000
    idle_timeout_ms: 600000
    max_lifetime_ms: 1800000
    auto_commit: true
  properties:
    driver: "com.mysql.cj.jdbc.Driver"
  spring_boot_config:
    properties:
      spring.datasource.url: "Database connection URL"
      spring.datasource.username: "Database username"
      spring.datasource.password: "Database password (use secrets management)"
      spring.datasource.hikari.maximum-pool-size: "20"
      spring.datasource.hikari.minimum-idle: "5"
      spring.datasource.hikari.connection-timeout: "30000"
      spring.datasource.hikari.idle-timeout: "600000"
      spring.datasource.hikari.max-lifetime: "1800000"
      spring.datasource.hikari.auto-commit: "true"
    starter_dependency: "spring-boot-starter-data-jpa"
  cloud_foundry_service:
    service_type: "p.mysql"
    service_plan: "db-small"
    cfenv_support: true
```

---

### JNDIDatasource

Represents a datasource that the legacy application looks up through JNDI, together with the vendor connection-pool configuration that the application server held outside the code.

**Use this entity instead of `ExternalResourceRequirement` with `type: DATASOURCE` whenever the datasource is JNDI-bound.** Emit one or the other, never both for the same resource. Keep `ExternalResourceRequirement` for non-JNDI external resources.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Datasource identifier used elsewhere in the spec (e.g. `"ordersDataSource"`) |
| `jndi_name` | string | Yes | Full JNDI name as used in code or descriptor (e.g. `"java:comp/env/jdbc/OrdersDB"`, `"java:jboss/datasources/OrdersDS"`) |
| `resource_ref_name` | string | No | Value of `<res-ref-name>` in the deployment descriptor (e.g. `"jdbc/OrdersDB"`) |
| `server_type` | enum | Yes | Source application server. See [Server Types](#server-types). |
| `vendor_pool_name` | string | No | Vendor connection-pool name (GlassFish `<jdbc-connection-pool name>`, JBoss `pool-name`, WebLogic data-source name) |
| `datasource_type` | enum | No | Declared type: `javax.sql.DataSource`, `javax.sql.XADataSource`, `javax.sql.ConnectionPoolDataSource` |
| `pool_config` | VendorPoolConfig | No | Vendor pool settings found in the descriptor |
| `multi_tenancy_context` | string | No | Tenant identifier when this datasource serves one tenant (e.g. `"TenantA"`). See [MultiTenancyRequirement](#multitenancyrequirement). |
| `lookup_pattern` | string | No | Code pattern used for the lookup (e.g. `"new InitialContext().lookup('java:comp/env/jdbc/OrdersDB')"`) |
| `spring_boot_migration` | SpringBootDatasourceMigration | No | Recommended Spring Boot properties |
| `cloud_foundry_service` | CFServiceBinding | No | Cloud Foundry service binding recommendation |
| `migration_effort` | enum | No | `LOW`, `MEDIUM`, `HIGH` |
| `remediation_steps` | array[string] | No | Ordered actions a developer must take that the property mapping does not cover |

**VendorPoolConfig Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `max_pool_size` | integer | No | Vendor maximum pool size, as declared (do not adjust it here) |
| `min_pool_size` | integer | No | Vendor minimum/steady pool size |
| `idle_timeout_seconds` | integer | No | Idle timeout in seconds |
| `max_wait_time_millis` | integer | No | Maximum wait for a connection, in milliseconds |
| `connection_validation_method` | string | No | Validation method (e.g. `"table"`, `"auto-commit"`, `"custom-validation"`) |
| `validation_table` | string | No | Validation query or table (e.g. `"select 1"`) |

**SpringBootDatasourceMigration Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `datasource_url` | string | No | JDBC URL, or a note that Cloud Foundry supplies it through a service binding |
| `hikari_maximum_pool_size` | integer | No | Per-instance HikariCP maximum. **This is not a 1:1 copy of `max_pool_size`** — see the sizing rule below. |
| `hikari_minimum_idle` | integer | No | Per-instance HikariCP minimum idle |
| `hikari_idle_timeout_ms` | integer | No | HikariCP idle timeout in milliseconds |
| `hikari_connection_timeout_ms` | integer | No | HikariCP connection timeout in milliseconds |
| `starter_dependency` | string | No | Required Spring Boot starter |

**Pool sizing rule (mandatory):** the vendor pool size was tuned for one always-on server. Cloud Foundry runs several ephemeral instances of the same code, and each one opens its own pool, so `total_connections = maximum_pool_size × instance_count`. Record the vendor number in `pool_config.max_pool_size` unchanged, and record the per-instance number in `spring_boot_migration.hikari_maximum_pool_size`. When the target instance count and the database `max_connections` limit are known, compute it as `floor((db_max_connections × 0.8) / max_planned_instances)`. When they are not known, set `migration_effort: MEDIUM` and add a remediation step that asks for both numbers.

**XA rule:** when `datasource_type` is `javax.sql.XADataSource`, also emit a [TransactionRequirement](#transactionrequirement) with `xa_datasource_detected: true`.

**Example:**

```yaml
JNDIDatasource:
  name: "ordersDataSource"
  jndi_name: "java:comp/env/jdbc/OrdersDB"
  resource_ref_name: "jdbc/OrdersDB"
  server_type: GLASSFISH
  vendor_pool_name: "OrdersDataSourcePool"
  datasource_type: javax.sql.DataSource
  pool_config:
    max_pool_size: 20
    min_pool_size: 5
    idle_timeout_seconds: 900
    max_wait_time_millis: 6000
    connection_validation_method: "table"
    validation_table: "select 1"
  lookup_pattern: "new InitialContext().lookup('java:comp/env/jdbc/OrdersDB')"
  spring_boot_migration:
    datasource_url: "Supplied by the Cloud Foundry mysql service binding"
    hikari_maximum_pool_size: 20
    hikari_minimum_idle: 5
    hikari_idle_timeout_ms: 900000
    starter_dependency: "spring-boot-starter-data-jpa"
  cloud_foundry_service:
    service_type: "p.mysql"
    service_plan: "db-small"
    cfenv_support: true
  migration_effort: LOW
  remediation_steps:
    - "Delete the <resource-ref> block from the deployment descriptor"
    - "Confirm the target instance count, then recompute hikari_maximum_pool_size"
```

---

### MailRequirement

Represents a mail session that the legacy application looks up through JNDI.

**Use this entity instead of `ExternalResourceRequirement` with `type: MAIL_SESSION`.** Emit one or the other, never both for the same resource.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Identifier for this mail configuration (e.g. `"mailSession"`) |
| `legacy_jndi_name` | string | Yes | JNDI name of the mail session (e.g. `"mail/GlassfishMailSession"`) |
| `server_type` | enum | Yes | Source application server. See [Server Types](#server-types). |
| `lookup_pattern` | string | No | Code pattern used for the lookup (e.g. `"ctx.lookup('mail/GlassfishMailSession')"`) |
| `properties` | MailProperties | No | SMTP settings found in the server configuration |
| `spring_boot_alternative` | string | No | Recommended Spring Boot approach (e.g. `"spring.mail.* + JavaMailSender"`) |
| `cloud_foundry_service` | CFServiceBinding | No | Cloud Foundry service binding recommendation, when a mail service is available |
| `migration_effort` | enum | No | `LOW`, `MEDIUM`, `HIGH` |

**MailProperties Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `smtp_host` | string | No | SMTP host |
| `smtp_port` | integer | No | SMTP port |
| `smtp_auth` | boolean | No | `true` when SMTP authentication is enabled |
| `starttls_enabled` | boolean | No | `true` when STARTTLS is enabled |
| `smtp_timeout_ms` | integer | No | Socket timeout in milliseconds |

**Example:**

```yaml
MailRequirement:
  name: "mailSession"
  legacy_jndi_name: "mail/GlassfishMailSession"
  server_type: GLASSFISH
  lookup_pattern: "ctx.lookup('mail/GlassfishMailSession')"
  properties:
    smtp_host: "smtp.example.com"
    smtp_port: 587
    smtp_auth: true
    starttls_enabled: true
  spring_boot_alternative: "spring.mail.* properties injected into JavaMailSender"
  migration_effort: LOW
```

---

### MessagingRequirement

Represents asynchronous messaging patterns and requirements.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Messaging component identifier |
| `pattern` | enum | Yes | Pattern: `POINT_TO_POINT`, `PUBLISH_SUBSCRIBE`, `REQUEST_REPLY`, `FIRE_AND_FORGET` |
| `direction` | enum | Yes | Direction: `PRODUCER`, `CONSUMER`, `BOTH` |
| `destination_type` | enum | Yes | Destination: `QUEUE`, `TOPIC`, `EXCHANGE` |
| `destination_name` | string | Yes | Queue/topic name |
| `message_type` | string | No | Message payload type (DataStructure reference) |
| `legacy_config` | LegacyMessagingConfig | No | Legacy JMS/messaging configuration |
| `concurrency` | integer | No | Number of concurrent consumers |
| `acknowledgement` | enum | No | Ack mode: `AUTO`, `CLIENT`, `TRANSACTED` |
| `error_handling` | MessagingErrorHandling | No | Error handling configuration |
| `spring_boot_recommendation` | string | No | Recommended Spring Boot approach |

**LegacyMessagingConfig Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `connection_factory_jndi` | string | No | JNDI name for connection factory |
| `destination_jndi` | string | No | JNDI name for destination |
| `message_driven_bean` | string | No | MDB class name (for EJB) |

**MessagingErrorHandling Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `dead_letter_queue` | string | No | DLQ destination |
| `max_redeliveries` | integer | No | Maximum redelivery attempts |
| `error_handler` | string | No | Error handler class/method |

**Example:**

```yaml
MessagingRequirement:
  name: "orderNotificationConsumer"
  pattern: POINT_TO_POINT
  direction: CONSUMER
  destination_type: QUEUE
  destination_name: "order-notifications"
  message_type: OrderNotificationMessage
  legacy_config:
    connection_factory_jndi: "java:jboss/jms/ConnectionFactory"
    destination_jndi: "java:jboss/jms/queue/OrderNotifications"
    message_driven_bean: "OrderNotificationMDB"
  concurrency: 5
  acknowledgement: TRANSACTED
  error_handling:
    dead_letter_queue: "order-notifications-dlq"
    max_redeliveries: 3
  spring_boot_recommendation: "Use @JmsListener or Spring Cloud Stream"
```

---

### SecurityRequirement

Represents authentication and authorization requirements.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `authentication_method` | enum | Yes | Method: `FORM`, `BASIC`, `BEARER_TOKEN`, `OAUTH2`, `SAML`, `LDAP`, `CERTIFICATE`, `FILE_REALM`, `JAAS_LOGIN_MODULE`, `ELYTRON` |
| `session_management` | enum | No | Session strategy: `STATEFUL`, `STATELESS`, `HYBRID` |
| `realm_type` | enum | No | Application-server realm class: `FileRealm`, `LDAPRealm`, `CertificateRealm`, `JDBCRealm`, `Custom` |
| `server_type` | enum | No | Source application server, when the realm came from the server. See [Server Types](#server-types). |
| `role_mapping_present` | boolean | No | `true` when the descriptor maps realm principals or groups to application roles |
| `mtls_required` | boolean | No | `true` when client-certificate (mutual TLS) authentication is required |
| `roles` | array[RoleDefinition] | No | Role definitions |
| `protected_endpoints` | array[EndpointSecurity] | No | Endpoint-level security rules |
| `cors_config` | CorsConfig | No | CORS configuration |
| `csrf_protection` | boolean | No | Whether CSRF protection is enabled |
| `legacy_config` | LegacySecurityConfig | No | Original security configuration source |

**RoleDefinition Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Role name (e.g., `ROLE_ADMIN`) |
| `description` | string | No | Role description |
| `permissions` | array[string] | No | Permissions granted by this role |

**EndpointSecurity Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `pattern` | string | Yes | URL pattern (Ant-style or regex) |
| `access` | string | Yes | Access rule (e.g., `authenticated`, `hasRole('ADMIN')`, `permitAll`) |
| `methods` | array[string] | No | HTTP methods this applies to |

**CorsConfig Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `allowed_origins` | array[string] | No | Allowed origin URLs |
| `allowed_methods` | array[string] | No | Allowed HTTP methods |
| `allowed_headers` | array[string] | No | Allowed headers |
| `allow_credentials` | boolean | No | Whether credentials are allowed |
| `max_age` | integer | No | Preflight cache duration in seconds |

**LegacySecurityConfig Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `source` | string | Yes | Configuration source (e.g., `spring-security.xml`, `@Secured annotations`, `web.xml`, `glassfish-web.xml`, `domain.xml`) |
| `realm` | string | No | Security realm name |
| `realm_properties` | object | No | Realm-specific properties as declared by the server (e.g., `file`, `jaas-context`, `assign-groups`, `datasource-jndi`). For an LDAP realm, put the directory details in [LdapRequirement](#ldaprequirement) instead. |
| `role_mappings` | array[RoleMapping] | No | Realm principal/group to application role mappings |

**RoleMapping Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `role_name` | string | Yes | Application role (e.g., `admin`) |
| `principal_names` | array[string] | No | Realm principals mapped to the role |
| `group_names` | array[string] | No | Realm groups mapped to the role |

**Realm rules:**

- `FILE_REALM` — the user list lives inside the application server, not in the application. Record the mappings, and add a remediation note that the target needs a real user store (a database, LDAP, or an identity provider).
- `LDAP` / `realm_type: LDAPRealm` — also populate [LdapRequirement](#ldaprequirement) with the directory URL, base DN, and search filters. `SecurityRequirement` holds the authentication decision; `LdapRequirement` holds the directory details.
- `CERTIFICATE` / `realm_type: CertificateRealm` — set `mtls_required: true`. Cloud Foundry terminates TLS at the router, so record in `legacy_config.realm_properties` how the certificate reaches the application, and flag it for manual review.

**Example:**

```yaml
SecurityRequirement:
  authentication_method: OAUTH2
  session_management: STATELESS
  roles:
    - name: ROLE_USER
      description: "Standard user with read access"
      permissions: ["READ_ORDERS", "CREATE_ORDERS"]
    - name: ROLE_ADMIN
      description: "Administrator with full access"
      permissions: ["READ_ORDERS", "CREATE_ORDERS", "CANCEL_ORDERS", "MANAGE_USERS"]
  protected_endpoints:
    - pattern: "/api/**"
      access: "authenticated"
    - pattern: "/api/admin/**"
      access: "hasRole('ADMIN')"
    - pattern: "/actuator/health"
      access: "permitAll"
  cors_config:
    allowed_origins: ["https://app.example.com"]
    allowed_methods: ["GET", "POST", "PUT", "DELETE"]
    allow_credentials: true
  csrf_protection: false
```

---

### TransactionRequirement

Represents transaction management requirements at the application level.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `transaction_manager_type` | enum | Yes | Manager type: `JPA`, `JDBC`, `JTA`, `CONTAINER_MANAGED`, `REACTIVE` |
| `default_propagation` | enum | No | Default propagation behavior |
| `default_isolation` | enum | No | Default isolation level: `DEFAULT`, `READ_UNCOMMITTED`, `READ_COMMITTED`, `REPEATABLE_READ`, `SERIALIZABLE` |
| `default_timeout` | integer | No | Default transaction timeout in seconds |
| `rollback_rules` | array[RollbackRule] | No | Exception-based rollback rules |
| `distributed_transactions` | boolean | No | Whether XA/distributed transactions are needed |
| `xa_datasource_detected` | boolean | No | `true` when a `javax.sql.XADataSource` is referenced in a descriptor or in code |
| `legacy_datasource_jndi` | string | No | JNDI name of the XA datasource in the legacy system |
| `server_type` | enum | No | Source application server, when the transaction manager came from the server. See [Server Types](#server-types). |
| `scope` | enum | No | How much of the application depends on XA: `GLOBAL`, `LOCAL`, `REQUIRES_USER_FEEDBACK` |
| `spring_boot_alternative` | string | No | Recommended Spring Boot approach (e.g. `"@Transactional"`, `"Transactional Outbox"`, `"Saga with compensating actions"`) |
| `legacy_config` | string | No | Original configuration source |

**RollbackRule Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `exception_type` | string | Yes | Exception class name |
| `rollback` | boolean | Yes | Whether to rollback on this exception |

**XA scope rule (mandatory):** XA / two-phase commit needs a durable recovery log and a platform transaction coordinator. Cloud Foundry instances are ephemeral and the platform supplies neither, so a detected XA pattern is not a line-for-line translation. When `xa_datasource_detected` is `true`, set `scope`:

- `GLOBAL` — code shows XA across most flows. The application needs a redesign.
- `LOCAL` — code shows XA in one or two components. Refactor those components only.
- `REQUIRES_USER_FEEDBACK` — the code does not show which of the two it is. **Ask the user before translation starts:** *"Is XA / distributed transaction support critical across your application, or is it used only in specific components?"* Record the answer as `GLOBAL` or `LOCAL`.

Do not translate an XA pattern automatically while `scope` is `REQUIRES_USER_FEEDBACK`.

**Example:**

```yaml
TransactionRequirement:
  transaction_manager_type: JPA
  default_propagation: REQUIRED
  default_isolation: READ_COMMITTED
  default_timeout: 30
  rollback_rules:
    - exception_type: "BusinessException"
      rollback: false
    - exception_type: "RuntimeException"
      rollback: true
  distributed_transactions: false
  legacy_config: "tx:annotation-driven in applicationContext.xml"
```

**Example (XA detected, scope unresolved):**

```yaml
TransactionRequirement:
  transaction_manager_type: JTA
  distributed_transactions: true
  xa_datasource_detected: true
  legacy_datasource_jndi: "pool/XADataSource"
  server_type: GLASSFISH
  scope: REQUIRES_USER_FEEDBACK
  spring_boot_alternative: "Transactional Outbox (single database) or Saga with compensating actions (multiple systems)"
  legacy_config: "res-type=javax.sql.XADataSource in glassfish-resources.xml"
```

---

### SessionManagementRequirement

Represents how the legacy application server stored and replicated HTTP session state.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `persistence_type` | enum | Yes | Where sessions live: `MEMORY_ONLY`, `REPLICATED`, `MEMORY_WITH_FILESTORE`, `JDBC`, `HA_DEPRECATED` |
| `server_type` | enum | Yes | Source application server. See [Server Types](#server-types). |
| `clustering_enabled` | boolean | No | `true` when the descriptor configures session replication |
| `replication_scope` | enum | No | Replication reach: `instance`, `cluster`, `memory` |
| `replication_mechanism` | string | No | Vendor mechanism (e.g. `"GlassFish HA replication"`, `"Infinispan"`, `"WebLogic in-memory replication"`) |
| `persistence_frequency` | string | No | How often sessions are written (e.g. `"web-method"`, `"time-based"`) |
| `session_timeout_minutes` | integer | No | Session timeout in minutes |
| `cookie_secure` | boolean | No | `Secure` flag on the session cookie |
| `cookie_http_only` | boolean | No | `HttpOnly` flag on the session cookie |
| `file_based_persistence` | boolean | No | `true` when sessions are written to the local filesystem |
| `cloud_foundry_compatible` | boolean | Yes | `false` when the pattern needs one instance or local disk |
| `legacy_config_file` | string | No | Descriptor that holds the configuration (e.g. `"glassfish-web.xml"`) |
| `conflicts` | array[string] | No | Contradictory or deprecated declarations found (see the conflict rules below) |
| `spring_boot_store_type` | enum | No | Recommended Spring Session store: `redis`, `jdbc`, `hazelcast`, `mongodb` |
| `spring_properties` | object | No | Spring Boot property mappings (e.g. `spring.session.store-type`, `server.servlet.session.timeout`) |
| `migration_effort` | enum | No | `LOW`, `MEDIUM`, `HIGH` |

**Cloud Foundry rules (mandatory):**

- Set `cloud_foundry_compatible: false` whenever `file_based_persistence` is `true`. A Cloud Foundry instance has ephemeral local disk, so a file session store loses every session on restage, crash restart, or scale-down.
- Set `cloud_foundry_compatible: false` for `MEMORY_ONLY` when the application stores anything in the session, because more than one instance runs the same code.
- Whenever this entity is present, add `"Session State"` to `ModernizationRequirements.migration_considerations`.
- `REPLICATED` maps to Spring Session with an external store, not to a Spring-internal cluster. Default `spring_boot_store_type` to `redis`.

**Conflict rules:**

- More than one `<session-config>` block, or two different persistence types: record each declaration in `conflicts` and recommend one strategy. This is an error, not a warning.
- `persistence_type: HA_DEPRECATED` (GlassFish `<persistence-type>ha</persistence-type>`): record it in `conflicts` as deprecated and map it to `redis`.

**Example:**

```yaml
SessionManagementRequirement:
  persistence_type: REPLICATED
  server_type: GLASSFISH
  clustering_enabled: true
  replication_scope: instance
  replication_mechanism: "GlassFish HA session replication"
  session_timeout_minutes: 30
  cookie_secure: true
  cookie_http_only: true
  file_based_persistence: false
  cloud_foundry_compatible: false
  legacy_config_file: "glassfish-web.xml"
  spring_boot_store_type: redis
  spring_properties:
    spring.session.store-type: "redis"
    server.servlet.session.timeout: "30m"
    server.servlet.session.cookie.secure: "true"
    server.servlet.session.cookie.http-only: "true"
  migration_effort: MEDIUM
```

---

### CacheRequirement

Represents caching requirements in the application (in-process Spring Cache, Ehcache, JCache/JSR-107, Infinispan, app-server proprietary caching, or session caching) and its migration to Redis on Cloud Foundry.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Cache configuration or entity identifier |
| `source_technology` | enum | Yes | Source caching solution: `SPRING_CACHE`, `EHCACHE`, `JCACHE`, `INFINISPAN`, `APACHE_IGNITE`, `APACHE_JCS`, `HAZELCAST`, `WEBSPHERE_CACHE`, `WEBSPHERE_OBJECT_GRID`, `JBOSS_CACHE`, `SPYMEMCACHED`, `SESSION_STORAGE`, `GENERIC` |
| `source_backend` | string | No | Specific backend provider or implementation (e.g. `"caffeine"`, `"simple"`, `"ehcache"`, `"infinispan"`) |
| `scope` | enum | Yes | Cache scope: `INSTANCE_LOCAL`, `CLUSTERED`, `SESSION_STORAGE`, `DISTRIBUTED`, `REQUIRES_USER_FEEDBACK` |
| `is_distributed` | boolean | Yes | Whether the cache is distributed across instances |
| `cache_names` | array[string] | No | Names of individual caches configured or used |
| `ttl_seconds` | integer | No | Time-to-live expiration in seconds |
| `max_entries` | integer | No | Maximum entries in heap/memory |
| `server_type` | enum | No | Source application server, if vendor-coupled. See [Server Types](#server-types). |
| `cloud_foundry_compatible` | boolean | Yes | Whether current configuration is compatible with Cloud Foundry multi-instance scaling |
| `spring_boot_alternative` | string | No | Recommended Spring Boot configuration (e.g. `"spring.cache.type=redis + spring-boot-starter-data-redis"`) |
| `migration_effort` | enum | No | `LOW`, `MEDIUM`, `HIGH` |
| `migration_notes` | string | No | Additional migration considerations or notes |

**Cloud Foundry Scaling Rule (Mandatory):** In-process caches (`scope: INSTANCE_LOCAL`) and app-server cluster multicast/unicast replication are not compatible with Cloud Foundry horizontal autoscaling and ephemeral instances. All cache instances must externalize state to a shared Redis service instance.

**Example (Spring Cache Caffeine to Redis):**

```yaml
CacheRequirement:
  name: "appCache"
  source_technology: SPRING_CACHE
  source_backend: "caffeine"
  scope: INSTANCE_LOCAL
  is_distributed: false
  cache_names: ["users", "products"]
  ttl_seconds: 600
  max_entries: 500
  cloud_foundry_compatible: false
  spring_boot_alternative: "spring.cache.type=redis with spring-boot-starter-data-redis"
  migration_effort: LOW
```

---

### MonitoringRequirement

Represents how the legacy application produced metrics and health data, and what replaces it.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `metrics_source` | enum | Yes | Where metrics come from: `JMX`, `VENDOR_API`, `VENDOR_PROPERTIES`, `MANAGEMENT_API`, `ACTUATOR`, `CUSTOM` |
| `server_type` | enum | No | Source application server. See [Server Types](#server-types). |
| `mbean_pattern` | string | No | `ObjectName` pattern registered or queried (e.g. `"com.sun.appserv:type=server,*"`) |
| `configuration_source` | string | No | File or class that configures the monitoring (e.g. `"ServerMonitorMBean.java"`) |
| `management_interface_present` | boolean | No | `true` when the code calls a vendor management or admin API |
| `micrometer_enabled` | boolean | No | `true` when the target uses Micrometer |
| `prometheus_export` | boolean | No | `true` when metrics should be exported for Prometheus scraping |
| `per_instance_tagging` | boolean | No | `true` when exported metrics must carry per-instance tags |
| `metrics_tags` | object | No | Tags to attach to exported metrics |
| `requires_centralized_backend` | boolean | No | `true` when per-instance metrics must be aggregated outside the application |
| `spring_boot_alternative` | string | No | Recommended Spring Boot approach |
| `migration_notes` | string | No | Free text for anything the fields do not carry |
| `migration_effort` | enum | No | `LOW`, `MEDIUM`, `HIGH` |

**Cloud Foundry rule (mandatory):** an MBean or a local metrics file describes one instance only, and that instance is replaced without warning. Set `requires_centralized_backend: true` and `per_instance_tagging: true` whenever `metrics_source` is `JMX`, `VENDOR_API` or `MANAGEMENT_API`, and include `CF_INSTANCE_INDEX` and `CF_INSTANCE_GUID` in `metrics_tags`. Without those tags, metrics from several instances merge into one meaningless series.

**Example:**

```yaml
MonitoringRequirement:
  metrics_source: JMX
  server_type: GLASSFISH
  mbean_pattern: "com.sun.appserv:type=server,*"
  configuration_source: "ServerMonitorMBean.java"
  management_interface_present: true
  micrometer_enabled: true
  prometheus_export: true
  per_instance_tagging: true
  metrics_tags:
    cf_instance_index: "${CF_INSTANCE_INDEX}"
    cf_instance_guid: "${CF_INSTANCE_GUID}"
  requires_centralized_backend: true
  spring_boot_alternative: "Spring Boot Actuator + Micrometer MeterRegistry"
  migration_effort: LOW
```

---

### MultiTenancyRequirement

Represents a multi-tenant routing pattern where one deployment serves several tenants from separate datasources or schemas.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `pattern` | enum | Yes | Routing level: `DATASOURCE_ROUTING`, `DATABASE_ROUTING`, `SCHEMA_ROUTING` |
| `datasource_routing_pattern` | enum | No | Grouping: `PER_TENANT`, `PER_GROUP`, `DYNAMIC` |
| `tenant_identifier_source` | enum | No | Where the tenant is read from: `HEADER`, `URL_PATH`, `PRINCIPAL`, `THREADLOCAL` |
| `datasources` | array[string] | No | `JNDIDatasource.name` values, one per tenant |
| `legacy_router_class` | string | No | Class that selects the datasource today |
| `spring_boot_alternative` | string | No | Recommended Spring Boot approach (e.g. `"AbstractRoutingDataSource"`) |
| `migration_effort` | enum | No | `LOW`, `MEDIUM`, `HIGH` |

**Pool budget rule:** each tenant datasource opens its own pool on every instance, so the connection total is `sum(pool sizes) × instance_count`. Compute the per-instance HikariCP size for each [JNDIDatasource](#jndidatasource) against that total, not against a single pool.

**Example:**

```yaml
MultiTenancyRequirement:
  pattern: DATASOURCE_ROUTING
  datasource_routing_pattern: PER_TENANT
  tenant_identifier_source: HEADER
  datasources: ["tenantA_datasource", "tenantB_datasource", "audit_datasource"]
  legacy_router_class: "com.example.TenantDataSourceResolver"
  spring_boot_alternative: "AbstractRoutingDataSource keyed by a tenant resolved from the request"
  migration_effort: MEDIUM
```

---

### ServiceTask

Represents scheduled/cron job requirements.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Task identifier |
| `description` | string | No | Task purpose |
| `schedule_type` | enum | Yes | Schedule type: `CRON`, `FIXED_RATE`, `FIXED_DELAY` |
| `cron_expression` | string | No | Cron expression (for CRON type) |
| `interval_ms` | integer | No | Interval in milliseconds (for FIXED_RATE/FIXED_DELAY) |
| `initial_delay_ms` | integer | No | Initial delay before first execution |
| `operation` | ServiceOperation | No | The operation to execute |
| `concurrency_policy` | enum | No | What to do if previous execution is still running: `SKIP`, `QUEUE`, `PARALLEL` |
| `failure_handling` | string | No | How to handle execution failures |
| `legacy_config` | string | No | Original configuration (e.g., EJB timer, Spring task:scheduled) |

**Example:**

```yaml
ServiceTask:
  name: "dailyReportGenerator"
  description: "Generates daily sales report at midnight"
  schedule_type: CRON
  cron_expression: "0 0 0 * * ?"
  operation:
    name: "generateDailySalesReport"
    type: SCHEDULED
    transactional: true
  concurrency_policy: SKIP
  failure_handling: "Log error and retry next scheduled time"
  legacy_config: "@Schedule annotation in ReportService EJB"
```

---

### ServiceBatchProcess

Represents batch processing requirements (for COBOL batch jobs or Spring Batch migration).

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Batch job identifier |
| `description` | string | No | Job purpose |
| `trigger` | enum | Yes | How the job is triggered: `SCHEDULED`, `EVENT`, `MANUAL`, `FILE_ARRIVAL` |
| `steps` | array[BatchStep] | Yes | Ordered list of job steps |
| `input_sources` | array[BatchDataSource] | No | Input data sources |
| `output_targets` | array[BatchDataSource] | No | Output destinations |
| `restart_policy` | enum | No | Restart behavior: `ALWAYS`, `NEVER`, `ON_FAILURE` |
| `chunk_size` | integer | No | Items processed per chunk/transaction |
| `error_handling` | BatchErrorHandling | No | Error handling configuration |
| `legacy_config` | LegacyBatchConfig | No | Original batch configuration |

**BatchStep Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Step name |
| `type` | enum | Yes | Step type: `CHUNK`, `TASKLET`, `FLOW`, `DECISION` |
| `reader` | string | No | ItemReader description/class |
| `processor` | string | No | ItemProcessor description/class |
| `writer` | string | No | ItemWriter description/class |
| `tasklet` | string | No | Tasklet description (for TASKLET type) |
| `next_step` | string | No | Next step name |
| `on_failure` | string | No | Step to execute on failure |

**BatchDataSource Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `type` | enum | Yes | Source type: `DATABASE`, `FILE`, `API`, `MESSAGE_QUEUE` |
| `location` | string | Yes | Source location (table, file path, endpoint) |
| `format` | string | No | Data format (for files) |
| `query` | string | No | Query or selector (for database/API) |

**BatchErrorHandling Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `skip_limit` | integer | No | Maximum items to skip on error |
| `retry_limit` | integer | No | Maximum retries per item |
| `skippable_exceptions` | array[string] | No | Exceptions that allow skip |
| `retryable_exceptions` | array[string] | No | Exceptions that trigger retry |

**LegacyBatchConfig Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `type` | enum | Yes | Legacy type: `COBOL_JCL`, `SPRING_BATCH_XML`, `EJB_TIMER` |
| `jcl_name` | string | No | JCL job name (for COBOL) |
| `program_name` | string | No | Main program name |

When the batch job was inferred from a standalone COBOL program (no JCL file), omit `jcl_name` and set `program_name` to the COBOL PROGRAM-ID.

**Example:**

```yaml
ServiceBatchProcess:
  name: "monthlyBillingJob"
  description: "Processes monthly billing for all active accounts"
  trigger: SCHEDULED
  steps:
    - name: "readAccounts"
      type: CHUNK
      reader: "JpaPagingItemReader for Account entities"
      processor: "BillingCalculationProcessor"
      writer: "JpaItemWriter for Invoice entities"
      next_step: "generateInvoices"
    - name: "generateInvoices"
      type: TASKLET
      tasklet: "InvoicePdfGeneratorTasklet"
      next_step: "sendNotifications"
    - name: "sendNotifications"
      type: CHUNK
      reader: "Read generated invoices"
      writer: "Send email notifications"
  input_sources:
    - type: DATABASE
      location: "accounts"
      query: "SELECT * FROM accounts WHERE status = 'ACTIVE'"
  output_targets:
    - type: DATABASE
      location: "invoices"
    - type: FILE
      location: "/output/invoices/{date}/"
      format: "PDF"
  chunk_size: 100
  error_handling:
    skip_limit: 10
    retry_limit: 3
    skippable_exceptions: ["DataFormatException"]
    retryable_exceptions: ["TransientDataAccessException"]
  legacy_config:
    type: COBOL_JCL
    jcl_name: "BILLMTHLY"
    program_name: "BILLPROC"
```

---

### ServerMigrationRequirement

Captures the application-server concerns that no other entity holds: the descriptor files to remove, the vendor packages to replace, and special classloading.

**Scope:** the resources a descriptor declares belong in the resource entities, not here. Use [JNDIDatasource](#jndidatasource) for datasources, [MailRequirement](#mailrequirement) for mail sessions, [MessagingRequirement](#messagingrequirement) for JMS resources, [SessionManagementRequirement](#sessionmanagementrequirement) for session configuration, [SecurityRequirement](#securityrequirement) for realms, and [MonitoringRequirement](#monitoringrequirement) for JMX and management APIs. `jndi_mappings` is kept for resources that fit none of those entities.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `source_server` | enum | Yes | Source application server. See [Server Types](#server-types). |
| `target_platform` | string | Yes | Target platform (e.g., `Spring Boot on Cloud Foundry`) |
| `jndi_mappings` | array[JNDIMapping] | No | JNDI resources that no dedicated entity covers. Do not repeat a datasource, mail session, or JMS resource here. |
| `deployment_descriptors` | array[DeploymentDescriptor] | No | Server-specific deployment files |
| `vendor_dependencies` | array[VendorDependency] | No | Vendor-specific code/libraries to replace |
| `classloader_config` | string | No | Special classloader configuration to migrate |
| `clustering_config` | string | No | Clustering/HA configuration to migrate |

**JNDIMapping Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `jndi_name` | string | Yes | Original JNDI name |
| `resource_type` | string | Yes | Resource type |
| `spring_boot_equivalent` | string | Yes | Spring Boot configuration approach |
| `cf_service_binding` | string | No | Cloud Foundry service binding recommendation |

**DeploymentDescriptor Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `file_name` | string | Yes | Descriptor file name |
| `purpose` | string | Yes | What this file configures |
| `action` | enum | Yes | Migration action: `REMOVE`, `MIGRATE_TO_YAML`, `MIGRATE_TO_JAVA_CONFIG` |
| `migration_notes` | string | No | Specific migration instructions |

**VendorDependency Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `package_pattern` | string | Yes | Package pattern (e.g., `org.jboss.*`, `weblogic.*`) |
| `usage_description` | string | Yes | What this code does |
| `replacement` | string | Yes | Spring/standard replacement |

**Example:**

```yaml
ServerMigrationRequirement:
  source_server: WEBLOGIC
  target_platform: "Spring Boot 3.x on Cloud Foundry"
  jndi_mappings:
    - jndi_name: "jdbc/OrdersDS"
      resource_type: "DataSource"
      spring_boot_equivalent: "spring.datasource.* properties or @ConfigurationProperties"
      cf_service_binding: "Bind to p.mysql service instance"
    - jndi_name: "jms/ConnectionFactory"
      resource_type: "JMS ConnectionFactory"
      spring_boot_equivalent: "spring.activemq.* or spring.rabbitmq.*"
      cf_service_binding: "Bind to p-rabbitmq service instance"
  deployment_descriptors:
    - file_name: "weblogic.xml"
      purpose: "WebLogic-specific deployment settings"
      action: REMOVE
      migration_notes: "Context root moves to server.servlet.context-path"
    - file_name: "weblogic-application.xml"
      purpose: "Application-level settings"
      action: MIGRATE_TO_YAML
  vendor_dependencies:
    - package_pattern: "weblogic.jndi.*"
      usage_description: "JNDI lookups"
      replacement: "Remove - use Spring dependency injection"
    - package_pattern: "weblogic.transaction.*"
      usage_description: "Transaction management"
      replacement: "Use @Transactional annotation"
```

---

### ExternalComponentDependency

Captures information about external same-namespace components (from other modules/repositories) that are used by the application but not included in the analyzed source code.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `component_name` | string | Yes | Full package or class name (e.g., `com.example.shared.TaxCalculator`) |
| `component_type` | enum | Yes | Component category: `APPLICATION`, `LIBRARY`, `DTO`, `CLIENT` |
| `source_location` | string | Yes | Local path or git URL provided by user during requirements extraction |
| `generated_spec_location` | string | No | Path where spec was generated (e.g., `../shared-library/spec/`) |
| `invoked_methods` | array[string] | No | Specific methods called from current app (e.g., `["calculateRefund", "calculateVAT"]`) |
| `used_by_stories` | array[string] | No | Story IDs that use this component (e.g., `["US-018", "US-042"]`) |
| `referenced_operations` | array[string] | No | ServiceOperation names extracted from external component |
| `referenced_data_structures` | array[string] | No | DataStructure names extracted from external component |
| `migration_strategy` | enum | Yes | How to handle during translation: `USE_AS_DEPENDENCY`, `COPY_AND_ADAPT`, `REIMPLEMENT` |
| `dependency_coordinates` | DependencyCoordinates | No | Required when migration_strategy = USE_AS_DEPENDENCY (collected during translation, not requirements) |
| `migration_notes` | string | No | Additional context about the dependency |

**DependencyCoordinates Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `group_id` | string | Yes | Maven/Gradle group ID (e.g., `com.example`) |
| `artifact_id` | string | Yes | Maven/Gradle artifact ID (e.g., `shared-library`) |
| `version` | string | Yes | Version to use in Spring Boot 4 project (e.g., `3.0.0`) - collected during translation |
| `compatibility_notes` | string | No | Notes about Spring Boot 4 compatibility |

**Example:**

```yaml
ExternalComponentDependency:
  component_name: "com.example.shared.TaxCalculator"
  component_type: LIBRARY
  source_location: "../shared-library/src/main/java"
  generated_spec_location: "../shared-library/spec/"
  invoked_methods:
    - "calculateRefund"
    - "calculateVAT"
    - "calculateTotal"
  used_by_stories:
    - "US-002"  # Cancel Order
    - "US-003"  # Process Payment
    - "US-015"  # Generate Invoice
  referenced_operations:
    - "calculateRefund"
    - "calculateVAT"
    - "calculateTotal"
  migration_strategy: USE_AS_DEPENDENCY
  # dependency_coordinates populated during translation, not requirements extraction:
  dependency_coordinates:
    group_id: "com.example"
    artifact_id: "shared-library"
    version: "3.0.0"  # Target version for Spring Boot 4
    compatibility_notes: "Version 3.0.0 uses Jakarta namespace, compatible with Spring Boot 4"
  migration_notes: |
    Pure utility library with no external dependencies.
    Can be used as-is in Spring Boot 4 project.
```

**Usage in ModernizationRequirements:**

Add `external_dependencies` array to root-level technical context:

```yaml
ModernizationRequirements:
  # ... existing fields ...
  external_dependencies:
    - component_name: "com.example.shared.TaxCalculator"
      component_type: LIBRARY
      migration_strategy: USE_AS_DEPENDENCY
      # ... rest of fields
```

**Usage in UserStory Technical Context:**

When a story uses an external component, reference it:

```yaml
UserStory:
  technical_context:
    operations:
      - name: cancelOrder
        dependencies:
          - "com.example.shared.TaxCalculator.calculateRefund"  # External operation reference
    external_dependencies:
      - "com.example.shared.TaxCalculator"  # Reference to external component
```

---

### LdapRequirement

Captures LDAP directory access requirements extracted from programmatic JNDI/LDAP code (`InitialLdapContext`, `DirContext`, LDAP controls, event listeners). Provides all fields needed to generate correct `spring.ldap.*` configuration, Spring LDAP beans, and Spring Security LDAP authentication in Spring Boot 4.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Identifier (e.g. `"userDirectoryLdap"`, `"adLdap"`) |
| `description` | string | No | Purpose of this LDAP integration (e.g. "User authentication and group lookup") |
| `provider_url` | string | Yes | LDAP server URL (e.g. `ldap://ldap.example.com:389` or `ldaps://ldap.example.com:636`) |
| `search_base` | string | No | Base DN for directory searches (e.g. `"dc=example,dc=com"`) |
| `bind_dn` | string | No | Service account distinguished name used to bind (e.g. `"cn=admin,dc=example,dc=com"`) |
| `credentials_source` | enum | No | Where the bind password comes from: `ENVIRONMENT`, `VAULT`, `CONFIG`, `HARDCODED`. Prefer `ENVIRONMENT`. |
| `tls_mode` | enum | No | TLS strategy: `NONE` (plain LDAP), `LDAPS` (SSL from start; `ldaps://` URL), `STARTTLS` (TLS upgrade on plaintext connection via `StartTlsRequest`) |
| `referral_handling` | enum | No | How LDAP referrals are handled: `IGNORE`, `FOLLOW`, `THROW`. Defaults to `FOLLOW` when not specified. |
| `search_filter` | string | No | Primary search filter (e.g. `"(uid={0})"`, `"(&(objectClass=person)(uid={0}))"`) |
| `returning_attributes` | array[string] | No | LDAP attributes fetched (e.g. `["uid", "cn", "mail", "memberOf"]`) |
| `authentication_use` | enum | No | Whether LDAP is used for authentication: `BIND` (bind auth), `PASSWORD_COMPARE`, `NONE` (directory queries only) |
| `group_search_base` | string | No | Base DN for group/role searches (e.g. `"ou=groups,dc=example,dc=com"`) |
| `group_search_filter` | string | No | Filter to find groups for a user (e.g. `"member={0}"`) |
| `paging_enabled` | boolean | No | `true` when `PagedResultsControl` is detected |
| `page_size` | integer | No | Page size extracted from `PagedResultsControl` constructor argument |
| `sort_enabled` | boolean | No | `true` when `SortControl` or `SortKey` is detected |
| `sort_attributes` | array[string] | No | Sort attribute names extracted from `SortKey` constructors |
| `event_listeners` | array[string] | No | `NamingListener` implementation class names detected (triggers `migration_considerations: "LDAP Events"`) |
| `ldap_controls` | array[string] | No | Custom `BasicControl` subclass names detected (require manual `DirContextProcessor` migration) |
| `spring_boot_config` | SpringBootConfig | No | Recommended `spring.ldap.*` properties for `application.yml` |
| `spring_boot_recommendation` | string | No | Overall migration recommendation (e.g. `"Use spring-boot-starter-data-ldap + spring-security-ldap"`) |

**TLS mode mapping:**
- `NONE` → plain `ldap://` URL, no additional configuration
- `LDAPS` → use `ldaps://` in `spring.ldap.urls`; ensure server certificate is trusted
- `STARTTLS` → declare `DefaultTlsDirContextAuthenticationStrategy` bean and associate it to `LdapContextSource`; requires server to support the STARTTLS extension

**Event listeners rule:** When `event_listeners` is non-empty, add `"LDAP Events"` to `ModernizationRequirements.migration_considerations`. Spring LDAP has no direct `NamingListener` equivalent; the translator must emit a TODO comment and recommend polling via `@Scheduled` + `LdapTemplate.search()` as an alternative.

**Example:**

```yaml
LdapRequirement:
  name: "userDirectoryLdap"
  description: "User authentication and group lookup against corporate LDAP"
  provider_url: "ldap://ldap.example.com:389"
  search_base: "dc=example,dc=com"
  bind_dn: "cn=svc-app,ou=serviceaccounts,dc=example,dc=com"
  credentials_source: ENVIRONMENT
  tls_mode: STARTTLS
  referral_handling: IGNORE
  search_filter: "(uid={0})"
  returning_attributes: ["uid", "cn", "mail", "memberOf"]
  authentication_use: BIND
  group_search_base: "ou=groups,dc=example,dc=com"
  group_search_filter: "member={0}"
  paging_enabled: true
  page_size: 100
  sort_enabled: false
  spring_boot_config:
    properties:
      spring.ldap.urls: "ldap://ldap.example.com:389"
      spring.ldap.base: "dc=example,dc=com"
      spring.ldap.username: "Service account DN (use environment variable)"
      spring.ldap.password: "${LDAP_PASSWORD}"
      spring.ldap.base-environment.java.naming.referral: "ignore"
    starter_dependency: "spring-boot-starter-data-ldap"
  spring_boot_recommendation: "Use spring-boot-starter-data-ldap (LdapTemplate, LdapContextSource) + spring-security-ldap (LdapBindAuthenticationManagerFactory, DefaultLdapAuthoritiesPopulator). For STARTTLS declare DefaultTlsDirContextAuthenticationStrategy bean."
```

**Example (Active Directory):**

```yaml
LdapRequirement:
  name: "activeDirectoryLdap"
  description: "Active Directory authentication"
  provider_url: "ldap://company.example.com:389"
  search_base: "dc=company,dc=example,dc=com"
  credentials_source: ENVIRONMENT
  tls_mode: NONE
  authentication_use: BIND
  spring_boot_recommendation: "Use ActiveDirectoryLdapAuthenticationProvider(domain, ldapUrl) from spring-security-ldap. Domain-format usernames (user@domain) are handled automatically."
```

---

## Integration Framework Entities (MuleSoft)

These entities capture integration flows, connectors, transformations, and error handling policies from MuleSoft applications migrating to Spring Boot, Spring Integration, and Spring Cloud Stream.

### IntegrationFlowRequirement

Represents an integration flow (HTTP listener, message flow, routing flow, or sub-flow) in MuleSoft.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Flow name or identifier (e.g. `"getOrderFlow"`) |
| `description` | string | No | Business/technical purpose of the flow |
| `flow_type` | enum | Yes | Type of flow: `HTTP_REST`, `EVENT_DRIVEN`, `BATCH`, `ROUTING`, `SCATTER_GATHER`, `SUB_FLOW` |
| `http_method` | enum | No | HTTP method: `GET`, `POST`, `PUT`, `DELETE`, `PATCH` |
| `endpoint_path` | string | No | REST/HTTP endpoint path (e.g. `"/api/orders"`) |
| `server_type` | enum | Yes | Source server/framework: `MULESOFT`. See [Server Types](#server-types). |
| `spring_boot_alternative` | string | No | Recommended Spring Boot approach (e.g. `"@RestController"`, `"Spring Integration Flow"`) |
| `migration_effort` | enum | No | `LOW`, `MEDIUM`, `HIGH` |

**Example:**
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

---

### ConnectorRequirement

Represents inbound or outbound connectors (Database, JMS, Mail, FTP, HTTP client) used in MuleSoft integration flows.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Connector configuration or reference name |
| `connector_type` | enum | Yes | Connector category: `DATABASE`, `JMS`, `MAIL`, `FTP`, `SFTP`, `HTTP_CLIENT`, `KAFKA`, `CUSTOM` |
| `database_type` | enum | No | Target database dialect: `MYSQL`, `POSTGRESQL`, `ORACLE`, `SQL_SERVER`, `GENERIC` |
| `server_type` | enum | Yes | Source server/framework: `MULESOFT`. See [Server Types](#server-types). |
| `operations` | array[string] | No | Operations performed: `["SELECT", "INSERT", "UPDATE", "DELETE", "PUBLISH", "CONSUME", "SEND"]` |
| `config_ref` | string | No | Reference name of global configuration element |
| `spring_boot_alternative` | string | No | Recommended Spring Boot replacement (e.g. `"JdbcTemplate or Spring Data JPA"`, `"Spring Cloud Stream"`) |
| `migration_effort` | enum | No | `LOW`, `MEDIUM`, `HIGH` |

**Example:**
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

---

### TransformationRequirement

Represents data transformation logic implemented via DataWeave (DW 2.0 / 1.0) or custom mappings.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Transformation identifier |
| `transformation_type` | enum | Yes | Transformation mechanism: `DATA_WEAVE`, `XSLT`, `CUSTOM_MAPPER`, `EXPRESSION` |
| `input_format` | enum | No | Input data format: `APPLICATION_JSON`, `APPLICATION_XML`, `TEXT_PLAIN`, `APPLICATION_JAVA`, `MULTIPART` |
| `output_format` | enum | No | Output data format: `APPLICATION_JSON`, `APPLICATION_XML`, `TEXT_PLAIN`, `APPLICATION_JAVA` |
| `server_type` | enum | Yes | Source server/framework: `MULESOFT`. See [Server Types](#server-types). |
| `script_location` | string | No | Path or inline reference to DataWeave script |
| `spring_boot_alternative` | string | No | Recommended Spring replacement (e.g. `"ObjectMapper or custom mapper"`, `"MapStruct"`) |
| `migration_effort` | enum | No | `LOW`, `MEDIUM`, `HIGH` |

**Example:**
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

---

### ErrorHandlingRequirement

Represents error handling strategies, error handlers, and policies (catch-all, error propagate, error continue) in integration flows.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Flow or error handler identifier |
| `error_type` | string | No | Error type matched (e.g. `"DB_ERROR"`, `"ANY"`, `"HTTP:CONNECTIVITY"`) |
| `error_strategy` | enum | Yes | Strategy: `PROPAGATE`, `CONTINUE`, `RETRY`, `FALLBACK` |
| `server_type` | enum | Yes | Source server/framework: `MULESOFT`. See [Server Types](#server-types). |
| `http_status_code` | integer | No | HTTP status code returned on error (e.g. 500, 404) |
| `spring_boot_alternative` | string | No | Recommended Spring Boot approach (e.g. `"@ExceptionHandler"`, `"@RestControllerAdvice"`) |
| `migration_effort` | enum | No | `LOW`, `MEDIUM`, `HIGH` |

**Example:**
```yaml
ErrorHandlingRequirement:
  name: "orderFlow"
  error_type: DB_ERROR
  error_strategy: PROPAGATE
  server_type: MULESOFT
  http_status_code: 500
  spring_boot_alternative: "@ExceptionHandler"
  migration_effort: MEDIUM
```

---

## Server Types

Every `server_type`, `legacy_server` and `source_server` field uses this enum:

| Value | Applies to |
|-------|------------|
| `GLASSFISH` | Oracle GlassFish, Eclipse GlassFish, Payara, and the older Sun Java System Application Server |
| `JBOSS` | JBoss AS / JBoss EAP |
| `WILDFLY` | WildFly |
| `WEBLOGIC` | Oracle WebLogic Server |
| `WEBLOGIC_EXPRESS` | Oracle WebLogic Express |
| `WEBSPHERE` | IBM WebSphere Application Server (traditional) |
| `WEBSPHERE_LIBERTY` | IBM WebSphere Liberty / Open Liberty |
| `TOMCAT` | Apache Tomcat, Apache TomEE |
| `MULESOFT` | MuleSoft Mule runtime |
| `GENERIC` | A Java EE server that the detection patterns cannot identify |

A story that adds the patterns for another server adds its value to this table.

**Rule:** set the same value in `ModernizationRequirements.source_server` and in every entity that carries a
`server_type`. When two servers are detected, choose the one whose deployment descriptor is present and whose
JNDI names the code actually uses, and record the other in `ServerMigrationRequirement.deployment_descriptors`.

---

## UI Layer Entities

These entities capture the presentation layer requirements for legacy applications using JSP, Tiles, Struts, JSF, or plain HTML/JavaScript. Since these technologies are not directly supported in Spring Boot, these requirements guide the migration to modern alternatives (Thymeleaf, REST APIs + SPA, or server-side rendering).

### ViewRequirement

Represents the overall view technology and rendering requirements for the application.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `view_technology` | enum | Yes | Legacy view technology: `JSP`, `TILES`, `STRUTS`, `JSF`, `FACELETS`, `VELOCITY`, `FREEMARKER`, `HTML_JS` |
| `template_locations` | array[string] | No | Paths where view templates are located (e.g., `/WEB-INF/views/`, `/WEB-INF/jsp/`) |
| `layout_framework` | enum | No | Layout/decoration framework: `TILES`, `SITEMESH`, `NONE` |
| `view_resolver_config` | ViewResolverConfig | No | View resolver configuration details |
| `tag_libraries` | array[TagLibrary] | No | Custom or standard tag libraries used |
| `migration_target` | enum | Yes | Target UI approach: `THYMELEAF`, `REST_API_SPA`, `REST_API_MOBILE`, `HTMX`, `KEEP_JSP` |
| `migration_notes` | string | No | Special considerations for UI migration |

**ViewResolverConfig Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `prefix` | string | No | View name prefix (e.g., `/WEB-INF/views/`) |
| `suffix` | string | No | View name suffix (e.g., `.jsp`, `.html`) |
| `view_class` | string | No | View resolver class name |
| `content_type` | string | No | Default content type |

**TagLibrary Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `uri` | string | Yes | Tag library URI |
| `prefix` | string | Yes | Tag prefix used in views |
| `type` | enum | Yes | Library type: `JSTL`, `SPRING_FORM`, `STRUTS`, `TILES`, `JSF`, `CUSTOM` |
| `migration_approach` | string | No | How to handle this tag library in migration |

**Example:**

```yaml
ViewRequirement:
  view_technology: JSP
  template_locations:
    - "/WEB-INF/views/"
    - "/WEB-INF/jsp/"
  layout_framework: TILES
  view_resolver_config:
    prefix: "/WEB-INF/views/"
    suffix: ".jsp"
    view_class: "org.springframework.web.servlet.view.JstlView"
  tag_libraries:
    - uri: "http://java.sun.com/jsp/jstl/core"
      prefix: "c"
      type: JSTL
      migration_approach: "Replace with Thymeleaf th:* attributes"
    - uri: "http://www.springframework.org/tags/form"
      prefix: "form"
      type: SPRING_FORM
      migration_approach: "Replace with Thymeleaf th:field bindings"
    - uri: "http://tiles.apache.org/tags-tiles"
      prefix: "tiles"
      type: TILES
      migration_approach: "Replace with Thymeleaf layout dialect"
  migration_target: THYMELEAF
  migration_notes: "Application uses Tiles for layout composition. Consider Thymeleaf Layout Dialect."
```

---

### ViewComponent

Represents a specific view/page in the application with its UI elements and data requirements.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | View/template name |
| `path` | string | Yes | File path to the view template |
| `type` | enum | Yes | Component type: `PAGE`, `FRAGMENT`, `LAYOUT`, `TILE_DEFINITION`, `INCLUDE` |
| `controller_mapping` | string | No | Controller method that renders this view |
| `model_attributes` | array[ModelAttribute] | No | Data passed to the view from controller |
| `forms` | array[FormDefinition] | No | Forms contained in this view |
| `includes` | array[string] | No | Other views/fragments included |
| `javascript_files` | array[string] | No | JavaScript files used |
| `css_files` | array[string] | No | CSS files used |
| `ajax_endpoints` | array[string] | No | AJAX calls made from this view |
| `client_side_validation` | boolean | No | Whether client-side validation is present |
| `i18n_keys` | array[string] | No | Internationalization message keys used |

**ModelAttribute Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Attribute name in the model |
| `type` | string | Yes | Data type (reference to DataStructure or primitive) |
| `source` | string | No | How this attribute is populated (e.g., service call, session) |
| `required` | boolean | No | Whether this attribute must be present |

**Example:**

```yaml
ViewComponent:
  name: "orderList"
  path: "/WEB-INF/views/orders/list.jsp"
  type: PAGE
  controller_mapping: "OrderController.listOrders()"
  model_attributes:
    - name: "orders"
      type: "List<Order>"
      source: "orderService.findAll()"
      required: true
    - name: "statusFilter"
      type: "OrderStatus"
      source: "request parameter"
      required: false
  forms:
    - name: "searchForm"
      action: "/orders/search"
      method: POST
  includes:
    - "/WEB-INF/views/common/header.jsp"
    - "/WEB-INF/views/common/pagination.jsp"
  javascript_files:
    - "/js/orders.js"
    - "/js/datatables.min.js"
  css_files:
    - "/css/orders.css"
  ajax_endpoints:
    - "GET /api/orders/{id}/status"
  client_side_validation: true
  i18n_keys:
    - "orders.title"
    - "orders.status.pending"
    - "orders.status.shipped"
```

---

### NavigationFlow

Represents the navigation structure and page flows in the application.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Flow name/identifier |
| `description` | string | No | Purpose of this navigation flow |
| `entry_point` | string | Yes | Starting view/endpoint |
| `steps` | array[NavigationStep] | Yes | Ordered steps in the flow |
| `flow_type` | enum | Yes | Flow type: `WIZARD`, `CRUD`, `SEARCH_RESULTS`, `CHECKOUT`, `CUSTOM` |
| `state_management` | enum | No | How state is managed: `SESSION`, `HIDDEN_FIELDS`, `URL_PARAMS`, `SPRING_WEBFLOW`, `NONE` |
| `conversation_scope` | boolean | No | Whether the flow maintains conversational state |
| `error_view` | string | No | View shown on errors |
| `cancel_view` | string | No | View shown on cancellation |

**NavigationStep Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Step name |
| `view` | string | Yes | View component for this step |
| `action` | string | No | Form action or trigger |
| `next_steps` | array[ConditionalNavigation] | No | Possible next steps |
| `validation` | array[string] | No | Validations performed at this step |

**ConditionalNavigation Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `condition` | string | Yes | Condition for this navigation (e.g., "success", "validation_error", "cancel") |
| `target` | string | Yes | Target step or view name |

**Example:**

```yaml
NavigationFlow:
  name: "checkoutFlow"
  description: "Multi-step checkout process for orders"
  entry_point: "/checkout/start"
  flow_type: WIZARD
  state_management: SESSION
  conversation_scope: true
  steps:
    - name: "cart_review"
      view: "checkout/cart"
      action: "/checkout/review"
      next_steps:
        - condition: "continue"
          target: "shipping_info"
        - condition: "cancel"
          target: "/cart"
    - name: "shipping_info"
      view: "checkout/shipping"
      action: "/checkout/shipping"
      validation: ["shippingAddress.notEmpty", "shippingMethod.selected"]
      next_steps:
        - condition: "continue"
          target: "payment_info"
        - condition: "back"
          target: "cart_review"
    - name: "payment_info"
      view: "checkout/payment"
      action: "/checkout/payment"
      validation: ["paymentMethod.valid"]
      next_steps:
        - condition: "submit"
          target: "confirmation"
        - condition: "back"
          target: "shipping_info"
    - name: "confirmation"
      view: "checkout/confirmation"
      action: "/checkout/complete"
      next_steps:
        - condition: "complete"
          target: "/orders/{orderId}"
  error_view: "checkout/error"
  cancel_view: "/cart"
```

---

### FormDefinition

Represents an HTML form with its fields, validation, and submission handling.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Form identifier |
| `action` | string | Yes | Form submission URL |
| `method` | enum | Yes | HTTP method: `GET`, `POST` |
| `enctype` | string | No | Encoding type (e.g., `multipart/form-data` for file uploads) |
| `model_object` | string | No | Backing object type (for Spring form binding) |
| `fields` | array[FormField] | Yes | Form fields |
| `submit_buttons` | array[SubmitButton] | No | Submit buttons and their actions |
| `validation_type` | enum | No | Validation approach: `SERVER_SIDE`, `CLIENT_SIDE`, `BOTH`, `NONE` |
| `csrf_protected` | boolean | No | Whether CSRF token is included |
| `ajax_submit` | boolean | No | Whether form is submitted via AJAX |
| `success_view` | string | No | View shown on successful submission |
| `error_view` | string | No | View shown on validation errors |

**FormField Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Field name (maps to model property) |
| `type` | enum | Yes | Field type: `TEXT`, `PASSWORD`, `EMAIL`, `NUMBER`, `DATE`, `DATETIME`, `SELECT`, `RADIO`, `CHECKBOX`, `TEXTAREA`, `FILE`, `HIDDEN` |
| `label` | string | No | Display label (or i18n key) |
| `required` | boolean | No | Whether the field is required |
| `validations` | array[FieldValidation] | No | Validation rules |
| `options_source` | string | No | Source for select/radio options (e.g., "statusList", "countries") |
| `default_value` | string | No | Default value |
| `placeholder` | string | No | Placeholder text |
| `max_length` | integer | No | Maximum input length |
| `readonly` | boolean | No | Whether the field is read-only |
| `css_class` | string | No | CSS class for styling |

**FieldValidation Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `type` | enum | Yes | Validation type: `REQUIRED`, `MIN_LENGTH`, `MAX_LENGTH`, `PATTERN`, `EMAIL`, `RANGE`, `CUSTOM` |
| `value` | string | No | Validation parameter (e.g., regex pattern, min value) |
| `message` | string | No | Error message (or i18n key) |

**SubmitButton Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Button name/identifier |
| `label` | string | Yes | Button text |
| `action` | string | No | Specific action (if different from form action) |
| `type` | enum | Yes | Button type: `SUBMIT`, `RESET`, `BUTTON` |

**Example:**

```yaml
FormDefinition:
  name: "orderForm"
  action: "/orders/create"
  method: POST
  model_object: "OrderDTO"
  validation_type: BOTH
  csrf_protected: true
  ajax_submit: false
  fields:
    - name: "customerName"
      type: TEXT
      label: "order.customer.name"
      required: true
      validations:
        - type: REQUIRED
          message: "validation.customer.name.required"
        - type: MAX_LENGTH
          value: "100"
          message: "validation.customer.name.maxlength"
      max_length: 100
    - name: "email"
      type: EMAIL
      label: "order.customer.email"
      required: true
      validations:
        - type: EMAIL
          message: "validation.email.invalid"
    - name: "quantity"
      type: NUMBER
      label: "order.quantity"
      required: true
      validations:
        - type: RANGE
          value: "1-1000"
          message: "validation.quantity.range"
      default_value: "1"
    - name: "productId"
      type: SELECT
      label: "order.product"
      required: true
      options_source: "productList"
    - name: "notes"
      type: TEXTAREA
      label: "order.notes"
      max_length: 500
    - name: "attachment"
      type: FILE
      label: "order.attachment"
  submit_buttons:
    - name: "submit"
      label: "order.submit"
      type: SUBMIT
    - name: "saveDraft"
      label: "order.save.draft"
      type: SUBMIT
      action: "/orders/draft"
  success_view: "orders/confirmation"
  error_view: "orders/form"
```

---

### TilesDefinition

Represents Apache Tiles layout definitions for applications using Tiles framework.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Tiles definition name |
| `extends` | string | No | Parent definition name |
| `template` | string | No | Base template path |
| `attributes` | array[TilesAttribute] | Yes | Tiles attributes (regions) |
| `preparer` | string | No | ViewPreparer class for dynamic content |

**TilesAttribute Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Attribute name (e.g., "header", "body", "footer") |
| `value` | string | Yes | Template path or value |
| `type` | enum | Yes | Attribute type: `TEMPLATE`, `STRING`, `DEFINITION` |

**Example:**

```yaml
TilesDefinition:
  name: "orders.list"
  extends: "base.layout"
  attributes:
    - name: "title"
      value: "Order Management"
      type: STRING
    - name: "body"
      value: "/WEB-INF/views/orders/list.jsp"
      type: TEMPLATE
    - name: "sidebar"
      value: "/WEB-INF/views/orders/sidebar.jsp"
      type: TEMPLATE
```

---

### StrutsActionMapping

Represents Struts action mappings for applications using Struts framework.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `path` | string | Yes | Action path (e.g., "/orders/list") |
| `type` | string | Yes | Action class name |
| `name` | string | No | Form bean name |
| `scope` | enum | No | Form bean scope: `REQUEST`, `SESSION` |
| `validate` | boolean | No | Whether to validate form bean |
| `input` | string | No | Input view on validation failure |
| `forwards` | array[StrutsForward] | Yes | Action forwards |
| `parameter` | string | No | Action parameter (for DispatchAction) |

**StrutsForward Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Forward name |
| `path` | string | Yes | Forward path (view or action) |
| `redirect` | boolean | No | Whether to redirect |

**Example:**

```yaml
StrutsActionMapping:
  path: "/orders/create"
  type: "com.example.web.action.CreateOrderAction"
  name: "orderForm"
  scope: REQUEST
  validate: true
  input: "/WEB-INF/pages/orders/form.jsp"
  forwards:
    - name: "success"
      path: "/orders/list.do"
      redirect: true
    - name: "error"
      path: "/WEB-INF/pages/orders/form.jsp"
      redirect: false
    - name: "cancel"
      path: "/orders/list.do"
      redirect: true
  parameter: "method"
```

---

### JSFManagedBean

Represents JSF managed beans for applications using JavaServer Faces.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Bean name (EL expression name) |
| `class` | string | Yes | Fully qualified class name |
| `scope` | enum | Yes | Bean scope: `REQUEST`, `VIEW`, `SESSION`, `APPLICATION`, `CONVERSATION`, `FLOW` |
| `properties` | array[ManagedProperty] | No | Injected properties |
| `action_methods` | array[string] | No | Action methods (return navigation outcomes) |
| `ajax_listeners` | array[string] | No | AJAX event listener methods |

**ManagedProperty Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | Property name |
| `value` | string | Yes | Value expression (e.g., "#{otherBean}") |

**Example:**

```yaml
JSFManagedBean:
  name: "orderBean"
  class: "com.example.web.OrderBean"
  scope: VIEW
  properties:
    - name: "orderService"
      value: "#{orderService}"
  action_methods:
    - "createOrder"
    - "cancelOrder"
    - "searchOrders"
  ajax_listeners:
    - "onProductSelect"
    - "onQuantityChange"
```

---

## Composition Structure

### ModernizationRequirements

Top-level container that aggregates all extracted requirements.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `application_name` | string | Yes | Name of the application being modernized |
| `source_technology` | enum | Yes | Source technology: `SPRING_WEB`, `EJB`, `COBOL`, `STRUTS`, `JSF` |
| `source_server` | enum | No | Application server (for Java apps). See [Server Types](#server-types). |
| `target_platform` | string | Yes | Target platform description |
| `functional_requirements` | array[UserStory] | Yes | Business requirements with technical context |
| `service_endpoints` | array[ServiceEndpoint] | No | All extracted endpoints |
| `service_commands` | array[ServiceCommand] | No | All extracted CLI commands |
| `service_operations` | array[ServiceOperation] | No | All extracted operations |
| `data_structures` | array[DataStructure] | No | All data models |
| `integration_points` | array[IntegrationPoint] | No | External integrations |
| `external_resources` | array[ExternalResourceRequirement] | No | External resources that no dedicated entity covers |
| `jndi_datasources` | array[JNDIDatasource] | No | JNDI-bound datasources with their vendor pool configuration |
| `mail_requirements` | array[MailRequirement] | No | JNDI mail sessions |
| `messaging_requirements` | array[MessagingRequirement] | No | Messaging patterns |
| `security_requirement` | SecurityRequirement | No | Security configuration |
| `transaction_requirement` | TransactionRequirement | No | Transaction configuration |
| `session_management_requirement` | SessionManagementRequirement | No | Session persistence and clustering configuration |
| `cache_requirements` | array[CacheRequirement] | No | Caching solutions found in the source (one entry per distinct cache or backend) |
| `integration_flow_requirements` | array[IntegrationFlowRequirement] | No | Integration flows (MuleSoft), one entry per flow |
| `connector_requirements` | array[ConnectorRequirement] | No | Integration connectors (MuleSoft), one entry per connector configuration |
| `transformation_requirements` | array[TransformationRequirement] | No | Data transformations (MuleSoft DataWeave, XSLT, custom mappers) |
| `error_handling_requirements` | array[ErrorHandlingRequirement] | No | Flow error handling strategies (MuleSoft), one entry per handler |
| `monitoring_requirements` | array[MonitoringRequirement] | No | Metrics and monitoring sources (one entry per distinct source) |
| `multi_tenancy_requirement` | MultiTenancyRequirement | No | Multi-tenant datasource or schema routing |
| `scheduled_tasks` | array[ServiceTask] | No | Scheduled jobs |
| `batch_processes` | array[ServiceBatchProcess] | No | Batch jobs |
| `server_migration` | ServerMigrationRequirement | No | Server-specific migration |
| `ldap_requirements` | array[LdapRequirement] | No | LDAP directory integrations (one entry per distinct LDAP server or usage pattern) |
| `migration_considerations` | array[string] | No | Structured list of migration concern types that apply to this application. When the app stores data in HTTP session (or equivalent), include `"Session State"`. Other values (e.g. `"Server Migration"`, `"UI Migration"`, `"LDAP Events"`, `"JNDI Enumeration"`, `"Distributed Transactions"`, `"Vendor Monitoring"`, `"Caching"`) can be added as extraction identifies them. Translators use this to apply per-topic guidance. |
| `view_requirement` | ViewRequirement | No | UI/view layer configuration |
| `view_components` | array[ViewComponent] | No | Individual views/pages |
| `navigation_flows` | array[NavigationFlow] | No | Page navigation flows |
| `forms` | array[FormDefinition] | No | Form definitions |
| `tiles_definitions` | array[TilesDefinition] | No | Tiles layout definitions (if using Tiles) |
| `struts_actions` | array[StrutsActionMapping] | No | Struts action mappings (if using Struts) |
| `jsf_beans` | array[JSFManagedBean] | No | JSF managed beans (if using JSF) |
| `spring_boot_recommendations` | SpringBootRecommendations | No | Recommended Spring Boot setup |

**SpringBootRecommendations Schema:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `spring_boot_version` | string | Yes | Recommended Spring Boot version |
| `java_version` | string | Yes | Recommended Java version |
| `starters` | array[string] | Yes | Required Spring Boot starters |
| `additional_dependencies` | array[string] | No | Other dependencies needed |
| `cloud_foundry_buildpack` | string | No | Recommended buildpack |
| `manifest_yml` | string | No | Sample manifest.yml content |

**migration_considerations rule:** **Session State** is added when any of the following are detected: HTTP session storage (e.g. `HttpSession` get/setAttribute, `@SessionAttributes`), JSF `@SessionScoped`, EJB `@Stateful`, `NavigationFlow.state_management: SESSION`, or a `SessionManagementRequirement`. **Distributed Transactions** is added when `TransactionRequirement.xa_datasource_detected` is `true`. **Vendor Monitoring** is added when a `MonitoringRequirement` has `metrics_source` of `JMX`, `VENDOR_API`, `VENDOR_PROPERTIES` or `MANAGEMENT_API`. **Caching** is added when a `CacheRequirement` is present. Other consideration types (e.g. from ServerMigrationRequirement, ViewRequirement) can be added to the same array so all migration-related elements are reflected in one place. Existing `migration_notes` (string) on UserStory.technical_context, ViewRequirement, and ServerMigrationRequirement remain for free text; the array is the canonical structured list for translator branching.

---

### UserStory

Represents a business requirement linked to technical context.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `id` | string | Yes | Unique story identifier |
| `title` | string | Yes | Short descriptive title |
| `description` | string | Yes | User story in "As a... I want... So that..." format |
| `acceptance_criteria` | array[string] | Yes | Testable acceptance criteria. In generated specs, criteria may be numbered (e.g. AC-1, AC-2) for traceability to test case **AC Ref** columns. |
| `technical_context` | TechnicalContext | Yes | Technical implementation details |
| `priority` | enum | No | Priority: `HIGH`, `MEDIUM`, `LOW` |
| `complexity` | enum | No | Estimated complexity: `LOW`, `MEDIUM`, `HIGH`, `VERY_HIGH` |

**TechnicalContext Schema:**

**One story, one request:** Each user story must correspond to exactly one service request. So `endpoints` should contain at most one endpoint (the one that handles this story's request); `operations` must include exactly one primary operation (the one invoked by that endpoint, or the single operation for non-HTTP triggers). Additional operation names in `operations` are allowed only as dependencies (operations invoked by the primary operation), not as additional primary operations.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `endpoints` | array[string] | No | References to ServiceEndpoint names. At most one per story (the endpoint that serves this story's request). |
| `operations` | array[string] | No | References to ServiceOperation names. Exactly one primary operation (invoked by the endpoint); optional dependency operation names for traceability. |
| `data_structures` | array[string] | No | References to DataStructure names |
| `integrations` | array[string] | No | References to IntegrationPoint names |
| `source_files` | array[string] | No | Legacy source files implementing this story |

**Note:** `migration_notes` field is deprecated. Migration concerns should be captured at the root level in `ModernizationRequirements.migration_considerations` array, not per-story. User stories focus on requirements (what the system does), not migration strategies (how to modernize).

**Example:**

```yaml
UserStory:
  id: "US-001"
  title: "Cancel Order"
  description: |
    As an administrator,
    I want to cancel an existing order,
    So that I can handle customer refund requests and order issues.
  acceptance_criteria:
    - "Only users with ADMIN role can cancel orders"
    - "Order status is updated to CANCELLED"
    - "Customer receives cancellation notification via email"
    - "Cancelled orders cannot be modified further"
    - "The operation is atomic - partial cancellation is not allowed"
  technical_context:
    endpoints: ["cancelOrderEndpoint"]
    operations: ["cancelOrder", "sendCancellationNotification"]  # cancelOrder = primary; sendCancellationNotification = dependency (AC describe primary only)
    data_structures: ["Order", "OrderStatus", "CancellationNotification"]
    integrations: ["notification-queue"]
    source_files:
      - "com/example/ejb/OrderServiceBean.java"
      - "com/example/web/OrderController.java"
  priority: HIGH
  complexity: MEDIUM
```

---

## Complete Schema Reference

Below is the complete schema in YAML format for reference:

```yaml
ModernizationRequirements:
  application_name: string
  source_technology: enum [SPRING_WEB, EJB, COBOL, STRUTS, JSF]
  source_server: enum [GLASSFISH, JBOSS, WILDFLY, WEBLOGIC, WEBLOGIC_EXPRESS, WEBSPHERE, WEBSPHERE_LIBERTY, TOMCAT, MULESOFT, GENERIC]
  target_platform: string
  
  functional_requirements:
    - UserStory
  
  service_endpoints:
    - ServiceEndpoint
  
  service_commands:
    - ServiceCommand
  
  service_operations:
    - ServiceOperation
  
  data_structures:
    - DataStructure
  
  integration_points:
    - IntegrationPoint
  
  external_resources:
    - ExternalResourceRequirement
  
  jndi_datasources:
    - JNDIDatasource
  
  mail_requirements:
    - MailRequirement
  
  messaging_requirements:
    - MessagingRequirement
  
  security_requirement: SecurityRequirement
  
  transaction_requirement: TransactionRequirement
  
  session_management_requirement: SessionManagementRequirement

  cache_requirements:
    - CacheRequirement
  
  monitoring_requirements:
    - MonitoringRequirement
  
  multi_tenancy_requirement: MultiTenancyRequirement
  
  scheduled_tasks:
    - ServiceTask
  
  batch_processes:
    - ServiceBatchProcess
  
  server_migration: ServerMigrationRequirement

  ldap_requirements:
    - LdapRequirement  # If using LDAP/JNDI LDAP access

  # Integration framework (MuleSoft)
  integration_flow_requirements:
    - IntegrationFlowRequirement

  connector_requirements:
    - ConnectorRequirement

  transformation_requirements:
    - TransformationRequirement

  error_handling_requirements:
    - ErrorHandlingRequirement
  
  migration_considerations: array[string]  # e.g. ["Session State", "Server Migration", "LDAP Events", "Distributed Transactions", "Vendor Monitoring", "Caching"]
  
  # UI Layer
  view_requirement: ViewRequirement
  
  view_components:
    - ViewComponent
  
  navigation_flows:
    - NavigationFlow
  
  forms:
    - FormDefinition
  
  tiles_definitions:
    - TilesDefinition  # If using Tiles
  
  struts_actions:
    - StrutsActionMapping  # If using Struts
  
  jsf_beans:
    - JSFManagedBean  # If using JSF
  
  spring_boot_recommendations: SpringBootRecommendations
```

---

## Usage Guidelines

### For Requirements Extraction (Phase 1)

1. Use [detection patterns](detection-patterns/) to identify legacy constructs in source code
2. Map detected patterns to the appropriate schema entities
3. Generate UserStory entries with acceptance criteria derived from code behavior
4. Populate TechnicalContext with references to extracted entities

### For Spring Boot Translation (Phase 2)

1. Use ServiceEndpoint to generate `@RestController` classes
2. Use ServiceOperation to generate `@Service` methods with appropriate `@Transactional` settings
3. Use DataStructure to generate `@Entity` classes and Spring Data repositories
4. Use ExternalResourceRequirement to generate `application.yml` configuration
5. Use JNDIDatasource to generate `spring.datasource.*` + HikariCP properties and the Cloud Foundry service binding. Recompute the pool size per instance; never copy the vendor number.
6. Use MailRequirement to generate `spring.mail.*` properties and a `JavaMailSender` bean
7. Use SessionManagementRequirement to generate `spring.session.*` properties with an external store (Redis by default)
8. Use MonitoringRequirement to generate Spring Boot Actuator + Micrometer configuration with per-instance tags
9. Use MultiTenancyRequirement to generate an `AbstractRoutingDataSource`
10. Use TransactionRequirement `scope` to decide the transaction strategy. Stop and ask the user while `scope` is `REQUIRES_USER_FEEDBACK`.
11. Use SecurityRequirement `realm_type` and `role_mappings` to generate a `SecurityFilterChain`
12. Use ServerMigrationRequirement to identify code that needs replacement
13. Use SpringBootRecommendations to generate `pom.xml` and `manifest.yml`
14. Use `migration_considerations` to apply per-topic translation guidance (e.g. Session State → Spring Session with Redis)

### For UI Layer Migration

Since JSP, Tiles, Struts, and JSF are not directly supported in Spring Boot, use the UI layer entities to guide migration:

1. **ViewRequirement** - Determine the migration strategy:
   - `THYMELEAF`: Server-side rendering with Thymeleaf templates
   - `REST_API_SPA`: Convert to REST APIs + Single Page Application (React, Angular, Vue)
   - `REST_API_MOBILE`: REST APIs for mobile clients
   - `HTMX`: Server-side with HTMX for dynamic updates

2. **ViewComponent** - For each view:
   - Extract model attributes → DTO classes
   - Extract forms → REST endpoints or Thymeleaf forms
   - Identify AJAX calls → REST API endpoints
   - Migrate JavaScript/CSS assets

3. **NavigationFlow** - Convert page flows:
   - Wizard flows → Multi-step forms or SPA routes
   - Spring WebFlow → State machines or SPA state management
   - Session state → Stateless design with tokens or client-side state

4. **FormDefinition** - For each form:
   - Generate DTO classes with validation annotations
   - Create REST endpoints (POST/PUT) or Thymeleaf form controllers
   - Migrate validations to Bean Validation (`@NotNull`, `@Size`, etc.)
   - Convert client-side validation to modern frameworks

5. **Framework-Specific Migration**:
   - **Tiles**: Replace with Thymeleaf Layout Dialect or component-based architecture
   - **Struts**: Convert Actions to `@RestController` or `@Controller` methods
   - **JSF**: Convert managed beans to `@Service` + `@RestController`, facelets to Thymeleaf or SPA
