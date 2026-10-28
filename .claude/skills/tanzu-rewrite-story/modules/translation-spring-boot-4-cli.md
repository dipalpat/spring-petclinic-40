# Spring Boot 4 Translation: ServiceCommand

Map **ServiceCommand** to Spring Shell 4 (`@ShellComponent`, `@ShellMethod`, `@ShellOption`). For the core translation guide, see [../SKILL.md](../SKILL.md).

## Translation Workflow

For each `ServiceCommand` in the spec:

1. Read `ServiceCommand` for CLI contract (command key, options, parameters)
2. Read referenced `ServiceOperation` for business logic (inputs, steps, structural_events)
3. Generate `@ShellComponent` with `@ShellMethod` from ServiceCommand
4. Generate `@Service` implementation from ServiceOperation
5. Wire shell method to call service method

---

## Basic Patterns

### Pattern 1: Simple Command

**From Spec:**
```yaml
ServiceCommand:
  name: "greetCommand"
  command_key: "greet"
  description: "Greets the user"
  options:
    - name: "userName"
      short_name: "n"
      long_name: "name"
      type: String
      required: false
      default_value: "World"
      description: "Name to greet"
  operation: "greetUser"
  output_format: TEXT

ServiceOperation:
  name: "greetUser"
  type: COMMAND
  inputs:
    - name: userName
      type: String
      required: true
  output: String
  steps:
    - "Format greeting message with userName"
```

**To Spring Shell:**
```java
@ShellComponent
public class GreetCommands {
    
    private final GreetService greetService;
    
    public GreetCommands(GreetService greetService) {
        this.greetService = greetService;
    }
    
    @ShellMethod(key = "greet", value = "Greets the user")
    public String greet(
            @ShellOption(value = {"-n", "--name"}, 
                         defaultValue = "World",
                         help = "Name to greet") 
            String userName) {
        return greetService.greetUser(userName);
    }
}

@Service
public class GreetService {
    
    @Transactional(readOnly = true)
    public String greetUser(String userName) {
        return String.format("Hello, %s!", userName);
    }
}
```

---

### Pattern 2: Command with Multiple Options

**From Spec:**
```yaml
ServiceCommand:
  name: "exportDataCommand"
  command_key: "export-data"
  description: "Export order data to file"
  command_aliases: ["export", "dump"]
  options:
    - name: "filename"
      short_name: "f"
      long_name: "file"
      type: String
      required: true
      description: "Output file path"
    - name: "format"
      short_name: "t"
      long_name: "type"
      type: ExportFormat
      required: false
      default_value: "CSV"
      description: "Export format (CSV, JSON, XML)"
  operation: "exportOrderData"
  output_format: TEXT

ServiceOperation:
  name: "exportOrderData"
  type: COMMAND
  inputs:
    - name: filename
      type: String
      required: true
    - name: format
      type: ExportFormat
      required: true
  output: ExportResult
  transactional: true
  read_only: true
  steps:
    - "Get all orders as $orders"
    - "Format $orders according to format"
    - "Write formatted data to filename"
  structural_events:
    - kind: EXTERNAL_CALL
      description: "File written to filesystem"
```

**To Spring Shell:**
```java
@ShellComponent
public class ExportCommands {
    
    private final ExportService exportService;
    
    public ExportCommands(ExportService exportService) {
        this.exportService = exportService;
    }
    
    @ShellMethod(key = {"export-data", "export", "dump"}, 
                 value = "Export order data to file")
    public String exportData(
            @ShellOption(value = {"-f", "--file"}, 
                         help = "Output file path") 
            String filename,
            
            @ShellOption(value = {"-t", "--type"}, 
                         defaultValue = "CSV",
                         help = "Export format (CSV, JSON, XML)") 
            ExportFormat format) {
        
        ExportResult result = exportService.exportOrderData(filename, format);
        return String.format("Exported %d orders to %s", 
                           result.getRecordCount(), filename);
    }
}

@Service
public class ExportService {
    
    private final OrderRepository orderRepository;
    
    @Transactional(readOnly = true)
    public ExportResult exportOrderData(String filename, ExportFormat format) {
        List<Order> orders = orderRepository.findAll();
        
        String formattedData = switch (format) {
            case CSV -> formatAsCSV(orders);
            case JSON -> formatAsJSON(orders);
            case XML -> formatAsXML(orders);
        };
        
        Files.writeString(Path.of(filename), formattedData);
        
        return new ExportResult(orders.size(), filename);
    }
}
```

---

### Pattern 3: Subcommands

**From Spec:**
```yaml
ServiceCommand:
  name: "ordersListCommand"
  command_key: "orders list"
  description: "List all orders"
  parent_command: "orders"
  operation: "listOrders"

ServiceCommand:
  name: "ordersCancelCommand"
  command_key: "orders cancel"
  description: "Cancel an order"
  parent_command: "orders"
  options:
    - name: "orderId"
      short_name: "i"
      long_name: "id"
      type: Long
      required: true
      description: "Order ID to cancel"
  operation: "cancelOrder"
```

**To Spring Shell:**
```java
@ShellComponent
public class OrderCommands {
    
    private final OrderService orderService;
    
    public OrderCommands(OrderService orderService) {
        this.orderService = orderService;
    }
    
    @ShellMethod(key = "orders list", value = "List all orders")
    public String listOrders() {
        List<Order> orders = orderService.listOrders();
        return formatOrderTable(orders);
    }
    
    @ShellMethod(key = "orders cancel", value = "Cancel an order")
    public String cancelOrder(
            @ShellOption(value = {"-i", "--id"}, 
                         help = "Order ID to cancel") 
            Long orderId) {
        orderService.cancelOrder(orderId);
        return String.format("Order %d cancelled successfully", orderId);
    }
}
```

