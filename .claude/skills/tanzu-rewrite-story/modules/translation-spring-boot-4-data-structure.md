# Spring Boot 4 Translation: DataStructure

Map **DataStructure** (entity, DTO, enum) to JPA entities, Java records, and enums. For the core translation guide, see [../SKILL.md](../SKILL.md).

## Patterns

- **Entity:** `@Entity`, `@Table(name)`, `@Id`, `@GeneratedValue`, `@Column`, `@ManyToOne`/`@OneToMany`/etc. with `@JoinColumn`. Use JSpecify `@NonNull`/`@Nullable` on getters. Map fields to columns; relationships to JPA associations.
- **DTO:** Java records with Bean Validation (`@NotNull`, `@Size`, `@Email`) and JSpecify. For DTOs with entity-typed fields, use `loaded_shape` when present to define which properties are loaded.
- **Jackson 3.x:** Use `tools.jackson` package; configure `JsonMapper` if needed. Records serialize by default.
- **Enum:** Java enum; use `@Enumerated(EnumType.STRING)` when persisting.

For a worked entity and DTO example, see [translation-spring-boot-4-example-rest-crud.md](translation-spring-boot-4-example-rest-crud.md).
