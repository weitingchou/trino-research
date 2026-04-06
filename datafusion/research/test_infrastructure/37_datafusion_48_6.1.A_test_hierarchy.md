# Module Teardown: Test Organization & Infrastructure (Task 6.1.A) [KG-13]

## 0. Research Focus

This document analyzes DataFusion's complete testing infrastructure to inform the
testing strategy for a Rust-based Trino worker replacement. The focus areas are:

- **Test categories**: unit tests, integration tests, sqllogictest, fuzz tests, benchmarks, doc tests
- **Organizational patterns**: per-crate `#[cfg(test)]` modules, `tests/` directories, dedicated test crates
- **sqllogictest infrastructure**: how `.slt` files work, what they cover, how expected results are maintained
- **CI infrastructure**: GitHub Actions workflows, test matrix, feature flags, build profiles
- **Shared test utilities**: common helpers, fixture data, test data generation

**Source version**: DataFusion 53.0.0 (Rust 1.88.0 MSRV)
**Key paths examined**:
- `Cargo.toml` (workspace root)
- `.github/workflows/rust.yml`, `extended.yml`, `dev.yml`
- `datafusion/sqllogictest/`
- `datafusion/core/tests/`
- `test-utils/`
- `benchmarks/`
- Per-crate `src/` directories for `#[cfg(test)]` modules

---

## 1. High-Level Overview

DataFusion follows a layered **test pyramid** with six distinct testing tiers,
from fast, narrow unit tests up through full SQL-level regression suites and
multi-million-query compatibility checks.

### Test Tier Summary

| Tier | Mechanism | Scope | When Run |
|------|-----------|-------|----------|
| **Unit tests** | `#[cfg(test)]` modules in source files | Per-function/module logic | Every PR (`cargo test`) |
| **Integration tests** | `tests/` directories in crates | Cross-module, public API | Every PR (`cargo test`) |
| **sqllogictest** | `.slt` files in `datafusion/sqllogictest/test_files/` | End-to-end SQL behavior | Every PR (fast), PR merge (extended) |
| **Fuzz tests** | `datafusion/core/tests/fuzz_cases/` | Randomized correctness for joins, sorts, aggregates | Extended (post-merge), gated by `extended_tests` feature |
| **Benchmarks** | `benchmarks/` crate + `datafusion/core/benches/` | Performance regression detection | Manual / CI verification |
| **Doc tests** | Rust `#[doc]` examples + user guide examples | API correctness + documentation accuracy | Every PR (`cargo test --doc`) |

### Workspace Structure (68 members)

The workspace has 68 crate members. Of these, the key testing-related members are:

- **`datafusion/sqllogictest`** -- Dedicated crate housing the sqllogictest runner and 474 `.slt` files (~145,000 lines)
- **`test-utils`** -- Shared test utility crate (batch staggering, data generation, TPC-H/TPC-DS schema helpers)
- **`benchmarks`** -- Dedicated benchmark crate with TPC-H, TPC-DS, ClickBench, H2O, IMDB workloads
- **`datafusion/core`** -- Houses the bulk of Rust integration tests in `tests/` and Criterion micro-benchmarks in `benches/`

---

## 2. Detailed Analysis

### 2.1 Unit Tests (`#[cfg(test)]` modules)

DataFusion follows standard Rust convention: unit tests live inside the source
file they test, within a `#[cfg(test)] mod tests { ... }` block.

**Scale**: The `physical-plan` crate alone has 66 source files containing
`#[cfg(test)]` blocks (73 total occurrences). The `optimizer` crate has 33 such
files. Across all `datafusion/` sub-crates, this pattern is pervasive -- virtually
every non-trivial source module has co-located unit tests.

**Pattern**: Unit tests typically:
- Test individual functions/methods in isolation
- Use `assert_batches_eq!` / `assert_batches_sorted_eq!` macros from `datafusion-common::test_util`
- Use `assert_contains!` / `assert_not_contains!` for string-based output checks
- Create small `RecordBatch` fixtures inline