---

## Mapping Rules

### Command Key
- Use `ServiceCommand.command_key` for `@ShellMethod(key = "...")`
- Include `command_aliases` as additional keys: `key = {"primary", "alias1", "alias2"}`

### Options
- Map each `ServiceCommand.options[]` to `@ShellOption` parameter
- Short name + long name: `value = {"-f", "--file"}`
- Required options: use parameter directly (no defaultValue)
- Optional options: provide `defaultValue` from spec
- Help text: `help = "..."`

### Positional Parameters
- Map to method parameters without `@ShellOption`
- Preserve order by `index`

### Service Call
- Shell method calls `ServiceOperation.name` with mapped parameters
- Parameters match by name: `ServiceCommand.options[].name` = `ServiceOperation.inputs[].name`

### Output Format
Use `ServiceCommand.output_format` to determine display:
- `TEXT`: Return formatted string
- `JSON`: Serialize output using Jackson
- `TABLE`: Format as Spring Shell table
- `NONE`: Return void or simple success message

### Service Implementation
Generate from `ServiceOperation`:
- Method signature from inputs/output
- `@Transactional` from transactional flag
- Implementation from steps and structural_events
- Use standard ServiceOperation translation patterns

---

## Advanced Features

### Tab Completion

When `ServiceCommand.interactive_features.completion_support = true`:

```java
@ShellComponent
public class ExportCommands {
    
    @ShellMethod(key = "export", value = "Export data")
    public String export(
            @ShellOption(valueProvider = FormatCompletionProvider.class) 
            ExportFormat format) {
        // ...
    }
}

class FormatCompletionProvider implements ValueProvider {
    @Override
    public List<CompletionProposal> complete(CompletionContext context) {
        return Stream.of(ExportFormat.values())
            .map(f -> new CompletionProposal(f.name().toLowerCase()))
            .toList();
    }
}
```

### Conditional Availability

When `ServiceCommand.interactive_features.conditional_availability` is present:

```java
@ShellComponent
public class AdminCommands {
    
    @ShellMethod(key = "admin-reset", value = "Reset system (admin only)")
    public String adminReset() {
        // ...
    }
    
    public Availability adminResetAvailability() {
        return SecurityContextHolder.getContext().getAuthentication() != null
            && hasRole("ADMIN")
            ? Availability.available()
            : Availability.unavailable("Admin role required");
    }
}
```

---

## Testing

### Service Unit Test

Test ServiceOperation business logic:

```java
@ExtendWith(MockitoExtension.class)
class ExportServiceTest {
    
    @Mock
    private OrderRepository orderRepository;
    
    @InjectMocks
    private ExportService exportService;
    
    @Test
    void shouldExportOrdersToCSV() {
        List<Order> orders = List.of(
            new Order(1L, "ORD-001", OrderStatus.COMPLETED),
            new Order(2L, "ORD-002", OrderStatus.PENDING)
        );
        when(orderRepository.findAll()).thenReturn(orders);
        
        ExportResult result = exportService.exportOrderData("output.csv", CSV);
        
        assertThat(result.getRecordCount()).isEqualTo(2);
        assertThat(Path.of("output.csv")).exists();
    }
}
```

### Shell Integration Test

Test ServiceCommand CLI interface:

```java
@SpringBootTest
class ExportCommandsIntegrationTest {
    
    @Autowired
    private Shell shell;
    
    @Test
    void shouldExecuteExportDataCommand() {
        ShellTestClient client = ShellTestClient.builder()
            .shell(shell)
            .build();
        
        CommandResult result = client.run("export-data -f output.csv -t CSV");
        
        assertThat(result.isSuccess()).isTrue();
        assertThat(result.getOutput()).contains("Exported");
        assertThat(Path.of("output.csv")).exists();
    }
    
    @Test
    void shouldFailWhenFileOptionMissing() {
        ShellTestClient client = ShellTestClient.builder()
            .shell(shell)
            .build();
        
        CommandResult result = client.run("export-data -t CSV");
        
        assertThat(result.isSuccess()).isFalse();
        assertThat(result.getOutput()).contains("required");
    }
}
```

---

## Application Types

### Standalone CLI Application

When spec has only `ServiceCommand` entities (no `ServiceEndpoint`):

**application.yml:**
```yaml
spring:
  main:
    web-application-type: none
  shell:
    interactive:
      enabled: true
    command:
      version:
        enabled: true
```

**Main Class:**
```java
@SpringBootApplication
public class CliApplication {
    public static void main(String[] args) {
        SpringApplication.run(CliApplication.class, args);
    }
}
```

### Hybrid Application

When spec has both `ServiceEndpoint` and `ServiceCommand`:

**application.yml:**
```yaml
spring:
  shell:
    interactive:
      enabled: false  # Disable interactive shell for web app
```

Commands available but shell runs in non-interactive mode. Can be invoked programmatically or via profile.

---

## Dependencies

Ensure Spring Shell starter is included (see [prerequisites](translation-spring-boot-4-prerequisites.md)):

```xml
<dependency>
    <groupId>org.springframework.shell</groupId>
    <artifactId>spring-shell-starter</artifactId>
</dependency>
```

---

## Summary

1. **ServiceCommand** defines CLI contract (command key, options, flags)
2. **ServiceOperation** defines business logic (inputs, steps, structural_events)
3. Shell layer calls service layer
4. Generate both unit and integration tests
5. Configure for standalone or hybrid mode
