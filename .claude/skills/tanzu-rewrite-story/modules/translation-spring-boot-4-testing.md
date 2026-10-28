# Spring Boot 4 Translation: Testing

Load this module when generating JUnit 6 tests. For the core translation guide, see [../SKILL.md](../SKILL.md).

---

## Testing Patterns

### Generating tests from user stories

**Requirement:** For each user story in the extracted specification, generate unit and/or integration tests that verify the acceptance criteria and test cases (TC-*) documented in the spec. This is **not optional**—every user story must have corresponding generated tests so the generated suite validates the implemented behavior. **Coverage of only happy path is insufficient;** the generated suite must include tests for **all categories** present in the spec's test case table (Happy Path, Validation Errors, Authorization, Not Found, Business Rule Violations, Edge Cases, UI)—e.g. validation errors, 403/404, business rule violations, edge cases.

**Process:**

1. **Read each user story** in the spec: acceptance criteria and the test case table (TC-001, TC-002, …), including all corner-case and error scenarios.
2. **Generate JUnit 6 test classes and methods** that verify **every** spec test case. Map each spec test case to at least one test method (e.g. TC-001 "Admin cancels pending order" → `shouldCancelOrderAsAdmin`, TC-002 "Non-admin" → `shouldRejectNonAdminCancelOrder`). TC-* rows extracted from existing tests or covering validation, auth, not found, business rules, or edge cases must all have corresponding test methods.
3. **Use existing patterns:**
   - **Unit tests** (Mockito, `@ExtendWith(MockitoExtension.class)`) for service/operation scenarios.
   - **Integration tests** (RestTestClient or MockMvc, `@SpringBootTest`) for endpoint scenarios.
   - **Testcontainers** for DB when the story involves persistence.
4. **Test class naming:** Align to the user story ID or title (e.g. `{StorySlug}ControllerIntegrationTest`, `{Operation}ServiceTest`).

### ⚠️ Spring Boot 4 test changes vs. Spring Boot 3

#### @WebMvcTest is removed

`@WebMvcTest` no longer exists in Spring Boot 4 (`spring-boot-test-autoconfigure` only retains JSON and JDBC slices). Use `@SpringBootTest` + `MockMvc` instead:

```java
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.test.context.bean.override.mockito.MockitoBean;   // Spring Framework 7
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.context.WebApplicationContext;

@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class MyControllerTest {

    @Autowired
    private WebApplicationContext webApplicationContext;

    private MockMvc mockMvc;

    @MockitoBean
    private MyService myService;

    @BeforeEach
    void setup() {
        mockMvc = MockMvcBuilders.webAppContextSetup(webApplicationContext).build();
    }

    @Test
    void shouldReturnOk() throws Exception {
        given(myService.findById(1)).willReturn(someEntity);

        mockMvc.perform(get("/resource/1"))
            .andExpect(status().isOk());
    }
}
```

#### @MockitoBean package changed

| Spring Boot version | Import |
|---|---|
| Boot 3 | `org.springframework.boot.test.mock.mockito.MockBean` |
| Boot 4 | `org.springframework.test.context.bean.override.mockito.MockitoBean` |

Use `@MockitoBean` (not `@MockBean`) in Spring Boot 4.

#### Mocking void service methods

When the mocked method returns `void`, use `willDoNothing()` — not `given(...).willAnswer(...)` (which fails to compile on void return types):

```java
// Correct for void methods
willDoNothing().given(myService).save(any(MyEntity.class));

// Also correct (BDD style verify after act)
then(myService).should().save(any(MyEntity.class));

// WRONG — does not compile when save() returns void
given(myService.save(any())).willAnswer(inv -> null);
```

The spec produced by [the tanzu-extract-story-detail skill](../../tanzu-extract-story-detail/SKILL.md) includes per-story test case tables; translation must consume those and emit tests that cover them.

### JUnit 6 Test Structure