**Per-crate test helper modules**: Some crates have dedicated non-test helper modules
for building test fixtures. For example:
- `datafusion/physical-plan/src/test.rs` -- Provides `MockExec`, `TestStream`,
  `exec_from_batches()`, and other utilities for constructing mock execution plans.
  This is a *public* module (not `#[cfg(test)]`), usable by downstream crates.

### 2.2 Integration Tests (`tests/` directories)

Eight crates maintain `tests/` directories for integration-level tests:

| Crate | `tests/` Contents | Key Focus |
|-------|-------------------|-----------|
| **`core`** | 25+ test modules via `core_integration.rs` hub | SQL execution, dataframes, datasources, memory limits, fuzz, parquet, serde, catalogs, tracing, user-defined extensions |
| **`sql`** | `sql_integration.rs` + `cases/` dir | SQL planner: `plan_to_sql`, params, diagnostics (uses `insta` snapshot testing) |
| **`optimizer`** | `optimizer_integration.rs` | Optimizer pass validation |
| **`physical-expr`** | `tests/` | Physical expression evaluation |
| **`proto`** | `tests/` | Protobuf serialization round-trips |
| **`substrait`** | `substrait_integration.rs` + `cases/` | Substrait plan round-trips (heavy `insta` snapshot testing) |
| **`ffi`** | `tests/` | Foreign function interface validation |
| **`datasource-arrow`** | `tests/` | Arrow datasource tests |

**`core` integration test organization**: The `core_integration.rs` file acts as a
top-level hub using Rust module includes:

```rust
mod sql;
mod dataframe;
mod datasource;
mod macro_hygiene;
mod execution;
mod expr_api;
mod fifo;
mod memory_limit;
mod custom_sources_cases;
mod optimizer;
mod physical_optimizer;
mod serde;
mod catalog;
mod catalog_listing;
mod tracing;
```

Each of these is a directory under `tests/` containing multiple `.rs` files. This
single-binary approach compiles all integration tests into one binary, reducing
link time.

**Fuzz tests**: Gated behind the `extended_tests` feature flag. The `fuzz.rs` entry
point conditionally includes `fuzz_cases/`:

```rust
#[cfg(feature = "extended_tests")]
mod fuzz_cases;
```

Fuzz test cases cover:
- `aggregate_fuzz.rs` -- Randomized aggregation queries
- `join_fuzz.rs` -- Randomized join scenarios
- `sort_fuzz.rs` / `sort_query_fuzz.rs` -- Sort correctness
- `merge_fuzz.rs` -- Sort-preserving merge correctness
- `limit_fuzz.rs` -- Limit operator correctness
- `window_fuzz.rs` -- Window function correctness
- `spilling_fuzz_in_memory_constrained_env.rs` -- Spill behavior under memory pressure
- `sort_preserving_repartition_fuzz.rs` -- Repartition ordering guarantees
- `topk_filter_pushdown.rs` -- TopK optimization correctness

**Test data**: `datafusion/core/tests/data/` contains 59+ fixture files including:
- CSV files (`aggregate_simple.csv`, `cars.csv`, etc.)
- Parquet files (`clickbench_hits_10.parquet`, `int96_nested.parquet`, etc.)
- JSON files (`1.json` through `4.json`, `json_array.json`, etc.)
- Directories for edge cases (`empty_files/`, `filter_pushdown/`, etc.)

### 2.3 sqllogictest Infrastructure

This is DataFusion's **primary SQL regression testing mechanism** and the
recommended first-line test for contributors.

#### Architecture

The sqllogictest system is a dedicated workspace crate (`datafusion-sqllogictest`)
that wraps the `sqllogictest` Rust library (v0.29.1, from `sqllogictest-rs`).

**Key source files**:
- `bin/sqllogictests.rs` -- Entry point; custom `#[test]` harness (`harness = false`).
  Discovers `.slt` files, runs them in parallel via `futures::stream`, reports errors.
- `src/engines/datafusion_engine/` -- `DataFusion` engine adapter implementing
  `AsyncDB` trait for the sqllogictest runner
- `src/engines/postgres_engine/` -- `Postgres` engine adapter for compatibility testing
- `src/engines/datafusion_substrait_roundtrip_engine/` -- Substrait round-trip engine
- `src/test_context.rs` -- Per-file `SessionContext` setup with file-specific
  customization (e.g., registering special tables for `joins.slt`, `avro.slt`, etc.)
