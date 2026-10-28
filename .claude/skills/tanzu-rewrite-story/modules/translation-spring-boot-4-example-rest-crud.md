# Full Example: REST CRUD

This document contains the complete **Example 1: Basic REST CRUD** from the Spring Boot 4 translation instructions. Load it when you need a full end-to-end reference.

---

### Example 1: Basic REST CRUD

#### User Story

```
As a customer service representative,
I want to manage customer records through a REST API,
So that I can create, read, update, and delete customer information.
```

#### Technical Context (Extracted)

```yaml
ServiceEndpoint:
  - path: "/api/customers"
    method: GET
    response_type: JSON
    response_body:
      name: CustomerList
      type: COLLECTION
      element_type: Customer
    authentication: AUTHENTICATED
    produces: ["application/json"]

  - path: "/api/customers/{id}"
    method: GET
    path_variables:
      - name: id
        type: Long
        description: "Customer unique identifier"
    response_type: JSON
    response_body:
      name: Customer
      type: DOMAIN_OBJECT
    authentication: AUTHENTICATED
    error_responses:
      - status_code: 404
        condition: "Customer not found"

  - path: "/api/customers"
    method: POST
    request_body:
      name: CreateCustomerRequest
      type: DTO
    response_type: JSON
    response_body:
      name: Customer
      type: DOMAIN_OBJECT
    authentication: AUTHENTICATED
    authorization: ["ROLE_ADMIN", "ROLE_CUSTOMER_SERVICE"]
    consumes: ["application/json"]

  - path: "/api/customers/{id}"
    method: PUT
    path_variables:
      - name: id
        type: Long
    request_body:
      name: UpdateCustomerRequest
      type: DTO
    response_type: JSON
    authentication: AUTHENTICATED
    authorization: ["ROLE_ADMIN", "ROLE_CUSTOMER_SERVICE"]

  - path: "/api/customers/{id}"
    method: DELETE
    path_variables:
      - name: id
        type: Long
    response_type: VOID
    authentication: AUTHENTICATED
    authorization: ["ROLE_ADMIN"]

DataStructure:
  - name: Customer
    type: DOMAIN_OBJECT
    fields:
      - name: id
        type: Long
        constraints: [NOT_NULL]
      - name: firstName
        type: String
        constraints: [NOT_NULL, MAX_LENGTH_50]
      - name: lastName
        type: String
        constraints: [NOT_NULL, MAX_LENGTH_50]
      - name: email
        type: String
        constraints: [NOT_NULL, EMAIL, UNIQUE]
      - name: createdAt
        type: LocalDateTime
      - name: updatedAt
        type: LocalDateTime
```

#### Generated Spring Boot 4 Code

**Entity Class:**

```java
package com.example.customer.domain;

import jakarta.persistence.*;
import jakarta.validation.constraints.*;
import org.jspecify.annotations.NonNull;
import org.jspecify.annotations.Nullable;
import java.time.LocalDateTime;

@Entity
@Table(name = "customers")
public class Customer {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotNull
    @Size(max = 50)
    @Column(name = "first_name", nullable = false, length = 50)
    private String firstName;

    @NotNull
    @Size(max = 50)
    @Column(name = "last_name", nullable = false, length = 50)
    private String lastName;

    @NotNull
    @Email
    @Column(nullable = false, unique = true)
    private String email;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    // JSpecify-annotated getters
    public @Nullable Long getId() {
        return id;
    }

    public @NonNull String getFirstName() {
        return firstName;
    }

    public @NonNull String getLastName() {
        return lastName;
    }

    public @NonNull String getEmail() {
        return email;
    }

    public @Nullable LocalDateTime getCreatedAt() {
        return createdAt;
    }

    public @Nullable LocalDateTime getUpdatedAt() {
        return updatedAt;
    }

    // Setters
    public void setId(Long id) {
        this.id = id;
    }

    public void setFirstName(@NonNull String firstName) {
        this.firstName = firstName;
    }

    public void setLastName(@NonNull String lastName) {
        this.lastName = lastName;
    }

    public void setEmail(@NonNull String email) {
        this.email = email;
    }

    public void setCreatedAt(LocalDateTime createdAt) {
        this.createdAt = createdAt;
    }

    public void setUpdatedAt(LocalDateTime updatedAt) {
        this.updatedAt = updatedAt;
    }

    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }
}
```

