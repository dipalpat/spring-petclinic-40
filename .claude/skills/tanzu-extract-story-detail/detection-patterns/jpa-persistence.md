# JPA / Jakarta Persistence Patterns

This document defines detection patterns for extracting entity properties (fields and relationships) from Javax/Jakarta persistence annotations. Use these patterns whenever the application is **Spring Boot**, **Spring Web**, or **EJB** and JPA entities are present.

**When to use:** Always run when Spring Boot, Spring Web, or EJB is detected. See [the skill body](../SKILL.md) and [usage-guidelines.md](usage-guidelines.md#detection-workflow).

**Namespace:** Both `javax.persistence` and `jakarta.persistence` use the same annotation names. Apply the same extraction rules to both; only use javax vs jakarta elsewhere (e.g. Boot version). Map extracted data to the Technical Context Schema: [DataStructure](../technical-context-schema.md#datastructure), Field, Relationship, UniqueConstraint.

## Table of Contents

- [Entity Detection](#entity-detection)
- [@Table](#table)
- [@Column and Simple Fields](#column-and-simple-fields)
- [@Id and @GeneratedValue](#id-and-generatedvalue)
- [Relationship Annotations](#relationship-annotations)
- [@Embedded and @Embeddable](#embedded-and-embeddable)
- [@Enumerated and @Temporal](#enumerated-and-temporal)
- [Complete Example](#complete-example)

---

## Entity Detection

### @Entity Annotation

| Pattern | `@Entity` or `javax\.persistence\.Entity|jakarta\.persistence\.Entity` |
|---------|-------------------------------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | JPA entity class |
| **Schema Entity** | `DataStructure` with `persistence_type = ENTITY` |

**Example:**
```java
import javax.persistence.Entity;   // or jakarta.persistence.Entity

@Entity
public class Order {
    // ...
}
```

**Extraction Notes:**
- Class annotated with `@Entity` → create a `DataStructure` with `persistence_type: ENTITY`, `name` = class name.
- Populate `fields` and `relationships` using the patterns below (e.g. @Table, @Column, @Id, relationship annotations).

---

## @Table

| Pattern | `@Table\s*\(` |
|---------|----------------|
| **File Types** | `*.java` |
| **Indicates** | Table name and optional schema/catalog/constraints |
| **Schema Entity** | `DataStructure.table_name`, `DataStructure.unique_constraints` |

**Example:**
```java
@Entity
@Table(name = "orders", schema = "app", uniqueConstraints = {
    @UniqueConstraint(columnNames = {"order_number"})
})
public class Order { ... }
```

**Extraction Notes:**
- `@Table(name = "…")` → `DataStructure.table_name`.
- `@Table(schema = "…")` / `catalog = "…"` → document or note for translation (schema allows optional convention; no separate schema/catalog in current DataStructure).
- `@Table(uniqueConstraints = { ... })` → `DataStructure.unique_constraints`. For each `@UniqueConstraint`, add an entry with `name` (constraint name if present) and `fields` (list of **field names** corresponding to `columnNames`; map column names to Java field names when they differ via @Column).

---

## @Column and Simple Fields

| Pattern | `@Column\s*\(` |
|---------|----------------|
| **File Types** | `*.java` |
| **Indicates** | Column mapping and constraints for a persistent field |
| **Schema Entity** | `Field` (within `DataStructure.fields`) |

**Example:**
```java
@Column(name = "order_number", nullable = false, unique = true, length = 50)
private String orderNumber;

@Column(precision = 10, scale = 2)
private BigDecimal totalAmount;

@Column(insertable = false, updatable = false)
private String computedStatus;
```

**Extraction Notes:**
- Java field name → `Field.name`; Java type → `Field.type` (primitive, wrapper, enum, or reference to another DataStructure).
- `@Column(name = "…")` → `Field.column_name` (omit if same as field name).
- `@Column(nullable = …)` → `Field.nullable`.
- `@Column(unique = true)` → `Field.unique`.
- `@Column(length = …)` → `Field.length`.
- `@Column(precision = …, scale = …)` → `Field.precision`, `Field.scale`.
- `@Column(insertable = false, updatable = false)` → often used for derived/read-only; consider `Field.derived: true` and optional `Field.derivation_rule` if inferable.
- Bean Validation / JSR-380 on the same field (e.g. `@NotNull`, `@Size`, `@NotBlank`) → `Field.validations` (list of constraint names or short form, e.g. `["NotNull", "Size(max=50)"]`).

---

## @Id and @GeneratedValue

| Pattern | `@Id` |
|---------|-------|
| **File Types** | `*.java` |
| **Indicates** | Primary key field |
| **Schema Entity** | `Field.primary_key`, optional generation note |

**Example:**
```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

**Extraction Notes:**
- `@Id` → `Field.primary_key: true`. Ensure the entity has at least one primary key field (or use `DataStructure.primary_key_fields` for composite keys).
- `@GeneratedValue(strategy = …)` → note for translation: `GenerationType.IDENTITY` → database identity; `SEQUENCE` → sequence; `TABLE` → table generator; `AUTO` → provider default. No separate schema field required unless you add a convention for generation strategy.

---

## Relationship Annotations

### @ManyToOne / @OneToOne (owning side)

| Pattern | `@ManyToOne|@OneToOne` with `@JoinColumn` |
|---------|--------------------------------|
| **File Types** | `*.java` |
| **Indicates** | Single-valued association; owning side holds FK |
| **Schema Entity** | `DataStructure.relationships` (Relationship), and optionally a field reference |

**Example:**
```java
@ManyToOne(fetch = FetchType.LAZY, optional = false)
@JoinColumn(name = "customer_id")
private Customer customer;
```

**Extraction Notes:**
- Add a `Relationship` entry: `type: MANY_TO_ONE` or `ONE_TO_ONE`; `field_name` = Java field name; `target` = referenced entity (DataStructure name, e.g. `Customer`).
- `@JoinColumn(name = "…")` → `Relationship.join_column` (e.g. `"customer_id"`).
- `fetch` → `Relationship.fetch` (LAZY or EAGER).
- `optional` → `Relationship.optional` (e.g. `false` when `optional = false`).

### @OneToMany / @ManyToMany (inverse/collection side)

| Pattern | `@OneToMany|@ManyToMany` |
|---------|---------------------------|
| **File Types** | `*.java` |
| **Indicates** | Collection-valued association; may use `mappedBy` or join table |
| **Schema Entity** | `DataStructure.relationships` (Relationship) |

**Example:**
```java
@OneToMany(mappedBy = "order", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
private List<OrderItem> items;

@ManyToMany
@JoinTable(name = "order_tags", joinColumns = @JoinColumn(name = "order_id"),
           inverseJoinColumns = @JoinColumn(name = "tag_id"))
private Set<Tag> tags;
```

**Extraction Notes:**
- Add a `Relationship` entry: `type: ONE_TO_MANY` or `MANY_TO_MANY`; `field_name` = Java field name; `target` = referenced entity (e.g. `OrderItem`, `Tag`).
- `mappedBy = "…"` → `Relationship.mapped_by` (e.g. `"order"`).
- `cascade` → `Relationship.cascade` (e.g. `["ALL"]` or `["PERSIST", "MERGE", "REMOVE"]`).
- `fetch` → `Relationship.fetch`.
- `@JoinTable(name = "…")` → `Relationship.join_table`. Optionally note `joinColumns` / `inverseJoinColumns` for translation.

---

## @Embedded and @Embeddable

| Pattern | `@Embedded|@Embeddable` |
|---------|------------------------|
| **File Types** | `*.java` |
| **Indicates** | Embeddable component; fields stored in parent table |
| **Schema Entity** | `DataStructure` with `persistence_type: EMBEDDABLE`, or flattened fields in parent |

**Example:**
```java
@Embedded
private Address address;

@Embeddable
public static class Address {
    private String street;
    private String city;
    @Column(name = "zip_code")
    private String zipCode;
}
```

**Extraction Notes:**
- Create a `DataStructure` for the embeddable class with `persistence_type: EMBEDDABLE` and its `fields` (including @Column mappings). Reference it from the parent entity's `fields` (e.g. type = embeddable structure name), or document that embeddable fields are expanded into the parent's field list with a naming convention (e.g. prefix `address.` or column names). Prefer one approach consistently; see [Technical Context Schema](../technical-context-schema.md#datastructure) for Field/EMBEDDABLE usage.

---

## @Enumerated and @Temporal

| Pattern | `@Enumerated|@Temporal` |
|---------|------------------------|
| **File Types** | `*.java` |
| **Indicates** | Enum or date/time storage semantics |
| **Schema Entity** | `Field.type`, optional note for translation |

**Example:**
```java
@Enumerated(EnumType.STRING)
private OrderStatus status;

@Temporal(TemporalType.TIMESTAMP)
private Date createdAt;
```

**Extraction Notes:**
- `@Enumerated(EnumType.STRING)` / `ORDINAL` → keep `Field.type` as the enum name (e.g. `OrderStatus`); optionally note storage type (STRING vs ORDINAL) for translation.
- `@Temporal(TemporalType.DATE|TIME|TIMESTAMP)` → keep type (e.g. `Date`, `LocalDateTime`); optionally add description for date/time semantics for translation.

---

## Complete Example

**Source (javax or jakarta):**
```java
@Entity
@Table(name = "orders", uniqueConstraints = {
    @UniqueConstraint(name = "uk_order_number", columnNames = {"order_number"})
})
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "order_number", unique = true, nullable = false, length = 20)
    private String orderNumber;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    private Customer customer;
}
```

**Extracted DataStructure (YAML):**
```yaml
DataStructure:
  name: "Order"
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
    - name: orderNumber
      type: String
      column_name: "order_number"
      unique: true
      nullable: false
      length: 20
  relationships:
    - type: MANY_TO_ONE
      target: Customer
      field_name: customer
      join_column: "customer_id"
      fetch: LAZY
```

---

## Related Documentation

- [Technical Context Schema](../technical-context-schema.md) – DataStructure, Field, Relationship, UniqueConstraint
- [SKILL.md](../SKILL.md) – Per-Story Extraction Steps, step 4 (Map Technical Context)
- [spring-web.md](spring-web.md) – Data Access Patterns (entity detection; full extraction rules here)
- [ejb.md](ejb.md) – Persistence Context; entity structure extracted using this file