- `src/engines/conversion.rs` -- Type conversion rules (NULL rendering, float rounding, etc.)
- `src/filters.rs` -- Test filtering logic (file name patterns, line ranges)
- `src/util.rs` -- Scratch directory management, file discovery

#### .slt File Coverage

**Total**: 474 `.slt` / `.slt.part` files containing ~145,000 lines of SQL test cases.

**Top-level files** (152 entries in `test_files/`):
- Core SQL: `aggregate.slt`, `select.slt`, `joins.slt`, `window.slt`, `union.slt`, `cte.slt`
- DDL/DML: `ddl.slt`, `insert.slt`, `delete.slt`, `update.slt`, `copy.slt`
- Types: `cast.slt`, `decimal.slt`, `binary.slt`, `struct.slt`, `map.slt`, `array.slt`
- Explain: `explain.slt`, `explain_analyze.slt`, `explain_tree.slt`
- Optimizer: `simplify_expr.slt`, `simplify_predicates.slt`, `eliminate_outer_join.slt`
- Datasources: `parquet.slt`, `parquet_filter_pushdown.slt`, `csv_files.slt`, `json.slt`, `avro.slt`
- External benchmarks: `clickbench.slt`, `clickbench_extended.slt`, `imdb.slt`

**Subdirectories**:

| Directory | File Count | Coverage |
|-----------|-----------|----------|
| `spark/` | 239 slt files | Spark-compatible SQL behavior (20 subdirs: aggregate, array, string, datetime, math, etc.) |
| `datetime/` | 19 slt files | Date/time arithmetic and functions |
| `string/` | 5 slt files + parts | String functions and views |
| `regexp/` | 5 slt files | Regex functions (match, replace, count, instr, like) |
| `pg_compat/` | 6 slt files | Postgres compatibility (null, simple, type_coercion, types, union, window) |
| `tpch/` | TPC-H query files | TPC-H benchmark queries (requires external data generation) |
| `min_max/` | Min/max tests | Min/max optimization tests |

#### .slt File Format

Each `.slt` file is a sequence of records following the sqllogictest spec:

```
# comment
statement ok
CREATE TABLE foo AS VALUES (1);

query I
SELECT * from foo;
----
1

statement error <regex>
SELECT * FROM nonexistent;
```

Key format elements:
- **`statement ok`** -- DDL/DML that should succeed (no output checked)
- **`statement error <pattern>`** -- Statement expected to fail with matching error
- **`query <type_string> [sort_mode]`** -- Query with expected results below `----`
- **Type string**: `I` (integer), `R` (float), `T` (text), `B` (boolean), `D` (datetime), `P` (timestamp), `?` (any)
- **Sort modes**: `nosort` (default), `rowsort`, `valuesort`
- **`<slt:ignore>`** -- Wildcard marker for volatile output (timestamps, metrics)
- **`.slt.part`** files -- Include-able fragments (e.g., `join.slt.part`, `create_tables.slt.part`)

#### Expected Result Maintenance

Results are maintained via **`--complete` mode**: running with `--complete` automatically
fills in or updates expected output in `.slt` files:

```bash
cargo test --test sqllogictests -- aggregate --complete
```

This is a critical design decision: expected results are auto-generated, not
hand-written. This eliminates the recompile/relink cycle when iterating on tests
and makes bulk updates feasible when output format changes.

#### Execution Model

- Each `.slt` file runs in its **own isolated `SessionContext`**
- Files run in **parallel** (controlled by `--test-threads` / available parallelism)
- Scratch directories: `test_files/scratch/<filename>/` per file (auto-created/cleared)
- Per-file setup: `TestContext::try_new_for_test_file()` dispatches on filename to
  register special tables/UDFs for specific tests
- Progress reporting with `indicatif` progress bars
- Error limit: up to 10 errors per file before stopping

#### Postgres Compatibility Mode

Files prefixed with `pg_compat_` can run against both DataFusion and a real
Postgres instance. CI runs this with a Postgres service container:

