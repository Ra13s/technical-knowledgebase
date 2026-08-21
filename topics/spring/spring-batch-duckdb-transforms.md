# Use DuckDB for set-based Spring Batch transforms

## What it is

For transformation-heavy batch steps, embed DuckDB through JDBC and express the transform as SQL instead of reading millions of rows into Java objects and aggregating them in an `ItemProcessor` or custom loop.

DuckDB runs in-process; no database server is required.

## Use when

Evaluate this pattern when a batch step is mostly:

- grouping and aggregation;
- joins between large file/data sets;
- filtering, sorting, windowing, or derived columns;
- reading analytical formats such as CSV or Parquet and producing another file/table.

Keep normal Spring Batch item-oriented processing when each record needs application-specific side effects, external API calls, per-item retry/skip semantics, or a strict downstream writer.

## Add DuckDB JDBC

```xml
<dependency>
    <groupId>org.duckdb</groupId>
    <artifactId>duckdb_jdbc</artifactId>
    <version>1.5.5</version>
</dependency>
```

`1.5.5` is the stable Java client version documented by DuckDB at the time this entry was written. Pin an explicit tested version in the project rather than copying this forever.

An in-memory connection is enough when DuckDB is only the transform engine:

```java
try (Connection connection = DriverManager.getConnection("jdbc:duckdb:")) {
    // execute transform SQL
}
```

Use a file URL such as `jdbc:duckdb:/path/to/batch.duckdb` only when DuckDB state itself needs to persist across connections/runs.

## Put the transform in a Tasklet

A Spring Batch `Tasklet` is a better fit than pretending a single set-based SQL operation is item-oriented work:

```java
@Bean
Step summarizeOrders(JobRepository jobRepository,
                     PlatformTransactionManager transactionManager) {
    return new StepBuilder("summarize-orders", jobRepository)
        .tasklet((contribution, chunkContext) -> {
            try (Connection connection = DriverManager.getConnection("jdbc:duckdb:");
                 Statement statement = connection.createStatement()) {

                statement.execute("""
                    COPY (
                      SELECT customer_id,
                             category,
                             count(*)               AS order_count,
                             sum(amount * quantity) AS total_revenue,
                             sum(quantity)          AS total_quantity
                      FROM read_csv('orders.csv', header = true)
                      GROUP BY customer_id, category
                    ) TO 'summary.csv' (FORMAT CSV, HEADER true)
                    """);
            }
            return RepeatStatus.FINISHED;
        }, transactionManager)
        .build();
}
```

The Spring Batch job still owns orchestration, execution metadata, retries/restarts at the step level, and sequencing. DuckDB owns the analytical transform inside the step.

## Decision rule

Before implementing a large in-memory Java aggregation in a batch job, ask whether the operation can be expressed naturally as relational SQL.

If yes, prototype the same transform in DuckDB and benchmark both on representative input. Keep the DuckDB version only if it improves the actual workload enough to justify the extra dependency and SQL boundary.

A published comparison using Java 21 and DuckDB 1.5.5 reported roughly 6x speedup at 10 million rows and 8x at 50 million rows for one grouping/aggregation workload. Treat those numbers as workload-specific evidence, not an expected multiplier.

## Operational details

- Make file/object locations job parameters rather than hard-coded paths.
- Do not concatenate untrusted values into SQL. Use parameters for scalar values and validate/allow-list identifiers or file locations that must become SQL syntax.
- For file outputs that downstream systems consume, prefer writing to a temporary output and promoting/renaming it only after the step succeeds, so a failed run does not expose a partial artifact.
- Decide what restart means. A Tasklet gives a coarser restart boundary than chunk processing; make the SQL/output operation idempotent or clean up prior partial results before retry.
- Benchmark with the actual source format and storage. Local CSV performance does not predict remote object-store or database-scanner performance.

## Why it is useful

Analytical SQL lets DuckDB use vectorized execution, parallelism, and column-oriented processing without constructing a Java object graph for every row. It can also replace bespoke aggregation code with a declarative query that is easier to compare against expected results.

## Caveats

- Do not replace chunk processing merely because DuckDB is faster in a synthetic benchmark. Chunk processing gives useful item-level restart, retry, skip, validation, and writer semantics.
- DuckDB is an additional native-backed runtime dependency; include it in platform/container compatibility tests.
- A Spring transaction around the Tasklet does not make arbitrary external files transactionally atomic with the Spring Batch repository.

## Sources

- Foojay example and benchmark: https://foojay.io/today/duckdb-in-spring-batch-replace-in-memory-java-loops-with-one-sql-statement/
- DuckDB Java JDBC client: https://duckdb.org/docs/lts/clients/java
- Spring Batch `Tasklet`: https://docs.spring.io/spring-batch/reference/api/org/springframework/batch/core/step/tasklet/Tasklet.html
- Spring Batch `StepBuilder.tasklet`: https://docs.spring.io/spring-batch/reference/api/org/springframework/batch/core/step/builder/StepBuilder.html