**DTO Records:**

```java
package com.example.customer.dto;

import jakarta.validation.constraints.*;
import org.jspecify.annotations.NonNull;

public record CreateCustomerRequest(
    @NotNull @Size(max = 50) @NonNull String firstName,
    @NotNull @Size(max = 50) @NonNull String lastName,
    @NotNull @Email @NonNull String email
) {}

public record UpdateCustomerRequest(
    @Size(max = 50) String firstName,
    @Size(max = 50) String lastName,
    @Email String email
) {}

public record CustomerResponse(
    @NonNull Long id,
    @NonNull String firstName,
    @NonNull String lastName,
    @NonNull String email,
    LocalDateTime createdAt,
    LocalDateTime updatedAt
) {
    public static CustomerResponse from(@NonNull Customer customer) {
        return new CustomerResponse(
            customer.getId(),
            customer.getFirstName(),
            customer.getLastName(),
            customer.getEmail(),
            customer.getCreatedAt(),
            customer.getUpdatedAt()
        );
    }
}
```

**Repository:**

```java
package com.example.customer.repository;

import com.example.customer.domain.Customer;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import org.jspecify.annotations.Nullable;
import java.util.Optional;

@Repository
public interface CustomerRepository extends JpaRepository<Customer, Long> {
    
    Optional<Customer> findByEmail(String email);
    
    boolean existsByEmail(String email);
}
```

**Service:**

```java
package com.example.customer.service;

import com.example.customer.domain.Customer;
import com.example.customer.dto.*;
import com.example.customer.repository.CustomerRepository;
import com.example.customer.exception.CustomerNotFoundException;
import com.example.customer.exception.DuplicateEmailException;
import org.jspecify.annotations.NonNull;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;

@Service
@Transactional(readOnly = true)
public class CustomerService {

    private final CustomerRepository customerRepository;

    public CustomerService(CustomerRepository customerRepository) {
        this.customerRepository = customerRepository;
    }

    public @NonNull List<Customer> findAll() {
        return customerRepository.findAll();
    }

    public @NonNull Customer findById(@NonNull Long id) {
        return customerRepository.findById(id)
            .orElseThrow(() -> new CustomerNotFoundException(id));
    }

    @Transactional
    public @NonNull Customer create(@NonNull CreateCustomerRequest request) {
        if (customerRepository.existsByEmail(request.email())) {
            throw new DuplicateEmailException(request.email());
        }
        
        Customer customer = new Customer();
        customer.setFirstName(request.firstName());
        customer.setLastName(request.lastName());
        customer.setEmail(request.email());
        
        return customerRepository.save(customer);
    }

    @Transactional
    public @NonNull Customer update(@NonNull Long id, @NonNull UpdateCustomerRequest request) {
        Customer customer = findById(id);
        
        if (request.firstName() != null) {
            customer.setFirstName(request.firstName());
        }
        if (request.lastName() != null) {
            customer.setLastName(request.lastName());
        }
        if (request.email() != null && !request.email().equals(customer.getEmail())) {
            if (customerRepository.existsByEmail(request.email())) {
                throw new DuplicateEmailException(request.email());
            }
            customer.setEmail(request.email());
        }
        
        return customerRepository.save(customer);
    }

    @Transactional
    public void delete(@NonNull Long id) {
        if (!customerRepository.existsById(id)) {
            throw new CustomerNotFoundException(id);
        }
        customerRepository.deleteById(id);
    }
}
```

**Controller:**