```bash
PG_COMPAT=true PG_URI="postgresql://..." cargo test --features=postgres --test sqllogictests
```

#### Substrait Round-Trip Mode

An experimental mode converts `SQL -> DF logical -> Substrait -> DF logical -> DF physical -> execute`,
verifying Substrait serialization fidelity. Currently limited to a subset of tests.

### 2.4 Snapshot Testing (Insta)

DataFusion uses the `insta` crate for snapshot testing, primarily in:
- `datafusion/sql/tests/cases/plan_to_sql.rs` -- SQL unparser output
- `datafusion/sql/tests/cases/diagnostic.rs` -- Diagnostic messages
- `datafusion/substrait/tests/cases/` -- All Substrait test cases (7+ files)
- `datafusion/physical-plan/src/` -- Sort, partial sort, unnest, window operators
- `datafusion/pruning/` -- Pruning predicate output

Snapshots are auto-generated files reviewed via `cargo insta review`.

### 2.5 Benchmark Infrastructure

#### Criterion Micro-Benchmarks (`datafusion/core/benches/`)

28 benchmark files covering:
- SQL planning: `sql_planner.rs`, `sql_planner_extended.rs`
- Query execution: `aggregate_query_sql.rs`, `filter_query_sql.rs`, `sort_limit_query_sql.rs`
- Parquet: `parquet_query_sql.rs`, `parquet_struct_query.rs`, `parquet_struct_projection.rs`
- Physical plan operations: `physical_plan.rs`, `sort.rs`, `spm.rs`
- Data loading: `csv_load.rs`, `preserve_file_partitioning.rs`
- Expressions: `scalar.rs`, `math_query_sql.rs`, `map_query_sql.rs`
- Window functions: `window_query_sql.rs`
- Special: `topk_aggregate.rs`, `topk_repartition.rs`, `reset_plan_states.rs`

Usage: `cargo bench --bench sql_planner`
Baseline comparison: `cargo bench --bench sql_planner -- --save-baseline main`

#### Full Benchmark Suite (`benchmarks/` crate)

A standalone crate with binaries for end-to-end benchmark workloads:
- **TPC-H**: 22 standard queries (`queries/q1.sql` through `queries/q22.sql`)
- **TPC-DS**: Full TPC-DS suite (`queries/tpcds/`)
- **ClickBench**: ClickHouse benchmark suite (`queries/clickbench/`)
- **H2O**: H2O.ai benchmark (`src/h2o.rs`)
- **IMDB**: JOB (Join Order Benchmark) on IMDB data (`src/imdb/`, `queries/imdb/`)

Benchmark binaries:
- `dfbench` -- Main benchmark runner
- `external_aggr` -- External aggregation benchmark
- `imdb` -- IMDB/JOB benchmark
- `mem_profile` -- Memory profiling

Additional utilities: `bench.sh`, `compare_tpch.sh`, `compare_tpcds.sh`, `compare.py`

CI verifies benchmark query correctness (not performance) in `verify-benchmark-results`.

### 2.6 CI Infrastructure

#### Primary Workflow: `rust.yml`

Runs on every PR (paths-ignore: docs, markdown). Jobs (all need `linux-build-lib` first):

| Job | Purpose | Key Flags |
|-----|---------|-----------|
| `linux-build-lib` | Cargo check (validates lock file with `--locked`) | `--features integration-tests` |
| `linux-datafusion-common-features` | Check datafusion-common compiles with/without features | Default, no-default |
| `linux-datafusion-substrait-features` | Check substrait feature combinations | Default, no-default, physical, protoc |
| `linux-datafusion-proto-features` | Check proto feature combinations | Default, no-default, json, parquet, avro |
| `linux-cargo-check-datafusion` | Check all 18+ individual feature flags compile | Each flag individually |
| `linux-cargo-check-datafusion-functions` | Check function crate features | 8 feature flags |
| **`linux-test`** | **Main test suite** | `--workspace --lib --tests --bins --features serde,avro,json,backtrace,integration-tests,parquet_encryption,substrait` |
| `linux-test-datafusion-cli` | CLI tests | `--features backtrace` |
| `linux-test-example` | Run example programs | Runs `ci/scripts/rust_example.sh` |
| `linux-test-doc` | Doc tests | `--doc --features avro,json` |
| `linux-rustdoc` | Verify rustdoc clean | `ci/scripts/rust_docs.sh` |
| `linux-wasm-pack` | WASM compilation + headless browser tests | Firefox + Chrome |
| `verify-benchmark-results` | Run TPC-H queries, verify correct results | Generates TPC-H data, runs sqllogictests with `INCLUDE_TPCH=true` |
| `sqllogictest-postgres` | Run slt against Postgres | `--features postgres`, Postgres 15 service container |
| `sqllogictest-substrait` | Substrait round-trip on limited files | `--substrait-round-trip limit.slt` |

