# Data Model Template

> **Review this document LAST**, after validating all user stories.
> Each entity includes a "Used By" section showing which stories depend on it.

## Overview

Core data structures and entities used across the application. Each entity has its own UML class diagram showing only that entity and its direct relationships (no global diagram).

---

## Entities

### {EntityName}

**Description**: {what this entity represents}

**Table**: `{table_name}`

**Relationship diagram**: (UML class diagram showing this entity and its related entities only. If no relationships, omit the diagram or write "No relationships".)

```mermaid
classDiagram
    class {EntityName} {
        +id: key_type
        +{key_field}
    }
    class {RelatedEntityA} { }
    class {RelatedEntityB} { }
    {EntityName} "1" --> "*" {RelatedEntityA} : {relationship_label}
    {EntityName} "1" --> "1" {RelatedEntityB} : {relationship_label}
```

**Used By (User Stories):**

> Review the user stories below BEFORE validating this entity.
> This mapping helps ensure schema changes don't break dependent functionality.

| Story ID | Usage | Domain |
|----------|-------|--------|
| [US-{NNN}](../domains/{domain}/US-{NNN}-{slug}.md) | Created/Updated | {domain_name} |
| [US-{NNN}](../domains/{domain}/US-{NNN}-{slug}.md) | Read (display) | {domain_name} |
| [US-{NNN}](../domains/{domain}/US-{NNN}-{slug}.md) | Deleted | {domain_name} |

```yaml
DataStructure:
  name: "{EntityName}"
  description: "{description}"
  persistence_type: ENTITY
  table_name: "{table_name}"
  fields:
    - name: {field_name}
      type: {field_type}
      column_name: "{column_name}"
      nullable: {true|false}
      unique: {true|false}
      length: {max_length}
      validations: ["{validation_rules}"]
      description: "{field_purpose}"
  relationships:
    - type: {ONE_TO_ONE|ONE_TO_MANY|MANY_TO_ONE|MANY_TO_MANY}
      target: {TargetEntity}
      field_name: {field_name}
      join_column: "{fk_column}"
      cascade: ["{cascade_types}"]
      fetch: {LAZY|EAGER}
  auditing:
    created_date_field: {field_name}
    modified_date_field: {field_name}
```

**Source Files (Legacy)**: `{path/to/Entity.java}`

---

## Enumerations

### {EnumName}

**Description**: {what this enum represents}

```yaml
DataStructure:
  name: "{EnumName}"
  persistence_type: NONE
  fields:
    - name: {VALUE_1}
      description: "{meaning}"
    - name: {VALUE_2}
      description: "{meaning}"
```

**State Transitions** (if applicable):

```mermaid
stateDiagram-v2
    [*] --> {State1}
    {State1} --> {State2}: {trigger}
    {State2} --> [*]
```

---

## DTOs

### {DTOName}

**Description**: {purpose of this DTO}

```yaml
DataStructure:
  name: "{DTOName}"
  persistence_type: DTO
  fields:
    - name: {field_name}
      type: {field_type}
      loaded_shape: ["{path1}", "{path2}"]   # when type is entity or List<Entity>
      validations: ["{validation_rules}"]
```

When a field's type is an entity (or collection of entity), use `loaded_shape` to list which properties/relationships are loaded for that entity.
