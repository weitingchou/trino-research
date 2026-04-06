# Module Teardown: SQL-Level Correctness & Reference Validation (Task 6.4.A) [KG-13]

## 0. Research Focus

How does DataFusion validate end-to-end SQL correctness? This research traces the sqllogictest infrastructure -- from the test runner binary through the dual-engine (DataFusion + Postgres) execution model to the `.slt` file format -- to understand:

- How correctness is asserted via inline expected results in `.slt` files
- How Postgres serves as a reference oracle for a curated subset of tests (`pg_compat_*`)
- What query coverage the ~474 `.slt` / `.slt.part` files (~145K lines) provide
- How TPC-H queries double as both correctness and plan regression tests
- What patterns the Rust Trino worker can adopt for validating against the Java coordinator

**Key source files studied:**
- `datafusion/sqllogictest/bin/sqllogictests.rs` -- test runner binary (entry point)
- `datafusion/sqllogictest/bin/postgres_container.rs` -- testcontainers-based Postgres lifecycle
- `datafusion/sqllogictest/src/engines/datafusion_engine/runner.rs` -- DataFusion engine adapter
- `datafusion/sqllogictest/src/engines/postgres_engine/mod.rs` -- Postgres engine adapter
- `datafusion/sqllogictest/src/engines/conversion.rs` -- type normalization (NULL, NaN, floats, etc.)
- `datafusion/sqllogictest/src/filters.rs` -- record/file filtering for targeted test runs
- `datafusion/sqllogictest/src/util.rs` -- value validation including `<slt:ignore>` markers
- `datafusion/sqllogictest/src/test_context.rs` -- per-file SessionContext setup
- `datafusion/sqllogictest/src/test_file.rs` -- test file discovery and priority ordering
- `datafusion/sqllogictest/test_files/` -- all `.slt` test files

---

## 1. High-Level Overview

DataFusion's primary correctness testing strategy is **sqllogictest** (SLT), the same framework originally developed by SQLite. The implementation uses the `sqllogictest` Rust crate (v0.29.1) with a custom driver binary. The core idea: each `.slt` file contains SQL statements and their expected outputs inline. The runner executes every statement/query and compares actual output to the expected text.

### Architecture Summary

```
                                +--------------------+
                                |   sqllogictests    |  (bin/sqllogictests.rs)
                                |   binary entry     |
                                +--------+-----------+
                                         |
                    +--------------------+--------------------+
                    |                    |                    |
             run_test_file    run_test_file_with_postgres   run_complete_file
                    |                    |                    |
            +-------v-------+   +-------v-------+   +-------v-------+
            |  DataFusion   |   |   Postgres    |   |  DataFusion   |
            |  engine       |   |   engine      |   |  (complete    |
            |  (runner.rs)  |   |  (mod.rs)     |   |   mode)       |
            +-------+-------+   +-------+-------+   +-------+-------+
                    |                    |                    |
                    |     impl AsyncDB   |                   |
                    +--------------------+      update_test_file()
                    |                           (writes expected
          +--------+--------+                    results into .slt)
          | .slt test files |
          | (152 top-level  |
          |  + subdirs)     |
          +-----------------+
```

### Three Execution Modes

1. **Normal mode** (`cargo test --test sqllogictests`): Each `.slt` file is run against the DataFusion engine. Expected results are inline in the file. A mismatch is a test failure.

2. **Postgres runner mode** (`--postgres-runner` / `PG_COMPAT=1`): Only `pg_compat_*` files are run. Each `.slt` file is executed against a real Postgres instance (started via testcontainers). The same expected results must match Postgres output. This validates DataFusion-Postgres behavioral equivalence.

3. **Complete mode** (`--complete`): Instead of checking results, the runner executes queries and *writes the actual output back into the `.slt` file* as expected results. This is how expected values are initially generated or updated after intentional behavioral changes.

---

## 2. Detailed Analysis

### 2.1 The `.slt` File Format

DataFusion uses a superset of the standard sqllogictest format. Each file is a sequence of records:

#### Statement Records (DDL/DML, no result verification)

```
statement ok
CREATE TABLE users AS VALUES(1,2),(2,3);

statement error DataFusion error: Execution error: Table 'users' already exists
SELECT * INTO users FROM (VALUES(1,2),(2,3));
```