#### Extended Workflow: `extended.yml`

Runs on push to `main` (and PRs touching physical/expr/optimizer/sql paths):

| Job | Purpose | Key Flags |
|-----|---------|-----------|
| `linux-test-extended` | Full test suite + extended tests | `--features avro,json,backtrace,extended_tests,recursive_protection,parquet_encryption` |
| `hash-collisions` | All tests with forced hash collisions | `--features force_hash_collisions,avro` |
| `sqllogictest-sqlite` | Run against SQLite's test suite (5M+ queries) | `--include-sqlite`, `--profile ci-optimized` |

#### Dev Workflow: `dev.yml`

Non-test quality checks on every PR:
- License header check (HawkEye)
- Prettier formatting for docs
- `.asf.yaml` validation
- Typo checking (typos-cli)

#### Other Workflows

- `audit.yml` -- Security audit
- `codeql.yml` -- CodeQL analysis
- `dependencies.yml` -- Dependency checks
- `large_files.yml` -- Large file detection
- `labeler.yml` -- Auto-labeling
- `stale.yml` -- Stale issue management
- `docs.yaml` / `docs_pr.yaml` -- Documentation build

### 2.7 Build Profiles

Four custom Cargo profiles optimize for different scenarios:

| Profile | Base | Purpose | Key Settings |
|---------|------|---------|-------------|
| `ci` | `dev` | PR testing (fast compile, no debug info) | `debug = false`, `incremental = false`, deps: `debug-assertions = false`, `strip = "debuginfo"` |
| `ci-optimized` | `release` | Extended tests (sqlite suite needs speed) | `debug-assertions = true`, `lto = "thin"`, `codegen-units = 16` |
| `release-nonlto` | `release` | Benchmarking with flamegraphs | `lto = false`, `strip = false`, `codegen-units = 16` |
| `profiling` | `release` | Performance profiling | `debug = true`, `strip = false` |

### 2.8 Feature Flags as Test Gates

DataFusion uses feature flags to gate test categories:

| Feature Flag | Purpose | Where Used |
|-------------|---------|-----------|
| `extended_tests` | Enable fuzz tests (slow, randomized) | `datafusion/core` -- gates `fuzz_cases/` module |
| `force_hash_collisions` | Force all hash values to collide | `physical-plan`, `common` -- tests correctness under worst-case hashing |
| `integration-tests` | Enable integration tests | `core` -- used in CI `cargo check` |
| `avro` / `json` / `parquet` | Enable format-specific code paths | Throughout -- gates both code and tests |
| `parquet_encryption` | Enable parquet encryption | `core`, `datasource-parquet` |
| `postgres` | Enable Postgres compatibility runner | `sqllogictest` -- gates Postgres engine + testcontainers |
| `substrait` | Enable Substrait round-trip tests | `sqllogictest` -- gates Substrait engine |
| `recursive_protection` | Enable stack overflow protection | Multiple crates |

### 2.9 Shared Test Utilities

#### `test-utils` Crate

A dedicated workspace member providing test helpers:

```rust
// Key exports:
pub fn batches_to_vec(batches: &[RecordBatch]) -> Vec<Option<i32>>
pub fn partitions_to_sorted_vec(partitions: &[Vec<RecordBatch>]) -> Vec<Option<i32>>
pub fn add_empty_batches(batches: Vec<RecordBatch>, rng: &mut StdRng) -> Vec<RecordBatch>
pub fn stagger_batch(batch: RecordBatch) -> Vec<RecordBatch>
pub fn stagger_batch_with_seed(batch: RecordBatch, seed: u64) -> Vec<RecordBatch>
pub struct TableDef { name: String, schema: Schema }
```

