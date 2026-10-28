# Spring Boot 4 Translation: ServiceBatchProcess

Map **ServiceBatchProcess** to Spring Batch 6 (Job, Step, ItemReader, ItemProcessor, ItemWriter). For the core translation guide, see [../SKILL.md](../SKILL.md).

## Patterns

- **Job:** Define `Job` with `Step`(s); map trigger (SCHEDULED, FILE_ARRIVAL, etc.) to job launcher or scheduler.
- **Chunk step:** Map reader, processor, writer from spec; set chunk size.
- **Restart policy:** Map to batch metadata and restartability.

For a worked end-to-end example, see [translation-spring-boot-4-example-rest-crud.md](translation-spring-boot-4-example-rest-crud.md).

## Spring Batch 6 package changes (Spring Boot 4)

Spring Batch 6 (shipped with Spring Boot 4 via `spring-boot-starter-batch`) reorganized its core packages. The following imports from Spring Batch 5 no longer exist — use the Spring Batch 6 equivalents shown below or the code will not compile.

### Renamed / moved classes

| Spring Batch 5 import | Spring Batch 6 import |
|---|---|
| `org.springframework.batch.core.Job` | `org.springframework.batch.core.job.Job` |
| `org.springframework.batch.core.Step` | `org.springframework.batch.core.step.Step` |
| `org.springframework.batch.core.JobExecution` | `org.springframework.batch.core.job.JobExecution` |
| `org.springframework.batch.core.JobParameters` | `org.springframework.batch.core.job.parameters.JobParameters` |
| `org.springframework.batch.core.JobParametersBuilder` | `org.springframework.batch.core.job.parameters.JobParametersBuilder` |
| `org.springframework.batch.core.StepContribution` | `org.springframework.batch.core.step.StepContribution` |
| `org.springframework.batch.repeat.RepeatStatus` | `org.springframework.batch.infrastructure.repeat.RepeatStatus` |

Unchanged (same package in both Batch 5 and 6): `JobBuilder`, `StepBuilder`, `JobRepository`, `BatchStatus`, `ChunkContext`, `JobLauncher`.
`@SpringBatchTest` stays at `org.springframework.batch.test.context.SpringBatchTest`.

### PlatformTransactionManager — required companion dependency

`spring-boot-starter-batch` does **not** auto-configure a `PlatformTransactionManager` on its own. Every Step (including Tasklet steps) requires one at construction time. Without it the application context fails to start with:

```
No qualifying bean of type 'org.springframework.transaction.PlatformTransactionManager'
```

Add `spring-boot-starter-jdbc` to provide `DataSourceTransactionManager`, or `spring-boot-starter-data-jpa` if JPA entities are also needed:

```xml
<!-- Required companion for spring-boot-starter-batch in Spring Boot 4 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

In start.spring.io terms: add `jdbc` alongside `batch` in the `dependencies` parameter. If the story uses JPA entities use `data-jpa` instead — it implies `jdbc`.

## COBOL-sourced batch

When `legacy_config.type` is `COBOL_JCL` (spec came from COBOL batch extraction):

- **Job/step naming:** If `legacy_config.jcl_name` is present, use it for the Job name; otherwise use `legacy_config.program_name` (e.g. COBOL PROGRAM-ID) as the job name.
- **File sources/targets:** Map `input_sources` and `output_targets` with `type: FILE` and `location` to Spring Batch `Resource` paths (e.g. `file:${input.path}`). Use `FlatFileItemReader` / `FlatFileItemWriter` (or `FlatFileItemReaderBuilder` / `FlatFileItemWriterBuilder`) with record layout derived from the spec (e.g. from DataStructure or extraction notes).
- **SORT — use a Tasklet, not a chunk step:** The COBOL `SORT` verb is a full-dataset operation — all records must be held in memory before any record is written. **Do not** implement it as a chunk-oriented step with a custom `ItemStreamWriter` delegate: Spring Batch's `ItemStream` open/close lifecycle will close the delegate writer between chunks, causing silent data loss (0 records written to the output file). The correct translation is a single **`Tasklet`** that:
  1. Opens a `FlatFileItemReader`, reads all records into a `List<T>`, closes the reader.
  2. Sorts the list in-memory using a `Comparator` derived from the spec's sort keys.
  3. Opens the output file in append mode and writes all records in one pass.

  This directly mirrors the COBOL behavior and avoids `ItemStream` complexity entirely.

### Fixed-width COBOL file I/O — common pitfalls

#### `FixedLengthTokenizer` — always set `strict(false)`

The `FixedLengthTokenizer` defaults to `strict = true`, which throws `IncorrectLineLengthException` if any line is shorter than the maximum declared column end. For `LINE SEQUENTIAL` COBOL files, trailing spaces are stripped on write, so many lines will be shorter than the theoretical maximum. Always call `tokenizer.setStrict(false)`.

#### Do not use `readString()` for fixed-width fields — use `getValues()` instead

`DefaultFieldSet.readString(name)` **trims both leading and trailing whitespace** from the token value. For fixed-width COBOL records, leading spaces in a field are often semantically significant — for example, a description column may begin with a space character that separates it visually from an immediately preceding 10-char type field with no trailing padding. Using `readString()` silently strips this leading space, causing adjacent fields to concatenate without separation when the record is written back.

Use raw token access instead:

```java
FieldSetMapper<MyRecord> mapper = fieldSet -> {
    String[] values = fieldSet.getValues();  // raw, untrimmed
    String date   = values[0].trim();        // trim only fields where trimming is safe
    String amount = values[1].trim();
    String type   = values[2];               // preserve: may be exactly PIC X(10) with no trailing space
    String desc   = values[3];               // preserve: may have a leading space
    return new MyRecord(date, amount, type, desc);
};
```

#### Use `Locale.US` for decimal formatting

`String.format("%06.2f", amount)` uses the JVM default locale. On non-English systems (e.g. Spanish, French, German), this produces a comma decimal separator (`075,25`) instead of a dot (`075.25`), which will not match the legacy file format and will break any downstream parser.

Always specify `Locale.US`:

```java
String amountStr = String.format(Locale.US, "%06.2f", amount);
```

This applies to all numeric fields written to fixed-width output files.

### Post-translation verification (COBOL-sourced batch)

After the job compiles and unit/integration tests pass, run the modernized app against the original legacy input file and diff its output against the original legacy output:

```bash
# Run against real legacy input — use the actual env var names from application.yml
INPUT_FILE=../source-app/input/file.dat \
OUTPUT_FILE=/tmp/modern-output.dat \
mvn spring-boot:run -f {target-folder}/pom.xml

# Diff against the original COBOL output
diff /tmp/modern-output.dat ../source-app/output/expected-output.dat
```

A clean diff (no output) confirms byte-identical output to the original COBOL program. Any difference must be investigated before the migration is considered complete. Common causes:

- Decimal separator mismatch — ensure `Locale.US` is used in all numeric format calls (see above).
- Trimmed leading spaces — ensure `getValues()` is used instead of `readString()` (see above).
- Wrong column ranges — verify with `awk '{print length, $0}'` against the input file (see [cobol.md](../../tanzu-extract-story-detail/detection-patterns/cobol.md)).