- `statement ok` -- expects success, ignores output
- `statement error <regex>` -- expects failure matching the regex pattern

#### Query Records (SELECT, result verification)

```
query IIT
SELECT c2, c3, c10
FROM aggregate_test_100_by_sql
ORDER BY c2 ASC, c3 DESC, c10;
----
1 125 17869394731126786457
1 120 16439861276703750332
```

- `query <type_codes>` -- type codes are per-column: `I`=Integer, `R`=Float, `T`=Text, `B`=Boolean, `D`=DateTime, `P`=Timestamp
- After `----`, each line is one expected row, columns separated by spaces
- Optional sort modifier: `query II rowsort` sorts actual output before comparison

#### Error Query Records

```
query error
SELECT COUNT(DISTINCT) FROM aggregate_test_100
```

#### Conditional Execution (Engine-Specific Setup)

```
onlyif postgres
statement ok
CREATE TABLE aggregate_test_100_by_sql (...)

skipif postgres
statement ok
CREATE EXTERNAL TABLE aggregate_test_100_by_sql (...)
```

- `onlyif postgres` -- only runs when engine label is "postgres"
- `skipif postgres` -- skips when engine label is "postgres"
- This is critical for `pg_compat_*` files that need different DDL per engine (Postgres uses standard SQL types; DataFusion uses EXTERNAL TABLE with CSV/Parquet)

#### Include Directive

```
include ./create_tables.slt.part
include ./plans/q*.slt.part
include ./answers/q*.slt.part
```

- Files with `.slt.part` extension are included fragments, not standalone tests
- Glob patterns are supported (e.g., `q*.slt.part`)

#### Ignore Markers

```
query T
select 'DataFusion'
----
<slt:ignore>

query I
select * from generate_series(3);
----
0
1
<slt:ignore>
3
```

The `<slt:ignore>` marker in expected output allows skipping volatile portions (e.g., timestamps, memory addresses). The validator performs fragment-based matching: all non-ignore fragments must appear in order in the actual output.

### 2.2 The Test Runner Binary (sqllogictests.rs)

The runner is a custom Rust binary (not a `#[test]` function) with custom CLI argument parsing:

**Key CLI flags:**
- `--complete` -- auto-complete mode (generate expected results)
- `--postgres-runner` / `PG_COMPAT=1` -- run against Postgres
- `--substrait-round-trip` -- Substrait serialization round-trip before execution
- `--include-sqlite` / `INCLUDE_SQLITE=1` -- include SQLite-derived test files
- `--include-tpch` / `INCLUDE_TPCH=1` -- include TPC-H tests
- Filter arguments (positional) -- substring match on filenames, optionally with `:line_number`

**Parallelism:** Test files are executed in parallel (`buffer_unordered(test_threads)`) using `futures::stream`. Each file gets its own `SessionContext`, ensuring isolation. The number of threads defaults to available parallelism.

**Test priority ordering:** Long-running files (aggregate.slt, joins.slt, imdb.slt, window.slt) are scheduled first to optimize wall-clock time. Priority is hardcoded in `test_file.rs`:

```rust
const TEST_PRIORITY_ENTRIES: &[&str] = &[
    "aggregate.slt",     // longest-running files go first
    "joins.slt",
    "imdb.slt",
    "push_down_filter_regression.slt",
    "aggregate_skip_partial.slt",
    "array.slt",
    "window.slt",
    "group_by.slt",
    "clickbench.slt",
    "datetime/timestamps.slt",
];
```

**Config drift detection:** After each file, the runner checks that no DataFusion configuration options were left modified (via `validate_config_unchanged()`). If a `.slt` file uses `SET` to change config, it must `RESET` before the end. This prevents test-order-dependent failures.

### 2.3 The DataFusion Engine (runner.rs, normalize.rs)

The DataFusion engine adapter implements `sqllogictest::AsyncDB`:

```rust
async fn run(&mut self, sql: &str) -> Result<DFOutput> {
    let df = ctx.sql(sql).await?;
    let plan = df.create_physical_plan().await?;
    let stream = execute_stream(plan, task_ctx)?;
    let results: Vec<RecordBatch> = collect(stream).await?;
    let rows = normalize::convert_batches(&schema, results, is_spark_path)?;
    Ok(DBOutput::Rows { types, rows })
}
```