Sub-modules:
- `array_gen` -- Random array generation
- `data_gen` -- `AccessLogGenerator` for realistic test data
- `string_gen` -- `StringBatchGenerator` for string test data
- `tpcds` -- TPC-DS table definitions
- `tpch` -- TPC-H table definitions

Dependencies: `arrow`, `datafusion-common`, `rand`, `chrono-tz`, `env_logger`

#### `datafusion-common::test_util` Module

Core assertion macros used across the entire codebase:

- **`assert_batches_eq!(expected, batches)`** -- Compare pretty-printed RecordBatch output with expected string lines (exact order)
- **`assert_batches_sorted_eq!(expected, batches)`** -- Same but sorts non-header rows (for unordered results)
- **`assert_contains!(actual, expected)`** -- Assert string containment
- **`assert_not_contains!(actual, unexpected)`** -- Assert string non-containment
- **`format_batches(results)`** -- Format RecordBatch for comparison
- **`batches_to_string(batches)`** / **`batches_to_sort_string(batches)`** -- String conversion helpers

#### `datafusion/physical-plan/src/test.rs` Module

A **public** (not `#[cfg(test)]`) module providing physical plan test infrastructure:
- `MockExec` -- Configurable mock execution plan
- `TestStream` -- Test record batch stream
- Schema/batch construction helpers
- Equivalence property testing utilities

---

## 3. Key Design Insights

### 3.1 sqllogictest as the Primary Testing Strategy

DataFusion has made a deliberate architectural decision to favor sqllogictest over
Rust integration tests for SQL behavior testing. The reasoning:

1. **No recompile cycle**: Changing a `.slt` file requires zero recompilation. For a
   project with 68 crates, this saves enormous iteration time.
2. **Auto-completion**: The `--complete` flag auto-generates expected output, eliminating
   manual result construction.
3. **Parallel by default**: Each `.slt` file gets its own `SessionContext` and runs
   independently, enabling full parallelism.
4. **Cross-engine verification**: The same `.slt` files run against Postgres (compatibility)
   and through Substrait round-trips (serialization fidelity).

Trade-off: Higher barrier for new contributors unfamiliar with the format, but
dramatically lower maintenance cost over time.

### 3.2 Feature Flags as Test Stratification

The feature flag system serves double duty as both a build configuration mechanism
and a test stratification strategy. The `extended_tests` flag gates expensive fuzz
tests; `force_hash_collisions` enables worst-case correctness checks. This means
the default `cargo test` is fast (PR-blocking), while comprehensive tests run
post-merge.

### 3.3 Single Integration Test Binary Pattern

The `core_integration.rs` pattern of including all test modules via `mod` statements
into a single test binary is intentional: it avoids the Rust linker creating
separate binaries per test file, which would multiply link times. This is a
common pattern in large Rust projects.

### 3.4 Deterministic Test Isolation

Each sqllogictest file gets its own `SessionContext` with hardcoded
`target_partitions = 4`. Scratch directories are per-file. This ensures tests
are deterministic and parallelizable without interference.

### 3.5 Multi-Layer Correctness Verification

DataFusion verifies correctness at multiple layers simultaneously:
- **Unit tests**: Individual operator correctness
- **sqllogictest**: End-to-end SQL correctness
- **Postgres compatibility**: Cross-engine SQL semantics validation
- **Fuzz tests**: Randomized input correctness
- **Hash collision tests**: Worst-case data distribution correctness
- **SQLite test suite**: 5M+ queries from an independent test corpus
- **Benchmark verification**: TPC-H query result correctness (not just performance)

### 3.6 Snapshot Testing for Plan Representations

The `insta` crate is used strategically for testing plan representations
(SQL unparser, Substrait serialization, physical plan display) where the
output is complex structured text. This avoids brittle hand-written assertions
while providing clear diffs when output changes.

---

## 4. Porting Considerations (DataFusion Patterns -> Trino Rust Worker)