```java
package com.example.customer.controller;

import com.example.customer.dto.*;
import com.example.customer.service.CustomerService;
import jakarta.validation.Valid;
import org.jspecify.annotations.NonNull;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/api/customers")
public class CustomerController {

    private final CustomerService customerService;

    public CustomerController(CustomerService customerService) {
        this.customerService = customerService;
    }

    @GetMapping
    public @NonNull List<CustomerResponse> getAllCustomers() {
        return customerService.findAll().stream()
            .map(CustomerResponse::from)
            .toList();
    }

    @GetMapping("/{id}")
    public @NonNull CustomerResponse getCustomer(@PathVariable @NonNull Long id) {
        return CustomerResponse.from(customerService.findById(id));
    }

    @PostMapping
    @PreAuthorize("hasAnyRole('ADMIN', 'CUSTOMER_SERVICE')")
    @ResponseStatus(HttpStatus.CREATED)
    public @NonNull CustomerResponse createCustomer(
            @Valid @RequestBody @NonNull CreateCustomerRequest request) {
        return CustomerResponse.from(customerService.create(request));
    }

    @PutMapping("/{id}")
    @PreAuthorize("hasAnyRole('ADMIN', 'CUSTOMER_SERVICE')")
    public @NonNull CustomerResponse updateCustomer(
            @PathVariable @NonNull Long id,
            @Valid @RequestBody @NonNull UpdateCustomerRequest request) {
        return CustomerResponse.from(customerService.update(id, request));
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteCustomer(@PathVariable @NonNull Long id) {
        customerService.delete(id);
    }
}
```

**Exception Classes:**

```java
package com.example.customer.exception;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

@ResponseStatus(HttpStatus.NOT_FOUND)
public class CustomerNotFoundException extends RuntimeException {
    public CustomerNotFoundException(Long id) {
        super("Customer not found with id: " + id);
    }
}

@ResponseStatus(HttpStatus.CONFLICT)
public class DuplicateEmailException extends RuntimeException {
    public DuplicateEmailException(String email) {
        super("Email already exists: " + email);
    }
}
```

**Global Exception Handler:**

```java
package com.example.customer.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import java.time.Instant;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(CustomerNotFoundException.class)
    public ProblemDetail handleNotFound(CustomerNotFoundException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Customer Not Found");
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }

    @ExceptionHandler(DuplicateEmailException.class)
    public ProblemDetail handleDuplicateEmail(DuplicateEmailException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.CONFLICT, ex.getMessage());
        problem.setTitle("Duplicate Email");
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.BAD_REQUEST, "Validation failed");
        problem.setTitle("Validation Error");
        problem.setProperty("timestamp", Instant.now());
        problem.setProperty("errors", ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .toList());
        return problem;
    }
}
```

#### Tests

**Unit Test (Service):**

```java
package com.example.customer.service;

import com.example.customer.domain.Customer;
import com.example.customer.dto.CreateCustomerRequest;
import com.example.customer.exception.CustomerNotFoundException;
import com.example.customer.exception.DuplicateEmailException;
import com.example.customer.repository.CustomerRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.BDDMockito.*;

@ExtendWith(MockitoExtension.class)
class CustomerServiceTest {

    @Mock
    private CustomerRepository customerRepository;

    @InjectMocks
    private CustomerService customerService;

    private Customer testCustomer;

    @BeforeEach
    void setUp() {
        testCustomer = new Customer();
        testCustomer.setId(1L);
        testCustomer.setFirstName("John");
        testCustomer.setLastName("Doe");
        testCustomer.setEmail("john.doe@example.com");
    }

    @Nested
    @DisplayName("findById")
    class FindById {

        @Test
        @DisplayName("should return customer when found")
        void shouldReturnCustomerWhenFound() {
            given(customerRepository.findById(1L)).willReturn(Optional.of(testCustomer));

            Customer result = customerService.findById(1L);

            assertThat(result).isEqualTo(testCustomer);
        }

        @Test
        @DisplayName("should throw exception when not found")
        void shouldThrowExceptionWhenNotFound() {
            given(customerRepository.findById(1L)).willReturn(Optional.empty());

            assertThatThrownBy(() -> customerService.findById(1L))
                .isInstanceOf(CustomerNotFoundException.class)
                .hasMessageContaining("1");
        }
    }

    @Nested
    @DisplayName("create")
    class Create {

        @Test
        @DisplayName("should create customer successfully")
        void shouldCreateCustomerSuccessfully() {
            CreateCustomerRequest request = new CreateCustomerRequest(
                "John", "Doe", "john.doe@example.com");
            
            given(customerRepository.existsByEmail("john.doe@example.com")).willReturn(false);
            given(customerRepository.save(any(Customer.class))).willReturn(testCustomer);

            Customer result = customerService.create(request);

            assertThat(result.getEmail()).isEqualTo("john.doe@example.com");
            then(customerRepository).should().save(any(Customer.class));
        }

        @Test
        @DisplayName("should throw exception for duplicate email")
        void shouldThrowExceptionForDuplicateEmail() {
            CreateCustomerRequest request = new CreateCustomerRequest(
                "John", "Doe", "john.doe@example.com");
            
            given(customerRepository.existsByEmail("john.doe@example.com")).willReturn(true);

            assertThatThrownBy(() -> customerService.create(request))
                .isInstanceOf(DuplicateEmailException.class);
        }
    }
}
```