**Normalization is critical.** The `convert_batches` function converts Arrow RecordBatches to `Vec<Vec<String>>` with careful formatting:
- NULL -> `"NULL"`
- NaN -> `"NaN"` (sign-agnostic, since NaN sign varies by platform)
- Empty strings -> `"(empty)"`
- Floats -> `BigDecimal` string representation for deterministic precision
- Multiline cells (e.g., EXPLAIN output) are expanded: each `\n` becomes a separate row

The `TestContext` (test_context.rs) provides per-file setup:
- Hardcoded `target_partitions = 4` for deterministic plans
- File-specific table registration (e.g., `joins.slt` gets partition tables and a UDF; `avro.slt` gets Avro tables)
- Scratch directory setup per file for INSERT/COPY tests

### 2.4 The Postgres Engine (postgres_engine/mod.rs)

The Postgres engine adapter also implements `sqllogictest::AsyncDB`, connecting to a real Postgres instance:

```rust
pub async fn connect(relative_path: PathBuf, pb: ProgressBar) -> Result<Self> {
    let uri = std::env::var("PG_URI").unwrap_or(PG_URI.to_string());
    let (client, connection) = config.connect(tokio_postgres::NoTls).await?;
    // Create a fresh schema per test file for isolation
    client.execute(&format!("DROP SCHEMA IF EXISTS {schema} CASCADE"), &[]).await?;
    client.execute(&format!("CREATE SCHEMA {schema}"), &[]).await?;
    client.execute(&format!("SET search_path TO {schema}"), &[]).await?;
}
```

Key design decisions:
- **Schema-per-file isolation:** Each `.slt` file gets a fresh Postgres schema, preventing cross-test contamination.
- **COPY rewriting:** `COPY FROM 'filename'` is rewritten to `COPY FROM STDIN` since the Postgres container cannot access the host filesystem. Data is read locally and piped via `COPY ... FROM STDIN`.
- **Type normalization alignment:** Postgres results are normalized to match DataFusion's output format. The `cell_to_string` function parses Postgres text results by type (INT2, FLOAT8, NUMERIC, TIMESTAMP, etc.) and formats them identically to DataFusion's output.

### 2.5 Postgres Container Lifecycle (postgres_container.rs)

When `--postgres-runner` is used without `PG_URI` being set, the runner automatically starts a Postgres container using `testcontainers-modules`:

```rust
let container = postgres::Postgres::default()
    .with_user("postgres")
    .with_password("postgres")
    .with_db_name("test")
    .with_mapped_port(16432, 5432.tcp())
    .with_tag("17-alpine")
    .start().await.unwrap();
```

The container runs Postgres 17 Alpine. A background thread manages the container lifecycle via channels, and the `PG_URI` environment variable is set automatically.

### 2.6 Postgres Compatibility Test Files

Six `pg_compat_*` files validate DataFusion/Postgres behavioral equivalence:

| File | Coverage |
|------|----------|
| `pg_compat_simple.slt` | Basic math, string functions, column selection, ordering |
| `pg_compat_null.slt` | NULL handling in aggregates, COALESCE, IS NULL, DISTINCT |
| `pg_compat_type_coercion.slt` | Boolean logic (AND/OR with NULL), type promotion rules |
| `pg_compat_types.slt` | Type-specific behavior |
| `pg_compat_union.slt` | UNION operations |
| `pg_compat_window.slt` | Window functions: ROW_NUMBER, LEAD, LAG, FIRST_VALUE, LAST_VALUE, NTH_VALUE with PARTITION BY and ORDER BY |

Each file uses the `onlyif postgres` / `skipif postgres` pattern to provide engine-specific table setup while sharing the actual test queries and expected results. The same expected results must pass against both DataFusion and Postgres.

### 2.7 Query Coverage Analysis

The ~474 `.slt` / `.slt.part` files (~145K lines) provide comprehensive coverage:

