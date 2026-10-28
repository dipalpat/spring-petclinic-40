# Picocli CLI Framework Patterns

This document defines detection patterns for applications using the Picocli CLI framework. These patterns enable extraction of `ServiceCommand` and `ServiceOperation` entities from picocli-based applications for translation to Spring Shell 4.

## Table of Contents

- [Core Picocli Detection](#core-picocli-detection)
- [Spring Boot Integration](#spring-boot-integration)
- [Command Definition](#command-definition)
- [Subcommand Patterns](#subcommand-patterns)
- [Advanced Features](#advanced-features)

---

## Core Picocli Detection

### @Command Annotation

| Pattern | `@Command\s*\(` |
|---------|-----------------|
| **File Types** | `*.java` |
| **Indicates** | Picocli command definition |
| **Schema Entity** | `ServiceCommand` + `ServiceOperation` |

**Example:**
```java
@Command(name = "export-data", 
         description = "Export data to file",
         mixinStandardHelpOptions = true)
public class ExportCommand implements Runnable {
    @Option(names = {"-f", "--file"}, required = true)
    private String filename;
    
    @Option(names = {"-t", "--type"}, defaultValue = "CSV")
    private ExportType type;
    
    @Override
    public void run() {
        // Implementation
    }
}
```

**Extraction Notes:**
- Extract command name → `ServiceCommand.command_key`
- Extract description → `ServiceCommand.description`
- Extract all `@Option` and `@Parameters` → `ServiceCommand.options` and `positional_parameters`
- Analyze `run()` or `call()` method → create `ServiceOperation` with business logic
- Link: `ServiceCommand.operation` = `ServiceOperation.name`

---

### CommandLine Class Usage

| Pattern | `CommandLine\s+(new\s+)?CommandLine\(` |
|---------|----------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | Picocli command execution |
| **Schema Entity** | Confirms Picocli framework usage |

**Example:**
```java
public static void main(String[] args) {
    int exitCode = new CommandLine(new MyCommand()).execute(args);
    System.exit(exitCode);
}
```

**Extraction Notes:**
- Indicates standalone CLI application
- Extract main class → `ServiceCommand.legacy_config.main_class`

---

## Spring Boot Integration

### CommandLineRunner with Picocli

| Pattern | `implements\s+CommandLineRunner.*@Command` |
|---------|---------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | Spring Boot app with picocli CLI |
| **Schema Entity** | `ServiceCommand` with Spring Boot integration |

**Example:**
```java
@Component
@Command(name = "process", description = "Process data")
public class ProcessCommand implements CommandLineRunner {
    @Option(names = {"-i", "--input"})
    private String input;
    
    @Autowired
    private DataService dataService;
    
    @Override
    public void run(String... args) throws Exception {
        new CommandLine(this).execute(args);
    }
}
```

**Extraction Notes:**
- Set `ServiceCommand.legacy_config.spring_boot_integration = true`
- Set `runner_type = COMMAND_LINE_RUNNER`
- Extract dependencies for ServiceOperation

---

### ApplicationRunner with Picocli

| Pattern | `implements\s+ApplicationRunner.*@Command` |
|---------|---------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | Spring Boot app with picocli CLI |
| **Schema Entity** | `ServiceCommand` with Spring Boot integration |

**Example:**
```java
@Component
@Command(name = "migrate", description = "Run migration")
public class MigrateCommand implements ApplicationRunner {
    @Option(names = {"-d", "--dry-run"})
    private boolean dryRun;
    
    @Override
    public void run(ApplicationArguments args) throws Exception {
        new CommandLine(this).execute(args.getSourceArgs());
    }
}
```

**Extraction Notes:**
- Set `ServiceCommand.legacy_config.runner_type = APPLICATION_RUNNER`

---

## Command Definition

### @Option Annotation

| Pattern | `@Option\s*\(\s*names\s*=` |
|---------|----------------------------|
| **File Types** | `*.java` |
| **Indicates** | Command-line option definition |
| **Schema Entity** | `ServiceCommand.options[]` |

**Example:**
```java
@Option(names = {"-f", "--file"}, 
        description = "Input file path",
        required = true,
        defaultValue = "data.csv")
private String filename;
```

**Extraction Notes:**
- Extract short name (e.g., "f") → `CommandOption.short_name`
- Extract long name (e.g., "file") → `CommandOption.long_name`
- Extract description → `CommandOption.description`
- Extract required → `CommandOption.required`
- Extract defaultValue → `CommandOption.default_value`
- Field name → `CommandOption.name` (must map to ServiceOperation input)
- Field type → `CommandOption.type`

---

### @Parameters Annotation

| Pattern | `@Parameters\s*\(` |
|---------|-------------------|
| **File Types** | `*.java` |
| **Indicates** | Positional parameter definition |
| **Schema Entity** | `ServiceCommand.positional_parameters[]` |

**Example:**
```java
@Parameters(index = "0", description = "Source directory")
private File sourceDir;

@Parameters(index = "1", description = "Target directory")
private File targetDir;
```

**Extraction Notes:**
- Extract index → `CommandParameter.index`
- Extract description → `CommandParameter.description`
- Field name → `CommandParameter.name`
- Field type → `CommandParameter.type`

---

## Subcommand Patterns

### @Command with subcommands

| Pattern | `@Command\s*\(.*subcommands\s*=` |
|---------|-----------------------------------|
| **File Types** | `*.java` |
| **Indicates** | Parent command with subcommands |
| **Schema Entity** | Multiple `ServiceCommand` with `parent_command` |

**Example:**
```java
@Command(name = "orders",
         description = "Order management commands",
         subcommands = {
             OrderListCommand.class,
             OrderCancelCommand.class
         })
public class OrdersCommand {}

@Command(name = "list", description = "List all orders")
public class OrderListCommand implements Runnable {
    @Override
    public void run() { /* ... */ }
}

@Command(name = "cancel", description = "Cancel an order")
public class OrderCancelCommand implements Runnable {
    @Option(names = {"-i", "--id"}, required = true)
    private Long orderId;
    
    @Override
    public void run() { /* ... */ }
}
```

**Extraction Notes:**
- Create parent `ServiceCommand` with `command_key = "orders"`
- Create child commands with:
  - `command_key = "orders list"`
  - `command_key = "orders cancel"`
  - `parent_command = "orders"`

---

### @ParentCommand Annotation

| Pattern | `@ParentCommand` |
|---------|------------------|
| **File Types** | `*.java` |
| **Indicates** | Reference to parent command |
| **Schema Entity** | Links subcommand to parent |

**Example:**
```java
@Command(name = "cancel")
public class CancelCommand implements Runnable {
    @ParentCommand
    private OrdersCommand parent;
    
    @Override
    public void run() {
        // Can access parent command properties
    }
}
```

**Extraction Notes:**
- Use to establish `parent_command` relationship

---

## Advanced Features

### @Mixin Annotation

| Pattern | `@Mixin` |
|---------|----------|
| **File Types** | `*.java` |
| **Indicates** | Reusable option set |
| **Schema Entity** | Merge mixin options into command options |

**Example:**
```java
public class CommonOptions {
    @Option(names = {"-v", "--verbose"})
    boolean verbose;
    
    @Option(names = {"-q", "--quiet"})
    boolean quiet;
}

@Command(name = "process")
public class ProcessCommand {
    @Mixin
    CommonOptions common;
    
    @Option(names = {"-f", "--file"})
    String filename;
}
```

**Extraction Notes:**
- Include mixin fields in `ServiceCommand.options`

---

### ICompletionCandidate Interface

| Pattern | `implements\s+ICompletionCandidate\|@Completion` |
|---------|---------------------------------------------------|
| **File Types** | `*.java` |
| **Indicates** | Tab completion support |
| **Schema Entity** | `ServiceCommand.interactive_features.completion_support = true` |

**Example:**
```java
public enum ExportFormat implements ICompletionCandidate {
    CSV, JSON, XML;
    
    @Override
    public String completionValue() {
        return name().toLowerCase();
    }
}
```

**Extraction Notes:**
- Set `completion_support = true` when detected

---

## Extraction Strategy

When detecting picocli commands:

1. **Identify command classes** - Classes annotated with `@Command`
2. **Determine integration type**:
   - Standalone CLI: main class with `CommandLine.execute()`
   - Spring Boot: `@Component` + `CommandLineRunner`/`ApplicationRunner`
3. **Extract command metadata**:
   - Command name, description, aliases
   - Options and parameters
4. **Analyze implementation**:
   - Extract business logic from `run()` or `call()` method
   - Create `ServiceOperation` with:
     - inputs from options/parameters
     - steps from method body
     - structural_events (file I/O, DB ops, messages)
5. **Create entities**:
   - `ServiceCommand` with CLI contract details
   - `ServiceOperation` with business logic
   - Link via `ServiceCommand.operation`

---

## Usage

Use these patterns during Phase 1 (extraction) to identify picocli CLI applications and extract them to the Technical Context Schema for translation to Spring Shell 4 in Phase 2.
