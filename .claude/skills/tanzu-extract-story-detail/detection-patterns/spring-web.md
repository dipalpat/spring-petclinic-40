# Spring Web (non-Boot) Patterns

This document defines detection patterns for Spring Web applications that are not using Spring Boot. These patterns help identify legacy Spring MVC configurations, controllers, services, and other components that need modernization.

## Table of Contents

- [Core Framework Detection](#core-framework-detection)
- [Controller Detection](#controller-detection)
- [Service Layer Detection](#service-layer-detection)
- [Transaction Management](#transaction-management)
- [Data Access Patterns](#data-access-patterns)
- [JNDI Resource Lookups](#jndi-resource-lookups)
- [Security Configuration](#security-configuration)
- [Messaging Patterns](#messaging-patterns)
- [Scheduled Tasks](#scheduled-tasks)
- [View Technology Detection](#view-technology-detection)
- [Property Configuration](#property-configuration)
- [External API Clients](#external-api-clients)
- [File Transfer Patterns](#file-transfer-patterns)

---

## Core Framework Detection

### XML-Based Spring Configuration

| Pattern | `<beans\s+xmlns` |
|---------|------------------|
| **File Types** | `*.xml` |
| **Indicates** | Spring XML configuration file |
| **Schema Entity** | `ModernizationRequirements.source_technology = SPRING_WEB` |

**Example:**
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="...">
```

**Extraction Notes:** Parse XML to extract bean definitions, property values, and dependencies.

---

### Context Loader Listener

| Pattern | `ContextLoaderListener` |
|---------|-------------------------|
| **File Types** | `web.xml` |
| **Indicates** | Spring web application with root application context |
| **Schema Entity** | `ModernizationRequirements.source_technology = SPRING_WEB` |

**Example:**
```xml
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

**Extraction Notes:** Identify the `contextConfigLocation` param to find Spring config files.

---

### Dispatcher Servlet

| Pattern | `DispatcherServlet` |
|---------|---------------------|
| **File Types** | `web.xml`, `*.java` |
| **Indicates** | Spring MVC application |
| **Schema Entity** | `ModernizationRequirements.source_technology = SPRING_WEB` |

**Example:**
```xml
<servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
</servlet>
```

**Extraction Notes:** Extract servlet name to find `{servlet-name}-servlet.xml` configuration.

---

## Controller Detection

### @Controller Annotation

| Pattern | `@Controller` |
|---------|---------------|
| **File Types** | `*.java` |
| **Indicates** | Spring MVC controller class |
| **Schema Entity** | `ServiceEndpoint` |

**Example:**
```java
@Controller
@RequestMapping("/orders")
public class OrderController {
    // ...
}
```

**Extraction Notes:** 
- Extract class-level `@RequestMapping` for base path
- Scan methods for endpoint definitions

---

### @RequestMapping Annotation

| Pattern | `@RequestMapping\s*\(` |
|---------|------------------------|
| **File Types** | `*.java` |
| **Indicates** | HTTP endpoint definition |
| **Schema Entity** | `ServiceEndpoint` |

**Example:**
```java
@RequestMapping(value = "/orders/{id}", method = RequestMethod.GET)
public String viewOrder(@PathVariable Long id, Model model) {
    // ...
}
```

**Extraction Notes:**
- Extract `value` or first parameter for path
- Extract `method` for HTTP method
- Extract `produces`/`consumes` for content types
- Parse method parameters for `@PathVariable`, `@RequestParam`, `@RequestBody`

---

### @RestController Annotation

| Pattern | `@RestController` |
|---------|-------------------|
| **File Types** | `*.java` |
| **Indicates** | REST API controller (returns data, not views) |
| **Schema Entity** | `ServiceEndpoint` with `response_type = JSON` |

**Example:**
```java
@RestController
@RequestMapping("/api/orders")
public class OrderRestController {
    // ...
}
```

---

### HTTP Method Annotations

| Pattern | `@(GetMapping\|PostMapping\|PutMapping\|DeleteMapping\|PatchMapping)\s*\(` |
|---------|---------------------------------------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | HTTP endpoint with specific method |
| **Schema Entity** | `ServiceEndpoint` |

**Example:**
```java
@GetMapping("/{id}")
public Order getOrder(@PathVariable Long id) {
    return orderService.findById(id);
}
```

---

## Service Layer Detection

### @Service Annotation

| Pattern | `@Service` |
|---------|------------|
| **File Types** | `*.java` |
| **Indicates** | Business service class |
| **Schema Entity** | `ServiceOperation` |

**Example:**
```java
@Service
@Transactional
public class OrderService {
    public Order findById(Long id) {
        return orderRepository.findById(id).orElseThrow();
    }
}
```

**Extraction Notes:**
- Extract public methods as `ServiceOperation`
- Check for `@Transactional` annotations
- Identify injected dependencies

---

### @Repository Annotation

| Pattern | `@Repository` |
|---------|---------------|
| **File Types** | `*.java` |
| **Indicates** | Data access class |
| **Schema Entity** | `DataStructure` (identifies persistence needs) |

**Example:**
```java
@Repository
public class OrderDaoImpl implements OrderDao {
    @Autowired
    private SessionFactory sessionFactory;
}
```

---

### @Component Annotation

| Pattern | `@Component` |
|---------|--------------|
| **File Types** | `*.java` |
| **Indicates** | Generic Spring-managed bean |
| **Schema Entity** | `ServiceOperation` (when containing business logic) |

**Example:**
```java
@Component
public class OrderValidator {
    public void validate(Order order) {
        // Validation logic
    }
}
```

**Extraction Notes:**
- `@Component` is a generic stereotype; check class purpose to determine schema entity
- May be used for utilities, validators, converters, or other infrastructure components
- Extract public methods as potential `ServiceOperation` entries if they contain business logic

---

## Transaction Management

### @Transactional Annotation

| Pattern | `@Transactional` |
|---------|------------------|
| **File Types** | `*.java` |
| **Indicates** | Transaction-managed method or class |
| **Schema Entity** | `TransactionRequirement`, `ServiceOperation.transactional = true` |

**Example:**
```java
@Transactional(propagation = Propagation.REQUIRED, rollbackFor = Exception.class)
public void processOrder(Order order) {
    // ...
}
```

**Extraction Notes:**
- Extract `propagation` → `transaction_propagation`
- Extract `readOnly` → `read_only`
- Extract `rollbackFor`/`noRollbackFor` → `rollback_rules`

---

### XML Transaction Configuration

| Pattern | `<tx:annotation-driven` |
|---------|-------------------------|
| **File Types** | `*.xml` |
| **Indicates** | Annotation-based transaction management enabled |
| **Schema Entity** | `TransactionRequirement` |

**Example:**
```xml
<tx:annotation-driven transaction-manager="transactionManager"/>
```

---

## Data Access Patterns

### JdbcTemplate Usage

| Pattern | `JdbcTemplate\|NamedParameterJdbcTemplate` |
|---------|---------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | Direct JDBC database access |
| **Schema Entity** | `ExternalResourceRequirement.type = DATASOURCE` |

**Example:**
```java
@Autowired
private JdbcTemplate jdbcTemplate;

public List<Order> findAll() {
    return jdbcTemplate.query("SELECT * FROM orders", new OrderRowMapper());
}
```

---

### Hibernate/JPA Configuration

| Pattern | `LocalSessionFactoryBean\|LocalContainerEntityManagerFactoryBean` |
|---------|-------------------------------------------------------------------|
| **File Types** | `*.xml`, `*.java` |
| **Indicates** | Hibernate/JPA persistence configuration |
| **Schema Entity** | `TransactionRequirement.transaction_manager_type = JPA` |

**Example:**
```xml
<bean id="sessionFactory" class="org.springframework.orm.hibernate5.LocalSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <property name="packagesToScan" value="com.example.model"/>
</bean>
```

---

### @Entity Annotation

| Pattern | `@Entity` |
|---------|-----------|
| **File Types** | `*.java` |
| **Indicates** | JPA entity class |
| **Schema Entity** | `DataStructure` with `persistence_type = ENTITY` |

**Example:**
```java
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "order_number", unique = true)
    private String orderNumber;
}
```

**Extraction Notes:**
- For full extraction rules (fields, relationships, @Table, @Column, @Id, embeddables), use the [JPA persistence patterns](jpa-persistence.md). Apply whenever Spring Web or Spring Boot is in use. Both `javax.persistence` and `jakarta.persistence` are in scope.

---

## JNDI Resource Lookups

### JNDI Lookup in XML

| Pattern | `<jee:jndi-lookup` |
|---------|---------------------|
| **File Types** | `*.xml` |
| **Indicates** | JNDI resource reference |
| **Schema Entity** | `ExternalResourceRequirement` |

**Example:**
```xml
<jee:jndi-lookup id="dataSource" jndi-name="java:comp/env/jdbc/OrdersDS"/>
```

**Extraction Notes:**
- Extract `id` → resource name
- Extract `jndi-name` → `legacy_jndi_name`
- Infer `type` from naming pattern (jdbc → DATASOURCE, jms → CONNECTION_FACTORY)
- **For datasources (`type: DATASOURCE`)**: Attempt to resolve connection pool settings from server descriptors (see [app-servers.md](app-servers.md)). Populate `ExternalResourceRequirement.connection_pool` and `spring_boot_config.properties` with HikariCP property names. When available, extract: `maximum-pool-size`, `minimum-idle`, `connection-timeout`, `idle-timeout`, `max-lifetime`, `auto-commit` (map to `spring.datasource.hikari.*` in Spring Boot config).

---

### JndiTemplate/JndiObjectFactoryBean

| Pattern | `JndiTemplate\|JndiObjectFactoryBean` |
|---------|----------------------------------------|
| **File Types** | `*.java`, `*.xml` |
| **Indicates** | Programmatic JNDI lookup |
| **Schema Entity** | `ExternalResourceRequirement` |

---

## Security Configuration

### Spring Security XML

| Pattern | `<security:http` |
|---------|-------------------|
| **File Types** | `*.xml` |
| **Indicates** | Spring Security configuration |
| **Schema Entity** | `SecurityRequirement` |

**Example:**
```xml
<security:http auto-config="true">
    <security:intercept-url pattern="/admin/**" access="hasRole('ADMIN')"/>
    <security:form-login login-page="/login"/>
</security:http>
```

**Extraction Notes:**
- Extract `intercept-url` patterns → `protected_endpoints`
- Extract `form-login` → `authentication_method = FORM`
- Extract role references for `roles` array

---

### @Secured Annotation

| Pattern | `@Secured\s*\(` |
|---------|-----------------|
| **File Types** | `*.java` |
| **Indicates** | Method-level security |
| **Schema Entity** | `SecurityRequirement.protected_endpoints`, `ServiceEndpoint.authorization` |

**Example:**
```java
@Secured("ROLE_ADMIN")
public void deleteOrder(Long id) {
    // ...
}
```

---

### @PreAuthorize Annotation

| Pattern | `@PreAuthorize\s*\(` |
|---------|----------------------|
| **File Types** | `*.java` |
| **Indicates** | Expression-based security |
| **Schema Entity** | `SecurityRequirement` |

**Example:**
```java
@PreAuthorize("hasRole('ADMIN') or #order.createdBy == authentication.name")
public void updateOrder(Order order) {
    // ...
}
```

---

## Messaging Patterns

### JmsTemplate Usage

| Pattern | `JmsTemplate` |
|---------|---------------|
| **File Types** | `*.java` |
| **Indicates** | JMS message producer |
| **Schema Entity** | `MessagingRequirement` with `direction = PRODUCER` |

**Example:**
```java
@Autowired
private JmsTemplate jmsTemplate;

public void sendNotification(OrderEvent event) {
    jmsTemplate.convertAndSend("order-events", event);
}
```

**Extraction Notes:**
- Extract destination name from `convertAndSend`/`send` calls
- Determine `pattern` based on destination type

---

### Message Listener Container

| Pattern | `DefaultMessageListenerContainer\|@JmsListener` |
|---------|--------------------------------------------------|
| **File Types** | `*.java`, `*.xml` |
| **Indicates** | JMS message consumer |
| **Schema Entity** | `MessagingRequirement` with `direction = CONSUMER` |

**Example:**
```xml
<bean class="org.springframework.jms.listener.DefaultMessageListenerContainer">
    <property name="connectionFactory" ref="connectionFactory"/>
    <property name="destination" ref="orderQueue"/>
    <property name="messageListener" ref="orderMessageListener"/>
</bean>
```

---

## Scheduled Tasks

### @Scheduled Annotation

| Pattern | `@Scheduled\s*\(` |
|---------|-------------------|
| **File Types** | `*.java` |
| **Indicates** | Scheduled task |
| **Schema Entity** | `ServiceTask` |

**Example:**
```java
@Scheduled(cron = "0 0 2 * * ?")
public void generateDailyReport() {
    // ...
}
```

**Extraction Notes:**
- Extract `cron` → `cron_expression`, `schedule_type = CRON`
- Extract `fixedRate` → `interval_ms`, `schedule_type = FIXED_RATE`
- Extract `fixedDelay` → `interval_ms`, `schedule_type = FIXED_DELAY`

---

### XML Task Scheduler

| Pattern | `<task:scheduled\|<task:scheduler` |
|---------|-------------------------------------|
| **File Types** | `*.xml` |
| **Indicates** | XML-based scheduled tasks |
| **Schema Entity** | `ServiceTask` |

---

## View Technology Detection

### JSP View Resolver

| Pattern | `InternalResourceViewResolver\|\.jsp` |
|---------|----------------------------------------|
| **File Types** | `*.xml`, `*.java`, `*.jsp` |
| **Indicates** | JSP view technology |
| **Schema Entity** | `ServiceEndpoint.response_type = VIEW` |

**Extraction Notes:** Flag for migration to Thymeleaf or REST API.

---

### Thymeleaf Configuration

| Pattern | `ThymeleafViewResolver\|\.html` in templates |
|---------|-----------------------------------------------|
| **File Types** | `*.xml`, `*.java` |
| **Indicates** | Thymeleaf view technology |
| **Schema Entity** | `ServiceEndpoint.response_type = VIEW` |

---

## Property Configuration

### Property Placeholder

| Pattern | `PropertyPlaceholderConfigurer\|<context:property-placeholder` |
|---------|----------------------------------------------------------------|
| **File Types** | `*.xml`, `*.java` |
| **Indicates** | Externalized configuration |
| **Schema Entity** | `ExternalResourceRequirement` (for properties that need migration) |

**Example:**
```xml
<context:property-placeholder location="classpath:application.properties"/>
```

**Extraction Notes:** Parse referenced properties files to extract configuration that needs migration to `application.yml`.

---

## External API Clients

### RestTemplate Usage

| Pattern | `RestTemplate` |
|---------|----------------|
| **File Types** | `*.java` |
| **Indicates** | Synchronous REST API client |
| **Schema Entity** | `IntegrationPoint` with `type = REST_API` |

**Example:**
```java
@Autowired
private RestTemplate restTemplate;

public CustomerDTO getCustomer(Long customerId) {
    return restTemplate.getForObject(
        "http://customer-service/api/customers/{id}",
        CustomerDTO.class,
        customerId
    );
}
```

**Extraction Notes:**
- Extract base URL for `IntegrationPoint.endpoint`
- Identify HTTP method from `getForObject`, `postForEntity`, `exchange`, etc.
- Migration: Consider replacing with `WebClient` for reactive support

---

### WebClient Usage

| Pattern | `WebClient` |
|---------|-------------|
| **File Types** | `*.java` |
| **Indicates** | Reactive/async REST API client |
| **Schema Entity** | `IntegrationPoint` with `type = REST_API` |

**Example:**
```java
@Autowired
private WebClient webClient;

public Mono<CustomerDTO> getCustomer(Long customerId) {
    return webClient.get()
        .uri("/api/customers/{id}", customerId)
        .retrieve()
        .bodyToMono(CustomerDTO.class);
}
```

**Extraction Notes:**
- Modern Spring approach for HTTP clients
- Extract base URL from `WebClient.Builder` configuration
- Note reactive return types (`Mono`, `Flux`) for migration planning

---

### @FeignClient Annotation

| Pattern | `@FeignClient\s*\(` |
|---------|---------------------|
| **File Types** | `*.java` |
| **Indicates** | Declarative REST client (Spring Cloud) |
| **Schema Entity** | `IntegrationPoint` with `type = REST_API` |

**Example:**
```java
@FeignClient(name = "customer-service", url = "${customer.service.url}")
public interface CustomerClient {
    
    @GetMapping("/api/customers/{id}")
    CustomerDTO getCustomer(@PathVariable Long id);
    
    @PostMapping("/api/customers")
    CustomerDTO createCustomer(@RequestBody CustomerDTO customer);
}
```

**Extraction Notes:**
- Extract `name` or `value` → service name
- Extract `url` → `IntegrationPoint.endpoint`
- Extract method annotations for available operations
- Note: Requires Spring Cloud OpenFeign dependency

---

### SOAP Client (JAX-WS)

| Pattern | `@WebServiceClient\|JAXBContext\|Service\.create` |
|---------|---------------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | SOAP web service client |
| **Schema Entity** | `IntegrationPoint` with `type = SOAP_API` |

**Example:**
```java
@WebServiceClient(name = "CustomerService", 
                  targetNamespace = "http://example.com/customer",
                  wsdlLocation = "http://example.com/customer?wsdl")
public class CustomerService extends Service {
    // Generated client code
}
```

**Extraction Notes:**
- Extract `wsdlLocation` → `IntegrationPoint.endpoint`
- Extract `targetNamespace` for service identification
- Generated clients typically in `**/generated/**` or `**/ws/**` packages
- Migration: Consider converting to REST if possible

---

## File Transfer Patterns

### FTP/SFTP Client

| Pattern | `FTPClient\|SFTPClient\|JSch\|ChannelSftp\|FtpRemoteFileTemplate` |
|---------|-------------------------------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | FTP/SFTP file transfer |
| **Schema Entity** | `IntegrationPoint` with `type = FTP` |

**Example (Apache Commons Net):**
```java
FTPClient ftpClient = new FTPClient();
ftpClient.connect("ftp.example.com", 21);
ftpClient.login("user", "password");
ftpClient.retrieveFile("/remote/path/file.csv", outputStream);
```

**Example (JSch for SFTP):**
```java
JSch jsch = new JSch();
Session session = jsch.getSession("user", "sftp.example.com", 22);
Channel channel = session.openChannel("sftp");
ChannelSftp sftpChannel = (ChannelSftp) channel;
sftpChannel.get("/remote/path/file.csv", localOutputStream);
```

**Example (Spring Integration):**
```java
@Autowired
private FtpRemoteFileTemplate ftpTemplate;

public void downloadFile(String remotePath, String localPath) {
    ftpTemplate.get(remotePath, inputStream -> {
        Files.copy(inputStream, Paths.get(localPath));
        return true;
    });
}
```

**Extraction Notes:**
- Extract host, port, credentials source
- Identify direction: upload, download, or bidirectional
- Note file patterns and remote paths
- Migration: Consider Spring Integration FTP/SFTP adapters

---

### File I/O Operations

| Pattern | `FileInputStream\|FileOutputStream\|Files\.read\|Files\.write\|BufferedReader\|BufferedWriter` |
|---------|------------------------------------------------------------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | Local file system access |
| **Schema Entity** | `IntegrationPoint` with `type = FILE_SYSTEM` |

**Example:**
```java
// Reading file
try (BufferedReader reader = Files.newBufferedReader(Paths.get("/data/input.csv"))) {
    String line;
    while ((line = reader.readLine()) != null) {
        processLine(line);
    }
}

// Writing file
try (BufferedWriter writer = Files.newBufferedWriter(Paths.get("/data/output.csv"))) {
    writer.write(content);
}
```

**Extraction Notes:**
- Extract file paths (may be hardcoded or from configuration)
- Identify read vs. write operations
- Note file formats (CSV, XML, JSON, etc.)
- Migration: Ensure paths are configurable via `application.yml`

---

### Spring Resource Loading

| Pattern | `ResourceLoader\|@Value.*classpath:\|Resource\s+resource` |
|---------|-----------------------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | Spring resource abstraction for file access |
| **Schema Entity** | `IntegrationPoint` with `type = FILE_SYSTEM` |

**Example:**
```java
@Value("classpath:templates/email-template.html")
private Resource emailTemplate;

@Autowired
private ResourceLoader resourceLoader;

public String loadTemplate() {
    Resource resource = resourceLoader.getResource("classpath:config/rules.xml");
    return StreamUtils.copyToString(resource.getInputStream(), StandardCharsets.UTF_8);
}
```

**Extraction Notes:**
- Extract resource locations (classpath, file, URL)
- Note: `classpath:` resources are bundled with application
- Note: `file:` resources are external file system paths