| Category | Key Files | Coverage |
|----------|-----------|----------|
| **Aggregation** | `aggregate.slt`, `aggregate_skip_partial.slt`, `aggregates_simplify.slt`, `aggregates_topk.slt` | SUM, AVG, COUNT, MIN/MAX, DISTINCT aggregates, partial/full skip, TopK |
| **Joins** | `joins.slt`, `join.slt.part`, `join_disable_repartition_joins.slt`, `sort_merge_join.slt`, `lateral_join.slt` | Inner, outer, cross, anti, semi joins; repartition; sort-merge; lateral |
| **Window Functions** | `window.slt`, `window_limits.slt`, `window_topk_pushdown.slt` | ROW_NUMBER, RANK, LEAD/LAG, frame specs, partition/order |
| **Subqueries** | `subquery.slt`, `subquery_sort.slt` | Correlated, uncorrelated, EXISTS, IN, scalar subqueries |
| **DDL** | `ddl.slt`, `create_external_table.slt`, `create_function.slt` | CREATE/DROP TABLE, external tables, UDFs |
| **DML** | `insert.slt`, `delete.slt`, `update.slt`, `dml_delete.slt`, `dml_update.slt`, `copy.slt` | INSERT, DELETE, UPDATE, COPY |
| **Types** | `cast.slt`, `decimal.slt`, `binary.slt`, `string/`, `datetime/` | Type casting, decimal arithmetic, binary ops, string functions, date/time arithmetic |
| **Explain/Plans** | `explain.slt`, `explain_analyze.slt`, `explain_tree.slt` | Logical/physical plan output verification |
| **Optimization** | `predicates.slt`, `projection_pushdown.slt`, `push_down_filter_*.slt`, `limit_pruning.slt` | Filter pushdown, projection pruning, limit optimization |
| **Benchmarks** | `tpch/`, `clickbench.slt`, `clickbench_extended.slt`, `imdb.slt` | TPC-H Q1-Q22, ClickBench, IMDB queries |
| **Error handling** | `errors.slt` | Expected error messages for invalid SQL, bad casts, missing tables |
| **Data formats** | `parquet.slt`, `csv_files.slt`, `json.slt`, `avro.slt`, `arrow_files.slt` | Format-specific read/write behavior |

### 2.8 TPC-H as Correctness + Plan Regression

The TPC-H test suite (`test_files/tpch/`) serves dual purposes:

**Structure:**
```
tpch/
  tpch.slt              -- orchestrator: includes tables, plans, answers
  create_tables.slt.part -- DDL for 8 TPC-H tables (CSV external tables)
  drop_tables.slt.part   -- cleanup
  plans/q1..q22.slt.part -- EXPLAIN output verification (plan regression)
  answers/q1..q22.slt.part -- query result verification (correctness)
```

**Orchestration** (`tpch.slt`):
```
include ./create_tables.slt.part
include ./plans/q*.slt.part      -- verify plans haven't regressed
include ./answers/q*.slt.part    -- verify answer correctness (hash join)

statement ok
set datafusion.optimizer.prefer_hash_join = false;

include ./answers/q*.slt.part    -- re-verify with sort-merge join

include ./drop_tables.slt.part
```

This runs every TPC-H query twice -- once with hash join, once with sort-merge join -- to validate correctness under both join strategies.

**Plan files** verify the exact EXPLAIN output (both logical and physical plans):
```
query TT
explain select l_returnflag, l_linestatus, ...
----
logical_plan
01)Sort: lineitem.l_returnflag ASC NULLS LAST, ...
02)--Projection: ...
physical_plan
01)SortPreservingMergeExec: [l_returnflag@0 ASC NULLS LAST, ...]
02)--SortExec: ...
```

This detects optimizer regressions -- if a plan changes unexpectedly, the test fails.

**Answer files** verify actual query results with inline expected data:
```
query TTRRRRRRRI
select l_returnflag, l_linestatus, sum(l_quantity) as sum_qty, ...
----
A F 3774200 5320753880.69 5054096266.6828 5256751331.449234 25.537587 ...
N F 95257 133737795.84 127132372.6512 132286291.229445 25.300664 ...
```

### 2.9 Benchmark Suite (benchmarks/)

Separate from the correctness tests, the `benchmarks/` directory provides performance benchmarking:

- **TPC-H** (queries/q1.sql through q22.sql) -- run via `bench.sh run tpch`
- **TPC-DS** (queries referenced in `compare_tpcds.sh`)
- **ClickBench** (queries/clickbench/)
- **IMDB** (src/bin/imdb.rs)
- **H2o.ai db-benchmark**

The benchmark runner (`src/bin/dfbench.rs`) measures execution time rather than correctness. Performance is tracked by comparing results across runs via `compare_tpch.sh` / `compare_tpcds.sh` scripts.

