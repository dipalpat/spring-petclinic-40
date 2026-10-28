# Spring Boot 4 Translation: Cloud Foundry

Load this module when generating application.yml and manifest.yml. For the core translation guide, see [../SKILL.md](../SKILL.md).

---

## Cloud Foundry Deployment

### manifest.yml Template

```yaml
applications:
  - name: ${APP_NAME:my-application}
    memory: ${MEMORY:1G}
    instances: ${INSTANCES:2}
    path: target/${APP_NAME}-${VERSION}.jar
    
    buildpacks:
      - java_buildpack_offline
    
    env:
      # JVM Configuration
      JBP_CONFIG_OPEN_JDK_JRE: '{ jre: { version: 25.+ } }'
      JAVA_OPTS: >-
        -XX:+UseZGC
        -XX:+ZGenerational
        -XX:MaxRAMPercentage=75.0
        -Dspring.profiles.active=cloud
      
      # Application Configuration
      SPRING_APPLICATION_NAME: ${APP_NAME}
      
    services:
      - ${APP_NAME}-db
      - ${APP_NAME}-mq
      - config-server
    
    routes:
      - route: ${APP_NAME}.${CF_DOMAIN}
      - route: ${APP_NAME}.apps.internal
    
    health-check-type: http
    health-check-http-endpoint: /actuator/health
    timeout: 180
```

### ExternalResourceRequirement (DATASOURCE) and HikariCP

When translating `ExternalResourceRequirement` with `type: DATASOURCE`, apply connection pool requirements to Spring Boot configuration so that HikariCP is configured correctly.

**When to apply:** For each `ExternalResourceRequirement` of type `DATASOURCE`, if the extracted spec includes `connection_pool` and/or `spring_boot_config.properties` with HikariCP keys, generate the corresponding `application.yml` (or profile-specific) entries under `spring.datasource.hikari`.

**Mapping from `connection_pool`:**

| connection_pool field     | Spring Boot property                              |
|---------------------------|---------------------------------------------------|
| `maximum_pool_size`       | `spring.datasource.hikari.maximum-pool-size`      |
| `minimum_idle`            | `spring.datasource.hikari.minimum-idle`           |
| `connection_timeout_ms`   | `spring.datasource.hikari.connection-timeout`     |
| `idle_timeout_ms`         | `spring.datasource.hikari.idle-timeout`            |
| `max_lifetime_ms`         | `spring.datasource.hikari.max-lifetime`            |
| `auto_commit`             | `spring.datasource.hikari.auto-commit`             |

If `spring_boot_config.properties` already contains `spring.datasource.hikari.*` keys, use those values in the generated YAML. If no `connection_pool` or HikariCP properties are in the requirements, omit or use Spring Boot defaults (no change to current behavior).

**Example** (when requirements specify pool settings):

```yaml
spring:
  datasource:
    # url, username, password are injected automatically by java-cfenv-boot from VCAP_SERVICES — do not set them here
    hikari:
      maximum-pool-size: 20      # from connection_pool.maximum_pool_size or requirements
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      auto-commit: true
```

### application.yml for Cloud Foundry

```yaml
spring:
  application:
    name: ${SPRING_APPLICATION_NAME:my-application}
  
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:local}

---
# Cloud profile
spring:
  config:
    activate:
      on-profile: cloud
  
  # java-cfenv-boot reads VCAP_SERVICES and sets spring.datasource.url/username/password automatically.
  # Do not set them here — explicit values (even placeholders with defaults) resolve eagerly and
  # override what java-cfenv-boot injects, causing the app to connect to localhost instead of the
  # bound CF service.
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
  
  # JPA Configuration
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
  
  # JMS from CF service binding
  artemis:
    broker-url: ${vcap.services.${spring.application.name}-mq.credentials.uri}
    user: ${vcap.services.${spring.application.name}-mq.credentials.username}
    password: ${vcap.services.${spring.application.name}-mq.credentials.password}
  
  # Virtual threads
  threads:
    virtual:
      enabled: true

# Actuator endpoints
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when_authorized
      probes:
        enabled: true
  health:
    ssl:
      enabled: true
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true

# Server configuration
server:
  port: ${PORT:8080}
  shutdown: graceful

# Logging
logging:
  level:
    root: INFO
    com.example: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"

---
# Local development profile — datasource defaults for running without a bound CF service
spring:
  config:
    activate:
      on-profile: local
  datasource:
    url: jdbc:postgresql://localhost:5432/myapp
    username: postgres
    password: postgres
```

### Service Binding Examples

**PostgreSQL:**

```yaml
# In manifest.yml
services:
  - my-app-db

# Create service
# cf create-service postgresql small my-app-db
```

**RabbitMQ/ActiveMQ:**

```yaml
# In manifest.yml
services:
  - my-app-mq

# Create service
# cf create-service rabbitmq standard my-app-mq
```

**Config Server:**

```yaml
# In manifest.yml
services:
  - config-server

# Spring Cloud Config integration
spring:
  config:
    import: optional:configserver:${vcap.services.config-server.credentials.uri}
```

---
