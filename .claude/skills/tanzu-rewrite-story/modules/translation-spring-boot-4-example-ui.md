# Full Example: UI Migration

There is no worked UI example in this skill yet. Use
[translation-spring-boot-4-ui.md](translation-spring-boot-4-ui.md) for the `ViewRequirement` and
`FormDefinition` patterns, and [translation-spring-boot-4-example-rest-crud.md](translation-spring-boot-4-example-rest-crud.md)
for the controller, service, repository and test shape that a Thymeleaf `@Controller` also follows.

The two migration targets differ:

- `migration_target: THYMELEAF` — a `@Controller` returning a view name, Thymeleaf templates under
  `src/main/resources/templates/`, the Layout Dialect in place of Tiles, `th:object` / `th:field` forms with
  `BindingResult`, and `messages.properties` for anything the JSP read from a resource bundle.
- `migration_target: REST_API_SPA` — a `@RestController` per resource, plus a separate front end. Add CORS
  configuration and a forwarding controller for client-side routes.

For the core translation guide, see [../SKILL.md](../SKILL.md).