### 2.10 Regression Test Pattern

When a bug is found and fixed, the pattern for regression testing is:

1. **Issue-named files:** Create a dedicated `.slt` file named after the issue (e.g., `issue_17138.slt`). This file contains the minimal reproduction case.

2. **Inline in existing files:** Add test cases to the relevant thematic file (e.g., a window function bug gets added to `window.slt`).

Example from `issue_17138.slt`:
```
statement ok
CREATE TABLE tab1(col0 INTEGER, col1 INTEGER, col2 INTEGER)

statement ok
INSERT INTO tab1 VALUES(51,14,96)

query R
SELECT NULL * AVG(DISTINCT 4) + SUM(col1) AS col0 FROM tab1
----
NULL

query TT
EXPLAIN SELECT NULL * AVG(DISTINCT 4) + SUM(col1) AS col0 FROM tab1
----
logical_plan
01)Projection: Float64(NULL) AS col0
02)--EmptyRelation: rows=1
```

This captures both the correct result AND the expected plan, ensuring the optimization that caused the bug does not regress.

### 2.11 Value Validation and Normalization

The `df_value_validator` function in `util.rs` handles comparison:

```rust
pub fn df_value_validator(
    normalizer: Normalizer,
    actual: &[Vec<String>],
    expected: &[String],
) -> bool {
    // Support <slt:ignore> markers for volatile output
    // Normalize trailing whitespace
    // Join actual row columns with spaces
    // Compare normalized_actual == normalized_expected
}
```

Key normalization rules shared between engines:
- NULL -> `"NULL"` string literal
- Empty string -> `"(empty)"`
- Boolean -> `"true"` / `"false"` (lowercase)
- NaN -> `"NaN"` (sign stripped for platform independence)
- Floats -> `BigDecimal` string (avoids platform-specific float formatting)
- Trailing whitespace trimmed
- Null bytes escaped as `\0`

### 2.12 Filter System for Targeted Execution

The `Filter` struct enables running specific tests:

```rust
// Run only tests in join.slt
cargo test --test sqllogictests -- join.slt

// Run a specific test at line 100 in join.slt
cargo test --test sqllogictests -- join.slt:100
```

The filter system is smart about what can be skipped: DDL statements (CREATE TABLE, INSERT INTO, SELECT INTO) are never skipped even when they don't match a filter, because they set up state for later queries. Only SELECT and EXPLAIN are skippable.

---

## 3. Key Design Insights

1. **Single shared file for two engines.** The `pg_compat_*` pattern is elegant: one `.slt` file contains setup for both DataFusion and Postgres (via `onlyif`/`skipif`), with shared queries and shared expected results. This guarantees behavioral equivalence without duplicating test logic.

2. **`--complete` mode as a result generator.** Rather than hand-writing expected results, developers write queries and use `--complete` to have the engine fill in the `----` sections. This works with both DataFusion and Postgres, making it easy to seed new tests. For Postgres-compatible tests, `--complete --postgres-runner` generates Postgres-authoritative expected values.

3. **Normalization is the hardest problem.** The bulk of the engine adapter code deals with normalizing output values to a canonical string form. Both engines must produce identical strings for the same logical value. This includes handling type-specific formatting (BigDecimal for floats, explicit NULL strings, platform-agnostic NaN, etc.).

4. **Plan regression tests are separate from correctness tests.** TPC-H splits EXPLAIN tests (`plans/`) from result tests (`answers/`). This means optimizer improvements that change plans can update plan files without touching answer files, and vice versa.

5. **Parallel execution with isolation.** Each `.slt` file gets its own `SessionContext` (DataFusion) or schema (Postgres), enabling parallel execution without shared state. Priority scheduling for long-running files optimizes total wall-clock time.

6. **Config drift guard.** The `validate_config_unchanged()` mechanism catches `.slt` files that forget to reset configuration changes. This prevents subtle test-order-dependent failures that would be extremely hard to debug.

7. **Scratch directory isolation.** Files that write to disk (INSERT INTO external table, COPY) use per-file scratch directories. The naming convention is enforced: `join.slt` must write to `scratch/join/`.

8. **`<slt:ignore>` for volatile output.** Rather than accepting any output or doing regex matching, the ignore marker allows precise control: "these parts must match exactly, these parts can be anything." This is much more maintainable than regex-based expected output.