### 4.1 Adopt sqllogictest Early

**Recommendation**: Implement a sqllogictest runner as one of the first testing
infrastructure investments. DataFusion's experience shows this pays for itself
quickly in a SQL engine project.

**Adaptation needed**: The Trino worker is not a standalone SQL engine -- it
receives physical plan fragments from a coordinator. The sqllogictest approach
would need to be adapted:
- For unit-level operator testing, a sqllogictest-like framework could drive
  queries through a local planning path
- For integration testing, the runner would need to serialize plan fragments
  and feed them to the worker execution engine
- Consider a custom `.slt`-like format for plan-fragment-level testing

### 4.2 Feature-Flag-Gated Test Tiers

**Recommendation**: Replicate the `extended_tests` pattern. Define a fast default
test suite (< 5 minutes) that blocks PRs, and an extended suite (fuzz, large data,
compatibility) that runs post-merge or nightly.

Key feature flags to consider:
- `extended_tests` -- Fuzz tests, large-scale correctness
- `integration_tests` -- Tests requiring a Trino coordinator
- `force_hash_collisions` -- Worst-case hash table behavior

### 4.3 Shared Test Utility Crate

**Recommendation**: Create a `test-utils` crate from day one. DataFusion's
`test-utils` + `datafusion-common::test_util` pattern works well:
- Workspace-level crate for data generation (`RecordBatch` builders, random data, TPC-H/TPC-DS schemas)
- Core assertion macros (`assert_batches_eq!`, `assert_contains!`) in the common crate
- Physical plan mock infrastructure (equivalent to `physical-plan/src/test.rs`) as a public module

### 4.4 Single Binary Integration Tests

**Recommendation**: Use the `core_integration.rs` hub pattern for integration tests.
In a large workspace, having one `mod` file that includes all test modules into a
single binary avoids the N-binary link time explosion.

### 4.5 CI Profile Strategy

**Recommendation**: Define custom Cargo profiles:
- `ci` (inherits `dev`): `debug = false`, `incremental = false` -- fast compilation for PR checks
- `ci-optimized` (inherits `release`): `lto = "thin"` -- for benchmarks and extended tests
- `profiling` (inherits `release`): `debug = true` -- for flamegraphs

### 4.6 Fuzz Testing for Operator Correctness

**Recommendation**: Implement fuzz tests for critical operators early. DataFusion's
fuzz test coverage is instructive -- they fuzz:
- Join operators (hash join, sort-merge join)
- Sort operators (sort, partial sort, sort-preserving merge)
- Aggregation operators
- Window functions
- Limit operators
- Spilling behavior under memory pressure

For the Trino worker, priority fuzz targets would be:
- Exchange/shuffle operators (the worker's primary differentiator)
- Join operators with various input distributions
- Aggregation with spilling
- Memory limit enforcement

### 4.7 Postgres/Trino Compatibility Testing

DataFusion's `pg_compat_` pattern is directly relevant. For a Trino-compatible
worker, run the same queries against both the Rust worker and a real Trino
instance to verify behavioral compatibility. The sqllogictest infrastructure
supports this with pluggable engine backends.

### 4.8 Benchmark Verification (Correctness, Not Just Speed)

DataFusion's CI verifies that benchmark queries produce correct results without
measuring performance. This is a low-cost way to ensure benchmark code stays
functional. Adopt this for TPC-H/TPC-DS benchmarks from the start.

### 4.9 Snapshot Testing for Plan Representations

**Recommendation**: Use `insta` for testing:
- Physical plan display/explain output
- Serialized plan representations (protobuf, JSON)
- Error message formatting

This is more maintainable than hand-written string assertions for complex
structured output.

### 4.10 Test Data Management

DataFusion stores small fixture files directly in `tests/data/` (59+ files: CSV,
Parquet, JSON). Larger datasets (TPC-H, SQLite test suite) are generated or
fetched at test time. For the Trino worker:
- Small fixtures: commit to repo
- TPC-H/TPC-DS data: generate in CI (DataFusion's pattern with `tpch-dbgen`)
- Trino-specific test data: consider a `datafusion-testing`-style submodule
