# Spring Boot 4 Translation: SecurityRequirement

Map **SecurityRequirement** to Spring Security 7 (component-based configuration). For the core translation guide, see [../SKILL.md](../SKILL.md).

## Patterns

- **Authentication:** Map authentication_method to form login, BASIC, Bearer, OAuth2, etc. Use `SecurityFilterChain` and `UserDetailsService`.
- **Authorization:** `@PreAuthorize("hasRole('ADMIN')")` on controllers; map protected_endpoints to path matchers and access rules.
- **Roles:** Map role definitions to authorities; use `hasRole`, `hasAnyRole` in `@PreAuthorize`.

## Application-Server Realms

When `realm_type` and `server_type` are set, the authentication came from the application server, not from
the application. The server is gone in the target, so the realm must be replaced, not configured.

| `realm_type` / `authentication_method` | Target |
|----------------------------------------|--------|
| `FileRealm` / `FILE_REALM` | The user list lived inside the server and is **not** in the repository. Generate the `SecurityFilterChain` and the role rules from `legacy_config.role_mappings`, and use an `InMemoryUserDetailsManager` **only** as a placeholder marked `TODO`. Tell the user the target needs a real user store (a database table with `JdbcUserDetailsManager`, LDAP, or an identity provider). Never invent credentials. |
| `LDAPRealm` / `LDAP` | Load [translation-spring-boot-4-ldap.md](translation-spring-boot-4-ldap.md). The realm properties are already in `LdapRequirement`. Convert the filter placeholder: an application-server filter uses `%s`, Spring Security uses `{0}`. |
| `CertificateRealm` / `CERTIFICATE` | `http.x509(...)` with a `UserDetailsService` keyed on the certificate subject. Map `assign-groups` to the authorities every valid certificate holder receives. |
| `JAAS_LOGIN_MODULE`, `ELYTRON` | Read `legacy_config.realm_properties` for what the login module actually did (a database query, a directory bind, a custom check), then implement that as an `AuthenticationProvider`. Do not port the login module itself. |

- `<security-role-mapping>` maps a server principal or group to an application role. Generate the role rules
  from it, and record that the principal-to-role mapping now needs a source in the target.
- Read `protected_endpoints` for the URL rules that came from `<security-constraint>` in `web.xml`.
- A password found in a descriptor is a leaked credential. Never copy it into `application.yml`. Read it from
  the environment and say so.

### mTLS on Cloud Foundry

When `mtls_required: true`, the platform router terminates TLS before the request reaches the container, so
`server.ssl.client-auth` in the application does not see the client certificate. The certificate arrives in a
forwarded header instead. Do not generate a key-store-based mTLS configuration and claim it works — state the
constraint, and flag the pattern for manual review with the platform team.

For a worked example that includes a security configuration, see [translation-spring-boot-4-example-rest-crud.md](translation-spring-boot-4-example-rest-crud.md).