**Integration Test (Controller):**

```java
package com.example.customer.controller;

import com.example.customer.domain.Customer;
import com.example.customer.dto.CreateCustomerRequest;
import com.example.customer.repository.CustomerRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.web.test.RestTestClient;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import tools.jackson.databind.JsonMapper;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
@Testcontainers
class CustomerControllerIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private RestTestClient restTestClient;

    @Autowired
    private CustomerRepository customerRepository;

    @Autowired
    private JsonMapper jsonMapper;

    @BeforeEach
    void setUp() {
        customerRepository.deleteAll();
    }

    @Test
    @DisplayName("GET /api/customers - should return empty list")
    @WithMockUser
    void shouldReturnEmptyList() throws Exception {
        mockMvc.perform(get("/api/customers"))
            .andExpect(status().isOk())
            .andExpect(content().contentType(MediaType.APPLICATION_JSON))
            .andExpect(jsonPath("$").isArray())
            .andExpect(jsonPath("$").isEmpty());
    }

    @Test
    @DisplayName("POST /api/customers - should create customer")
    @WithMockUser(roles = "ADMIN")
    void shouldCreateCustomer() throws Exception {
        CreateCustomerRequest request = new CreateCustomerRequest(
            "John", "Doe", "john.doe@example.com");

        mockMvc.perform(post("/api/customers")
                .contentType(MediaType.APPLICATION_JSON)
                .content(jsonMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.firstName").value("John"))
            .andExpect(jsonPath("$.lastName").value("Doe"))
            .andExpect(jsonPath("$.email").value("john.doe@example.com"));
    }

    @Test
    @DisplayName("POST /api/customers - should reject unauthorized")
    @WithMockUser(roles = "USER")
    void shouldRejectUnauthorized() throws Exception {
        CreateCustomerRequest request = new CreateCustomerRequest(
            "John", "Doe", "john.doe@example.com");

        mockMvc.perform(post("/api/customers")
                .contentType(MediaType.APPLICATION_JSON)
                .content(jsonMapper.writeValueAsString(request)))
            .andExpect(status().isForbidden());
    }

    @Test
    @DisplayName("GET /api/customers/{id} - should return 404 for unknown id")
    @WithMockUser
    void shouldReturn404ForUnknownId() throws Exception {
        mockMvc.perform(get("/api/customers/999"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.title").value("Customer Not Found"));
    }
}
```

#### Cloud Foundry Configuration

**application.yml:**

```yaml
spring:
  application:
    name: customer-service
  
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false
    properties:
      hibernate:
        format_sql: true
  
  threads:
    virtual:
      enabled: true

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when_authorized
  health:
    ssl:
      enabled: true

server:
  port: ${PORT:8080}

---
# Local development profile — datasource defaults for running without a bound CF service.
# On CF, java-cfenv-boot reads VCAP_SERVICES and sets spring.datasource.* automatically.
# Do not add datasource properties to the base document — explicit values (even placeholders
# with defaults) resolve eagerly and override what java-cfenv-boot injects.
spring:
  config:
    activate:
      on-profile: local
  datasource:
    url: jdbc:postgresql://localhost:5432/customers
    username: postgres
    password: postgres
```

**manifest.yml:**

```yaml
applications:
  - name: customer-service
    memory: 1G
    instances: 2
    path: target/customer-service-1.0.0.jar
    buildpacks:
      - java_buildpack_offline
    env:
      JBP_CONFIG_OPEN_JDK_JRE: '{ jre: { version: 25.+ } }'
      JAVA_OPTS: '-XX:+UseZGC -XX:+ZGenerational'
    services:
      - customer-db
    routes:
      - route: customer-service.apps.internal
    health-check-type: http
    health-check-http-endpoint: /actuator/health
```