```java
package com.example.service;

import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.BDDMockito.*;

@ExtendWith(MockitoExtension.class)
@DisplayName("OrderService Tests")
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @Mock
    private InventoryService inventoryService;

    @InjectMocks
    private OrderService orderService;

    @Nested
    @DisplayName("when creating orders")
    class CreateOrder {

        @Test
        @DisplayName("should create order successfully with valid items")
        void shouldCreateOrderSuccessfully() {
            // Given
            CreateOrderRequest request = new CreateOrderRequest(/* ... */);
            given(inventoryService.checkAvailability(any())).willReturn(true);
            given(orderRepository.save(any())).willAnswer(inv -> inv.getArgument(0));

            // When
            Order result = orderService.createOrder(request);

            // Then
            assertThat(result).isNotNull();
            assertThat(result.getStatus()).isEqualTo(OrderStatus.PENDING);
            then(orderRepository).should().save(any(Order.class));
        }

        @ParameterizedTest
        @CsvSource({
            "0, false",
            "1, true",
            "100, true",
            "-1, false"
        })
        @DisplayName("should validate quantity correctly")
        void shouldValidateQuantity(int quantity, boolean expectedValid) {
            // Test implementation
        }
    }

    @Nested
    @DisplayName("when cancelling orders")
    class CancelOrder {

        @Test
        @DisplayName("should throw exception for non-existent order")
        void shouldThrowForNonExistentOrder() {
            // Given
            given(orderRepository.findById(999L)).willReturn(Optional.empty());

            // When/Then
            assertThatThrownBy(() -> orderService.cancelOrder(999L))
                .isInstanceOf(OrderNotFoundException.class)
                .hasMessageContaining("999");
        }
    }
}
```

### Integration Test with RestTestClient

```java
package com.example.controller;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.web.test.RestTestClient;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class OrderControllerIntegrationTest {

    @Autowired
    private RestTestClient restTestClient;

    @Test
    @WithMockUser(roles = "USER")
    void shouldReturnOrdersForAuthenticatedUser() {
        restTestClient.get()
            .uri("/api/orders")
            .exchange()
            .expectStatus().isOk()
            .expectBody()
            .jsonPath("$.content").isArray();
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void shouldCreateOrderAsAdmin() {
        CreateOrderRequest request = new CreateOrderRequest(/* ... */);

        restTestClient.post()
            .uri("/api/orders")
            .bodyValue(request)
            .exchange()
            .expectStatus().isCreated()
            .expectBody(OrderResponse.class)
            .value(response -> {
                assertThat(response.id()).isNotNull();
                assertThat(response.status()).isEqualTo("PENDING");
            });
    }

    @Test
    void shouldRejectUnauthenticatedRequest() {
        restTestClient.get()
            .uri("/api/orders")
            .exchange()
            .expectStatus().isUnauthorized();
    }
}
```

### Testcontainers 2.0 Integration

```java
package com.example.repository;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
@Testcontainers
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class OrderRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void shouldSaveAndRetrieveOrder() {
        // Given
        Order order = new Order();
        order.setOrderNumber("ORD-001");
        order.setStatus(OrderStatus.PENDING);

        // When
        Order saved = orderRepository.save(order);
        Order found = orderRepository.findById(saved.getId()).orElseThrow();

        // Then
        assertThat(found.getOrderNumber()).isEqualTo("ORD-001");
        assertThat(found.getStatus()).isEqualTo(OrderStatus.PENDING);
    }

    @Test
    void shouldFindOrdersByStatus() {
        // Given
        createOrderWithStatus(OrderStatus.PENDING);
        createOrderWithStatus(OrderStatus.PENDING);
        createOrderWithStatus(OrderStatus.COMPLETED);

        // When
        List<Order> pendingOrders = orderRepository.findByStatus(OrderStatus.PENDING);

        // Then
        assertThat(pendingOrders).hasSize(2);
    }

    private Order createOrderWithStatus(OrderStatus status) {
        Order order = new Order();
        order.setOrderNumber("ORD-" + System.nanoTime());
        order.setStatus(status);
        return orderRepository.save(order);
    }
}
```

---
