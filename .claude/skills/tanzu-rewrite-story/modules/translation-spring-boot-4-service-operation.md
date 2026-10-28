# Spring Boot 4 Translation: ServiceOperation

Map **ServiceOperation** (steps, preconditions, structural_events) to `@Service` methods with `@Transactional`. For the core translation guide, see [../SKILL.md](../SKILL.md).

## Patterns

- **Steps with variables:** Get steps introduce variables (e.g. "Get Order as $order (by orderId)"); later steps reference them. Map to service method logic: load entity, update, save, publish event. When a step uses explicit output assignments (e.g. "Set output.visitatori = $reservation.visitatori to load the visitors"), translate each assignment to populating the DTO/view model field from the corresponding variable; " to load the …" indicates that the source path (e.g. $reservation.visitatori) must be loaded (e.g. via JPA fetch or repository). The output type and field names are in the operation's `output` DataStructure.
- **Structured output (one step per output property):** For operations that return data, the spec uses one step per output field in the form **Set output.\<field\> = \<source\>** [** to load the \<description\>**]. Use these steps and the operation's `output` DataStructure (and `output_shape` when present) to build the return value: each step maps to one DTO/view model field and its source; do not invent or omit fields. When a DTO field's type is an entity (or collection), use the field's `loaded_shape` if present for JPA projections or DTO constructors.
- **Preconditions:** Each precondition (description, condition, inputs_used, failure_exception) → validation or guard that throws the specified exception. Do not put "Validate X" in steps; use preconditions.
- **Structural events:** `ENTITY_CREATED` → repository.save(new entity); `ENTITY_UPDATED` → update instance variable then save; `ENTITY_DELETED` → repository.delete; `MESSAGE_SENT` → event publisher or JMS. Use `instance_variable` (e.g. `$order`) when the effect applies to an instance from a Get step. For CREATE/UPDATE, use `value_sources` (array of input roots) and `attributes` (map: entity attribute → qualified path `value_source_root.property`) to map which input fields set which entity properties; each mapping value is a qualified path (e.g. `prenotazioneForm.dataOra`, `mdzDatePrenotazioni.data`).
- **Transactional:** Map `transactional`, `transaction_propagation`, `rollback_for`, `isolation`, `timeout` to `@Transactional` attributes.
- **Retry / resilience:** When spec has retry or error handling, use `@Retryable` or `@ConcurrencyLimit` as in full doc.

For a worked example, see [translation-spring-boot-4-example-rest-crud.md](translation-spring-boot-4-example-rest-crud.md).