---

## 4. Porting Considerations (DataFusion Patterns -> Trino Rust Worker)

### 4.1 Reference Oracle Pattern

DataFusion validates against Postgres. The Trino Rust worker has a natural reference oracle: **the Java Trino coordinator itself**. The testing pattern would be:

```
[Trino SQL Query] --> [Java Coordinator (reference)] --> expected results
                  --> [Rust Worker]                   --> actual results
                  --> compare
```

A `trino_compat_*.slt` suite could use the same `onlyif`/`skipif` mechanism, with engine labels like `trino_java` and `trino_rust`. The key difference: instead of testcontainers for Postgres, the Java Trino would be started as an external process or Docker container.

### 4.2 Adopting sqllogictest

The Trino Rust worker should absolutely adopt the sqllogictest framework:
- Use the same `sqllogictest` Rust crate (v0.29.1)
- Implement `AsyncDB` for the Rust worker, similar to DataFusion's engine adapter
- Implement `AsyncDB` for the Java Trino coordinator (likely via JDBC or the Trino HTTP client)
- Write `.slt` files with queries and expected results

The `--complete` mode is particularly valuable: run queries against the Java coordinator to generate expected results, then validate the Rust worker against them.

### 4.3 Normalization Challenges

The biggest challenge will be output normalization. Trino and DataFusion have different type systems and formatting conventions. The Rust worker's normalizer must:
- Handle Trino-specific types (e.g., `IPADDRESS`, `HyperLogLog`, `UUID`)
- Match Trino's decimal/float formatting rules exactly
- Handle NULL representation consistently
- Normalize timestamp/date formatting

DataFusion's `conversion.rs` module is a good reference for structuring this normalization code.

### 4.4 Test Coverage Strategy

Start with the highest-value test categories:
1. **TPC-H Q1-Q22** -- Complex queries covering joins, aggregations, subqueries, window functions. Use Trino's TPC-H connector as the data source.
2. **Type coercion / NULL handling** -- Modeled on `pg_compat_type_coercion.slt` and `pg_compat_null.slt`
3. **Window functions** -- A common source of subtle bugs
4. **Error handling** -- Ensure the Rust worker returns the same error conditions as Java Trino

### 4.5 Plan Regression Testing

DataFusion's dual approach (plan tests + answer tests in TPC-H) should be adopted:
- **Answer tests:** Query results must match the Java coordinator
- **Plan tests (Rust-specific):** The Rust worker's physical plan should be validated separately. As the optimizer matures, plan regression tests catch unintended changes.

### 4.6 Practical Workflow

1. **Seed phase:** Run queries against Java Trino with `--complete` mode to generate expected results into `.slt` files
2. **Validation phase:** Run the same `.slt` files against the Rust worker
3. **Regression phase:** When a bug is found, add a test case (issue-named `.slt` file or inline in the appropriate thematic file)
4. **CI integration:** Run the sqllogictest suite in CI on every PR, with the Java Trino coordinator started in a Docker container

### 4.7 Differences from DataFusion's Approach

| Aspect | DataFusion | Trino Rust Worker |
|--------|-----------|-------------------|
| Reference oracle | Postgres (via testcontainers) | Java Trino coordinator (Docker) |
| Table setup | `CREATE EXTERNAL TABLE` (CSV/Parquet) | Trino connectors (TPC-H, memory, Hive) |
| Type system | Arrow types | Trino types (wider variety) |
| Plan testing | EXPLAIN text comparison | Rust worker plan text comparison |
| Scope | Full SQL engine | Worker-level execution only (plan comes from coordinator) |

### 4.8 The "Worker Only" Constraint

A critical difference: the Rust worker receives a physical plan from the Java coordinator -- it does not parse SQL or optimize plans. This means:
- The test harness must send serialized plans (not SQL) to the Rust worker
- SQL-level tests still work for end-to-end validation (coordinator produces the plan, worker executes it)
- A separate "plan execution" test layer is needed where plans are constructed programmatically in Rust, independent of the Java coordinator

This suggests a two-tier testing approach:
1. **End-to-end** (`.slt` files): SQL -> Java coordinator -> plan -> Rust worker -> results -> compare with Java results
2. **Unit/integration** (Rust tests): Manually constructed plans -> Rust worker -> expected results (no Java dependency)
