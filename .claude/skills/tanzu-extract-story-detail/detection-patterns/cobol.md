# COBOL Patterns

This document defines detection patterns for COBOL applications. These patterns help identify COBOL program structures, data definitions, database access, screen handling, and batch processing components that need modernization to Spring Boot.

## Table of Contents

- [Program Structure](#program-structure)
- [Data Definitions](#data-definitions)
- [Database Access](#database-access)
- [Screen Handling (BMS)](#screen-handling-bms)
- [Batch Processing (JCL)](#batch-processing-jcl)
- [COBOL batch program (standalone)](#cobol-batch-program-standalone)
- [SORT verb](#sort-verb)
- [COBOL-Specific Constructs](#cobol-specific-constructs)

---

## Program Structure

### IDENTIFICATION DIVISION

| Pattern | `IDENTIFICATION\s+DIVISION` |
|---------|------------------------------|
| **File Types** | `*.cbl`, `*.cob`, `*.cobol` |
| **Indicates** | COBOL program start |
| **Schema Entity** | `ModernizationRequirements.source_technology = COBOL` |

**Example:**
```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ORDPROC.
       AUTHOR. JOHN DOE.
```

**Extraction Notes:**
- Extract `PROGRAM-ID` as the service/module name
- Use as basis for `ServiceOperation` grouping

---

### PROCEDURE DIVISION

| Pattern | `PROCEDURE\s+DIVISION` |
|---------|------------------------|
| **File Types** | `*.cbl`, `*.cob`, `*.cobol` |
| **Indicates** | Business logic section |
| **Schema Entity** | `ServiceOperation` |

**Example:**
```cobol
       PROCEDURE DIVISION.
       MAIN-PROCESS.
           PERFORM VALIDATE-ORDER.
           PERFORM CALCULATE-TOTAL.
           PERFORM UPDATE-DATABASE.
           STOP RUN.
```

**Extraction Notes:**
- Each PARAGRAPH or SECTION name represents a potential `ServiceOperation`
- `PERFORM` statements indicate operation dependencies/flow

---

### PERFORM Statement

| Pattern | `PERFORM\s+[A-Z0-9-]+` |
|---------|------------------------|
| **File Types** | `*.cbl`, `*.cob`, `*.cobol` |
| **Indicates** | Subroutine/procedure call |
| **Schema Entity** | `ServiceOperation.steps` |

**Example:**
```cobol
           PERFORM VALIDATE-CUSTOMER
           PERFORM CALCULATE-DISCOUNT THRU CALCULATE-DISCOUNT-EXIT
           PERFORM PROCESS-LINE-ITEMS VARYING I FROM 1 BY 1
               UNTIL I > LINE-ITEM-COUNT
```

**Extraction Notes:**
- Map paragraph names to potential method names
- Track execution flow for business logic extraction

---

## Data Definitions

### DATA DIVISION / WORKING-STORAGE

| Pattern | `WORKING-STORAGE\s+SECTION` |
|---------|------------------------------|
| **File Types** | `*.cbl`, `*.cob`, `*.cobol` |
| **Indicates** | Program variables and data structures |
| **Schema Entity** | `DataStructure` |

**Example:**
```cobol
       WORKING-STORAGE SECTION.
       01  WS-ORDER-RECORD.
           05  WS-ORDER-ID         PIC 9(10).
           05  WS-CUSTOMER-ID      PIC 9(8).
           05  WS-ORDER-DATE       PIC 9(8).
           05  WS-TOTAL-AMOUNT     PIC 9(7)V99.
           05  WS-STATUS           PIC X(10).
```

**Extraction Notes:**
- `01` level items are record structures → `DataStructure`
- `05`/`10`/etc. levels are fields → `DataStructure.fields`
- `PIC 9(n)` → numeric types
- `PIC X(n)` → string types
- `V` indicates an **implied** decimal point (not stored in the file). For `LINE SEQUENTIAL` files, numeric values are written in **display form with a literal `.` character** (e.g. `PIC 9(6)V99` stores `075.25` = 6 characters on disk, not 8). For `RECORDING MODE F` (fixed binary/packed) files the decimal is truly implied and the field occupies only the digit count. Always verify the physical character width by checking the file organisation and inspecting a sample data file.

---

### COPY Statement (Copybooks)

| Pattern | `COPY\s+[A-Z0-9]+` |
|---------|---------------------|
| **File Types** | `*.cbl`, `*.cob`, `*.cobol` |
| **Indicates** | Shared data structure include |
| **Schema Entity** | `DataStructure` (reference to copybook) |

**Example:**
```cobol
       COPY ORDRREC.
       COPY CUSTREC.
```

**Extraction Notes:**
- Copybooks define reusable data structures
- Parse referenced copybook files (`.cpy`) for structure definitions
- Map to shared DTOs or entities

---

### FILE SECTION

| Pattern | `FILE\s+SECTION` |
|---------|-------------------|
| **File Types** | `*.cbl`, `*.cob`, `*.cobol` |
| **Indicates** | File I/O definitions |
| **Schema Entity** | `IntegrationPoint.type = FILE_SYSTEM` or `ServiceBatchProcess.input_sources` |

**Example:**
```cobol
       FILE SECTION.
       FD  ORDER-FILE
           RECORDING MODE IS F
           BLOCK CONTAINS 0 RECORDS.
       01  ORDER-RECORD.
           05  ORD-ID              PIC 9(10).
           05  ORD-CUSTOMER        PIC 9(8).
           05  ORD-AMOUNT          PIC 9(7)V99.
```

**Extraction Notes:**
- Extract file names and record layouts
- Identify file organization (sequential, indexed, relative)
- Map to Spring Batch `ItemReader`/`ItemWriter` patterns
- When the program has no `EXEC CICS` and no BMS usage, also create or contribute to a `ServiceBatchProcess`: populate `input_sources` and `output_targets` from FD/SD and `SELECT ASSIGN TO` paths
- **IMPORTANT — physical field widths for numeric fields**: When computing total record length, use the **physical character count** actually written to the file, not the COBOL digit count alone. For `LINE SEQUENTIAL` files, a field declared `PIC 9(m)V9(n)` is written in display form with a literal decimal point: physical width = `m + 1 + n`. For example, `PIC 9(6)V99` = 6 integer digits + `.` + 2 fractional digits = **6 characters** (e.g. `075.25`). Always cross-check by inspecting a sample data file.
- **MANDATORY — verify column ranges against a physical data file**: After computing column ranges from PIC declarations, measure the actual line length of a sample data file with `awk '{print length, $0}' input/file.dat | head -5`. If the measured length differs from the declaration-derived total, use the measured length and document the discrepancy. **Never publish column ranges in the spec until this check has been performed** — wrong ranges cause `IncorrectLineLengthException` at runtime. Record the verified column ranges in the `input_sources.format` field as: `"FIXED_WIDTH N chars/record (field1 cols 1-A, field2 cols B-C, ...)"`.
- **Leading spaces in character fields**: For `LINE SEQUENTIAL` files, character fields (`PIC X(n)`) may begin with a space character that is semantically meaningful (e.g. the first character of a description column). Do NOT trim field values during parsing — use raw token access (e.g. `fieldSet.getValues()` in Spring Batch) instead of `readString()`, which trims both leading and trailing whitespace and will silently corrupt fixed-width alignment.

---

### 88-Level Conditions

| Pattern | `88\s+[A-Z0-9-]+\s+VALUE` |
|---------|---------------------------|
| **File Types** | `*.cbl`, `*.cob`, `*.cobol` |
| **Indicates** | Condition names / business rules |
| **Schema Entity** | Enums or validation logic |

**Example:**
```cobol
       05  WS-ORDER-STATUS         PIC X(10).
           88  STATUS-PENDING      VALUE 'PENDING'.
           88  STATUS-APPROVED     VALUE 'APPROVED'.
           88  STATUS-SHIPPED      VALUE 'SHIPPED'.
           88  STATUS-CANCELLED    VALUE 'CANCELLED'.
```

**Extraction Notes:**
- Map to Java enums
- Extract as validation rules
- Document business state machines

---

## Database Access

### Embedded SQL

| Pattern | `EXEC\s+SQL` |
|---------|--------------|
| **File Types** | `*.cbl`, `*.cob`, `*.cobol` |
| **Indicates** | Database operations |
| **Schema Entity** | `ExternalResourceRequirement.type = DATASOURCE`, `ServiceOperation.structural_events` |

**Example:**
```cobol
           EXEC SQL
               SELECT CUSTOMER_NAME, CUSTOMER_ADDRESS
               INTO :WS-CUST-NAME, :WS-CUST-ADDR
               FROM CUSTOMERS
               WHERE CUSTOMER_ID = :WS-CUSTOMER-ID
           END-EXEC.
```

**Extraction Notes:**
- Extract SQL statements for query patterns
- Identify tables accessed → potential `DataStructure` entities
- Map to Spring Data JPA repositories or `JdbcTemplate`

---

### EXEC CICS

| Pattern | `EXEC\s+CICS` |
|---------|---------------|
| **File Types** | `*.cbl`, `*.cob`, `*.cobol` |
| **Indicates** | CICS transaction processing |
| **Schema Entity** | Multiple (depends on CICS command) |

**Example:**
```cobol
           EXEC CICS
               READ FILE('ORDFILE')
               INTO(WS-ORDER-RECORD)
               RIDFLD(WS-ORDER-KEY)
               RESP(WS-RESP)
           END-EXEC.
           
           EXEC CICS
               SEND MAP('ORDMAP')
               MAPSET('ORDMAPS')
               FROM(ORDER-MAP-DATA)
           END-EXEC.
```

**Extraction Notes:**
- `READ/WRITE/REWRITE/DELETE FILE` → File or database operations
- `SEND MAP/RECEIVE MAP` → UI interactions (map to REST endpoints)
- `LINK/XCTL` → Service calls
- `SYNCPOINT` → Transaction boundaries

---

## Screen Handling (BMS)

### BMS Map Definitions

| Pattern | `DFHMSD\|DFHMDI\|DFHMDF` |
|---------|--------------------------|
| **File Types** | `*.bms`, `*.map` |
| **Indicates** | Screen (UI) definitions |
| **Schema Entity** | `ServiceEndpoint` (screen → REST endpoint), `DataStructure` (for DTOs) |

**Example:**
```
ORDMAP   DFHMDI SIZE=(24,80),LINE=1,COLUMN=1
ORDERID  DFHMDF POS=(3,2),LENGTH=10,ATTRB=(UNPROT,NUM)
CUSTNAME DFHMDF POS=(5,2),LENGTH=30,ATTRB=(PROT)
AMOUNT   DFHMDF POS=(7,2),LENGTH=12,ATTRB=(UNPROT,NUM)
```

**Extraction Notes:**
- Each map represents a user interaction → potential `ServiceEndpoint`
- Field definitions → DTO fields with validation
- `ATTRB` attributes indicate validation rules (UNPROT=editable, NUM=numeric)

---

## Batch Processing (JCL)

### JCL Job Definition

| Pattern | `//[A-Z0-9]+\s+JOB` |
|---------|----------------------|
| **File Types** | `*.jcl`, `*.job` |
| **Indicates** | Batch job definition |
| **Schema Entity** | `ServiceBatchProcess` |

**Example:**
```jcl
//BILLJOB  JOB (ACCT),'MONTHLY BILLING',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=BILLPROC
//INFILE   DD DSN=PROD.CUSTOMER.FILE,DISP=SHR
//OUTFILE  DD DSN=PROD.INVOICE.FILE,DISP=(NEW,CATLG)
//SYSOUT   DD SYSOUT=*
```

**Extraction Notes:**
- Job name → `ServiceBatchProcess.name`
- Each `EXEC PGM=` → `BatchStep`
- `DD` statements → `input_sources` and `output_targets`
- Step dependencies (COND parameters) → step flow

---

### JCL Step Definition

| Pattern | `//[A-Z0-9]+\s+EXEC\s+PGM=` |
|---------|------------------------------|
| **File Types** | `*.jcl`, `*.job` |
| **Indicates** | Batch step executing a program |
| **Schema Entity** | `ServiceBatchProcess.steps` |

---

## COBOL batch program (standalone)

A COBOL program is treated as a **batch program** (and mapped to `ServiceBatchProcess`) when it has file I/O and is not online or screen-driven. Use this when no JCL is present.

**Decision rule (apply to each COBOL source file):**

| Condition | Pattern / check |
|-----------|------------------|
| Has file I/O | `FILE\s+SECTION` and at least one `SELECT\s+\w+\s+ASSIGN` in the same program |
| Not online | File does **not** contain `EXEC\s+CICS` |
| Not screen-driven | No BMS in scope (when analyzing only `.cbl`/`.cob`/`.cobol`, assume no BMS unless copybooks or separate `.bms`/`.map` are linked) |

| Pattern | `FILE\s+SECTION` and `SELECT\s+\w+\s+ASSIGN` (same file); absence of `EXEC\s+CICS` |
|---------|----------------------------------------------------------------------------------------|
| **File Types** | `*.cbl`, `*.cob`, `*.cobol` |
| **Indicates** | Standalone batch program (no JCL) |
| **Schema Entity** | `ServiceBatchProcess` |

**Extraction Notes:**
- Use `PROGRAM-ID` as the batch job name (and as the primary step name for a single-step job)
- Set `legacy_config.type = COBOL_JCL`, `legacy_config.program_name = PROGRAM-ID`; omit `legacy_config.jcl_name` when no JCL exists
- Derive `input_sources` from each `SELECT ... ASSIGN TO` that is opened as INPUT or used in INPUT PROCEDURE; derive `output_targets` from ASSIGN that is opened as OUTPUT or EXTEND
- Set `trigger` to `MANUAL` or `FILE_ARRIVAL` as appropriate (default `MANUAL` if unknown)

---

## SORT verb

| Pattern | `SORT\s+\w+` |
|---------|---------------|
| **File Types** | `*.cbl`, `*.cob`, `*.cobol` |
| **Indicates** | Batch sort step (read → sort → write) |
| **Schema Entity** | `ServiceBatchProcess.steps` |

**Optional patterns for extraction detail:** `INPUT\s+PROCEDURE`, `OUTPUT\s+PROCEDURE`, `ON\s+ASCENDING\s+KEY`, `ON\s+DESCENDING\s+KEY`

**Example:**
```cobol
           SORT SORT-WORK
               ON ASCENDING KEY SORT-DATE
               ON ASCENDING KEY SORT-AMOUNT
               INPUT PROCEDURE IS READ-INPUT
               OUTPUT PROCEDURE IS WRITE-OUTPUT
```

**Extraction Notes:**
- Map INPUT PROCEDURE to reader/source; OUTPUT PROCEDURE to writer/destination
- Extract sort keys from `ON ASCENDING KEY` / `ON DESCENDING KEY` for translation
- Represent as a **TASKLET** step (not CHUNK): the COBOL `SORT` verb is a full-dataset operation — all records must be in memory before any record is written. Mapping it to a chunk-oriented step introduces `ItemStream` open/close lifecycle complexity that causes silent data loss (0 records written). The correct translation is a single `Tasklet` that reads all records into a `List`, sorts in-memory, then writes all records in one pass. Document sort keys and file roles so translation can implement this pattern.
- In the spec `steps` array, set `type: TASKLET` (not `CHUNK`) for any step derived from a SORT verb.

---

## COBOL-Specific Constructs

### CALL Statement

| Pattern | `CALL\s+['"][A-Z0-9]+['"]` |
|---------|----------------------------|
| **File Types** | `*.cbl`, `*.cob`, `*.cobol` |
| **Indicates** | External program call |
| **Schema Entity** | `IntegrationPoint` or `ServiceOperation.dependencies` |

**Example:**
```cobol
           CALL 'VALIDATE' USING WS-ORDER-RECORD WS-RESULT.
           CALL 'CALCDISC' USING WS-CUSTOMER-TYPE WS-AMOUNT WS-DISCOUNT.
```

**Extraction Notes:**
- Called programs may be separate services or utilities
- Track for microservice boundary analysis

---

### EVALUATE Statement

| Pattern | `EVALUATE\s+` |
|---------|---------------|
| **File Types** | `*.cbl`, `*.cob`, `*.cobol` |
| **Indicates** | Business decision logic (switch/case) |
| **Schema Entity** | Business rules documentation |

**Example:**
```cobol
           EVALUATE WS-ORDER-STATUS
               WHEN 'PENDING'
                   PERFORM PROCESS-PENDING-ORDER
               WHEN 'APPROVED'
                   PERFORM SHIP-ORDER
               WHEN OTHER
                   PERFORM HANDLE-INVALID-STATUS
           END-EVALUATE.
```

**Extraction Notes:**
- Extract business rules and state transitions
- Document in `ServiceOperation.steps` or as separate business rule entities
