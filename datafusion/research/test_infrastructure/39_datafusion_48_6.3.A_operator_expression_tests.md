# Module Teardown: Operator & Expression Testing Patterns (Task 6.3.A) [KG-13]

## 0. Research Focus

This analysis examines how DataFusion tests physical operators and expressions in
isolation, with attention to:

- How operators are constructed and executed with synthetic RecordBatch input
- Assertion patterns used to verify correctness (assert_batches_eq, insta snapshots)
- Test utility infrastructure (TestMemoryExec, MockExec, helper macros)
- How stateful operators (joins, aggregations, sorts) test spill paths
- Parameterized testing across multiple data types

The goal is to extract patterns portable to a Rust-based Trino worker test harness.

Source: DataFusion commit at `datafusion/datafusion/` (v48.x timeframe).


## 1. High-Level Overview

DataFusion's test infrastructure follows a layered approach:

```
Layer 4: sqllogictest (SQL-level integration)       -- *.slt files
Layer 3: Core integration tests                     -- datafusion/core/tests/
Layer 2: Operator unit tests (#[cfg(test)] modules) -- physical-plan/src/
Layer 1: Expression unit tests                      -- physical-expr/src/
Layer 0: Test utilities & assertion macros           -- common/src/test_util.rs, physical-plan/src/test.rs
```

For operator and expression testing (Layers 0-2), the canonical pattern is:

1. Build synthetic data as `RecordBatch` using helper functions
2. Construct the operator/expression under test with its dependencies
3. Execute via `execute(partition, task_ctx)` -> `common::collect(stream)` (operators)
   or `evaluate(&batch)` (expressions)
4. Assert results using `assert_batches_eq!`, `assert_snapshot!`, or direct array comparison


## 2. Detailed Analysis

### 2.1 Test Utility Infrastructure

#### 2.1.A Core Assertion Macros (`datafusion/common/src/test_util.rs`)

Four primary assertion macros are defined as `#[macro_export]`:

| Macro | Purpose | Order-sensitive? |
|-------|---------|-----------------|
| `assert_batches_eq!` | Compares pretty-printed batch output against expected string lines | Yes |
| `assert_batches_sorted_eq!` | Same, but sorts data rows before comparison (preserves header/footer) | No |
| `assert_contains!` | Checks that one string contains another | N/A |
| `assert_not_contains!` | Checks that one string does NOT contain another | N/A |

Implementation detail: `assert_batches_sorted_eq!` sorts lines `[2..num_lines-1]`
(i.e., skips the table header and footer borders), so it handles non-deterministic
row ordering from hash-based or partitioned operators.

Helper functions:
- `batches_to_string(&[RecordBatch])` -- pretty-formats and trims
- `batches_to_sort_string(&[RecordBatch])` -- pretty-formats, sorts data rows, joins

**Record batch construction macro** (`record_batch!`):
```rust
let batch = record_batch!(
    ("a", Int32, vec![1, 2, 3]),
    ("b", Float64, vec![Some(4.0), None, Some(5.0)]),
    ("c", Utf8, vec!["alpha", "beta", "gamma"])
);
```
The `create_array!` macro dispatches on type name (Boolean, Int8..Float64, Utf8) to
create typed Arrow arrays. The `IntoArrayRef` trait provides `into_array_ref()` for
common Rust types (Vec<i32>, Vec<Option<f64>>, Vec<&str>, etc.).

#### 2.1.B Mock ExecutionPlan Types (`datafusion/physical-plan/src/test.rs` and `test/exec.rs`)

| Type | File | Purpose |
|------|------|---------|
| `TestMemoryExec` | `test.rs` | In-memory data source with partitions, projections, sort information. Primary mock for feeding data to operators. |
| `MockExec` | `test/exec.rs` | Wraps `Vec<Result<RecordBatch>>` -- can inject errors. Sends batches via separate tokio task by default to test poll loops. |
| `BarrierExec` | `test/exec.rs` | Uses `tokio::sync::Barrier` so all stream partitions start simultaneously. Tests concurrent execution scenarios. |
| `BlockingExec` | `test/exec.rs` | Creates an operator that blocks forever -- used with `assert_is_pending` and `assert_strong_count_converges_to_zero` for cancellation/cleanup tests. |
| `TestStream` | `test/exec.rs` | Simple `Stream` impl over `Vec<RecordBatch>` with a `BatchIndex` counter for tracking consumption. |
| `TestPartitionStream` | `test.rs` | Implements `PartitionStream` trait for streaming table tests. |
| `SortedUnboundedExec` | `sorts/sort.rs` (test module) | Custom unbounded exec that produces monotonically increasing UInt64 values. Tests sort behavior on infinite streams. |

Key helper functions in `test.rs`:

```
build_table_i32(a, b, c) -> RecordBatch          // 3-column i32 batch
build_table_i32_two_cols(a, b) -> RecordBatch     // 2-column i32 batch
build_table_scan_i32(a, b, c) -> Arc<dyn ExecutionPlan>  // wrapped in TestMemoryExec
make_partition(sz: i32) -> RecordBatch            // single column "i" with 0..sz
scan_partitioned(n: usize) -> Arc<dyn ExecutionPlan>  // n partitions of 100 rows each
assert_is_pending(fut)                             // asserts a future is Poll::Pending
```

`assert_join_metrics!` macro (test.rs, line 534): verifies output_rows, elapsed_compute,
join_time, and build_time metrics after join execution.

#### 2.1.C Snapshot Testing with Insta

DataFusion uses `insta::assert_snapshot!` extensively alongside `batches_to_string` and
`batches_to_sort_string`. The `allow_duplicates!` wrapper is used when the same snapshot
appears in multiple parameterized test expansions.

```rust
assert_snapshot!(batches_to_string(&batches), @r"
    +----+----+----+
    | a1 | b1 | c1 |
    +----+----+----+
    | 1  | 4  | 7  |
    +----+----+----+
");
```

### 2.2 Operator Testing Patterns

#### 2.2.A Join Operator Tests

**Location**: `physical-plan/src/joins/`

Each join type has its own test module. The sort merge join tests are the most
extensive, in a dedicated `tests.rs` file (4000+ lines).

**Test structure for sort merge join** (`joins/sort_merge_join/tests.rs`):

The file is organized with local helper functions that layer construction:

```rust
// Step 1: Build synthetic data
fn build_table(a, b, c) -> Arc<dyn ExecutionPlan>  // wraps build_table_i32 in TestMemoryExec

// Step 2: Construct operator
fn join(left, right, on, join_type) -> Result<SortMergeJoinExec>
fn join_with_filter(left, right, on, filter, join_type, ...) -> Result<SortMergeJoinExec>
fn join_with_options(left, right, on, join_type, sort_options, null_equality) -> Result<SortMergeJoinExec>

// Step 3: Execute and collect
async fn join_collect(left, right, on, join_type) -> Result<(Vec<String>, Vec<RecordBatch>)>
    // internally: join.execute(0, task_ctx)? -> common::collect(stream).await

// Step 4: Assert
assert_snapshot!(batches_to_string(&batches), @r"...");
// or: assert_snapshot!(batches_to_sort_string(&batches), @r"...");
```

Concrete example (`join_inner_one`, line 357):
```rust
#[tokio::test]
async fn join_inner_one() -> Result<()> {
    let left = build_table(("a1", &vec![1, 2, 3]), ("b1", &vec![4, 5, 5]), ("c1", &vec![7, 8, 9]));
    let right = build_table(("a2", &vec![10, 20, 30]), ("b1", &vec![4, 5, 6]), ("c2", &vec![70, 80, 90]));
    let on = vec![(
        Arc::new(Column::new_with_schema("b1", &left.schema())?) as _,
        Arc::new(Column::new_with_schema("b1", &right.schema())?) as _,
    )];
    let (_, batches) = join_collect(left, right, on, Inner).await?;
    assert_snapshot!(batches_to_string(&batches), @r"...");
    Ok(())
}
```

**Data type coverage**: The SMJ tests cover Int32, Date32, Date64, Binary,
FixedSizeBinary types via separate `build_date_table`, `build_binary_table`, etc.
Nullable variants tested via `build_table_i32_nullable`.

**Join type coverage**: Each join type (Inner, Left, Right, Full, LeftSemi, LeftAnti,
LeftMark) has dedicated tests with various data patterns.

**Hash join tests** (`joins/hash_join/exec.rs`, test module at line 2094):

Uses `rstest` for parameterized testing:
```rust
#[template]
#[rstest]
fn hash_join_exec_configs(
    #[values(8192, 10, 5, 2, 1)] batch_size: usize,
    #[values(true, false)] use_perfect_hash_join_as_possible: bool,
) {}

#[apply(hash_join_exec_configs)]
#[tokio::test]
async fn join_inner_one(batch_size: usize, use_perfect_hash_join_as_possible: bool) -> Result<()> {
    // ... each test runs with 5 * 2 = 10 combinations
}
```

This tests correctness across batch sizes from 1 to 8192 and both hash join
implementations (standard vs perfect hash join).

**Join test utilities** (`joins/test_utils.rs`):

- `compare_batches()` -- sorts lines and compares two result sets
- `partitioned_hash_join_with_filter()` -- sets up 4-partition hash join execution
- `partitioned_sym_join_with_filter()` -- sets up 4-partition symmetric hash join
- `split_record_batches()` -- splits a batch into smaller chunks for multi-batch tests
- `build_sides_record_batches()` -- generates complex test data with ordered, random,
  null-first/last, timestamp, interval, and float columns
- `create_memory_table()` -- wraps left/right partitions in TestMemoryExec with sort info
- `complicated_filter()` -- builds `(a + b > c + 10) AND (a + b < c + 100)` filter expression
- `join_expr_tests!` macro -- generates filter expression constructors for i32 and f64 types

#### 2.2.B Aggregation Operator Tests

**Location**: `physical-plan/src/aggregates/mod.rs` (test module at line 2004)

Test structure follows the two-phase aggregation model:

```rust
// Phase 1: Partial aggregation
let partial_aggregate = Arc::new(AggregateExec::try_new(
    AggregateMode::Partial,
    grouping_set.clone(),
    aggregates.clone(),
    vec![None],
    input,
    Arc::clone(&input_schema),
)?);
let result = collect(partial_aggregate.execute(0, task_ctx)?).await?;

// Phase 2: Final aggregation
let merge = Arc::new(CoalescePartitionsExec::new(partial_aggregate));
let merged_aggregate = Arc::new(AggregateExec::try_new(
    AggregateMode::Final, ...
)?);
let result = collect(merged_aggregate.execute(0, task_ctx)?).await?;
```

Key test functions:
- `check_aggregates(input, spill: bool)` -- tests AVG with partial->final pipeline
- `check_grouping_sets(input, spill: bool)` -- tests grouping sets with COUNT(1)
- `new_spill_ctx(batch_size, max_memory)` -- creates TaskContext with FairSpillPool
- `run_test_with_spill_pool_if_necessary(pool_size, expect_spill)` -- tests MIN/AVG
  aggregation with configurable memory pool, verifying identical results with/without spill
- `assert_spill_count_metric(expect_spill, aggregate_exec)` -- inspects MetricValue::SpillCount

**Row hash aggregation tests** (`aggregates/row_hash.rs`, line 1356):
Tests race conditions, e.g. `test_double_emission_race_condition_bug()` sets up:
- Memory limit of 1024 bytes
- batch_size=1024 with 1124 groups (intentionally exceeds batch_size)
- Skip aggregation thresholds to trigger both early-emit and skip-aggregation paths
  simultaneously, verifying no data loss.

**TopK aggregation tests** (`aggregates/topk/`):
- `topk/heap.rs` (line 648) -- unit tests for the binary heap data structure
- `topk/hash_table.rs` (line 425) -- unit tests for the hash table
- `topk/priority_map.rs` (line 94) -- unit tests for the combined priority map

#### 2.2.C Sort Operator Tests

**Location**: `physical-plan/src/sorts/sort.rs` (test module at line 1429)

**In-memory sort** (`test_in_mem_sort`, line 1585):
```rust
let partitions = 4;
let csv = test::scan_partitioned(partitions);
let sort_exec = Arc::new(SortExec::new(
    [PhysicalSortExpr { expr: col("i", &schema)?, options: SortOptions::default() }].into(),
    Arc::new(CoalescePartitionsExec::new(csv)),
));
let result = collect(sort_exec, task_ctx).await?;
assert_eq!(result.len(), 1);
assert_eq!(result[0].num_rows(), 400);
assert_eq!(task_ctx.runtime_env().memory_pool.reserved(), 0,
    "The sort should have returned all memory used back to the memory manager");
```

**Sort with spill** (`test_sort_spill`, line 1614):
```rust
let runtime = RuntimeEnvBuilder::new()
    .with_memory_limit(sort_spill_reservation_bytes + 12288, 1.0)
    .build_arc()?;
// ... 100 partitions of 100 rows = 40000 bytes with ~12KB memory limit
let result = collect(sort_exec, task_ctx).await?;
// Verify spill metrics
let metrics = sort_exec.metrics().unwrap();
assert!((3..=10).contains(&metrics.spill_count().unwrap()));
assert!((9000..=10000).contains(&metrics.spilled_rows().unwrap()));
assert!((38000..=44000).contains(&metrics.spilled_bytes().unwrap()));
```

**Memory reservation failure** (`test_batch_reservation_error`, line 1685):
Sets memory limit just below what the first batch needs, verifies graceful error.

**Sort preserving merge** (`sorts/sort_preserving_merge.rs`, test module at line 431):
- Tests memory limits and tie-breaker logic for k-way merge
- Uses `RuntimeEnvBuilder::new().with_memory_limit(20_000_000, 1.0)` to test OOM
- `BlockingExec` + `assert_is_pending` + `assert_strong_count_converges_to_zero` tests
  verify proper cleanup when the consuming future is dropped

#### 2.2.D Spill Path Testing Pattern (Sort Merge Join)

**Location**: `joins/sort_merge_join/tests.rs`, lines 2053-2614

The spill tests follow a systematic matrix:

| Test Name | Batches | Spill Enabled? | Purpose |
|-----------|---------|---------------|---------|
| `overallocation_single_batch_no_spill` | 1 | No (DiskManagerMode::Disabled) | Verify OOM error with disabled spill |
| `overallocation_multi_batch_no_spill` | 3+3 | No | Same with multi-batch input |
| `overallocation_single_batch_spill` | 1 | Yes (OsTmpDirectory) | Verify correct results with spilling |
| `overallocation_multi_batch_spill` | 3+3 | Yes | Same with multi-batch input |
| `spill_with_filter_deferred` | - | Yes | Spill with deferred filter evaluation |
| `spill_with_filter_multi_batch` | - | Yes | Spill with filter across multiple batches |
| `spill_full_join_filter_not_matched` | - | Yes | Full join unmatched rows survive spill |
| `spill_filtered_boundary_loses_outer_rows` | - | Yes | Edge case: outer rows at spill boundary |

Pattern for testing spill disabled:
```rust
let runtime = RuntimeEnvBuilder::new()
    .with_memory_limit(100, 1.0)
    .with_disk_manager_builder(
        DiskManagerBuilder::default().with_mode(DiskManagerMode::Disabled),
    )
    .build_arc()?;
// ... execute and expect error
let err = common::collect(stream).await.unwrap_err();
assert_contains!(err.to_string(), "Failed to allocate additional");
assert_contains!(err.to_string(), "Disk spilling disabled");
assert_eq!(join.metrics().unwrap().spill_count(), Some(0));
```

Pattern for testing spill enabled:
```rust
let runtime = RuntimeEnvBuilder::new()
    .with_memory_limit(100, 1.0)
    .with_disk_manager_builder(
        DiskManagerBuilder::default().with_mode(DiskManagerMode::OsTmpDirectory),
    )
    .build_arc()?;
// ... iterate over batch_sizes [1, 50] and join_types [Inner, Left, Right, Full]
let spilled_join_result = common::collect(stream).await.unwrap();
assert!(join.metrics().unwrap().spill_count().unwrap() > 0);
assert!(join.metrics().unwrap().spilled_bytes().unwrap() > 0);
// Compare spill result against known-good no-spill result
```

### 2.3 Expression Testing Patterns

#### 2.3.A Binary Expression Tests (`physical-expr/src/expressions/binary.rs`, line 997)

Tests follow the evaluate-and-compare pattern:
```rust
#[test]
fn binary_comparison() -> Result<()> {
    let schema = Schema::new(vec![
        Field::new("a", DataType::Int32, false),
        Field::new("b", DataType::Int32, false),
    ]);
    let a = Int32Array::from(vec![1, 2, 3, 4, 5]);
    let b = Int32Array::from(vec![1, 2, 4, 8, 16]);
    let lt = binary(col("a", &schema)?, Operator::Lt, col("b", &schema)?, &schema)?;
    let batch = RecordBatch::try_new(Arc::new(schema), vec![Arc::new(a), Arc::new(b)])?;
    let result = lt.evaluate(&batch)?.into_array(batch.num_rows())?;
    let result = as_boolean_array(&result)?;
    // Direct element-wise assertion
    for (i, &expected_item) in [false, false, true, true, true].iter().enumerate() {
        assert_eq!(result.value(i), expected_item);
    }
    Ok(())
}
```

The `test_coercion!` macro (line 1116) is used for cross-type binary operations:
```rust
macro_rules! test_coercion {
    ($A_ARRAY:ident, $A_TYPE:expr, $A_VEC:expr,
     $B_ARRAY:ident, $B_TYPE:expr, $B_VEC:expr,
     $OP:expr,
     $C_ARRAY:ident, $C_TYPE:expr, $VEC:expr,) => {{
        // 1. Create schema with (a: A_TYPE, b: B_TYPE)
        // 2. Apply type coercion
        // 3. Build binary expression
        // 4. Evaluate
        // 5. Assert result type and values
    }};
}
```

#### 2.3.B Cast Expression Tests (`physical-expr/src/expressions/cast.rs`, line 352)

Two macros for thorough type cast testing:
- `generic_test_cast!` -- casts A_TYPE to B_TYPE, verifies type, length, and values
- `generic_decimal_to_other_test_cast!` -- specialized for Decimal128 source

Each macro follows 5 verification steps:
1. Construct schema and batch
2. Build CAST expression
3. Verify display format (e.g., "CAST(a@0 AS Int32)")
4. Verify output data type
5. Compare result values element-by-element, handling None (null) cases

#### 2.3.C CASE Expression Tests (`physical-expr/src/expressions/case.rs`, line 1422)

Tests cover: CASE with expression match, dictionary-encoded inputs, CASE with boolean
WHEN conditions, ELSE clauses, all-null scenarios, and type coercion edge cases.

#### 2.3.D IN LIST Multi-Type Tests (`physical-expr/src/expressions/in_list.rs`, line 940)

This is the best example of parameterized expression testing across types.

Architecture:
```rust
struct InListPrimitiveTestCase {
    name: &'static str,
    value_in: ScalarValue,        // value that matches
    value_not_in: ScalarValue,    // value that doesn't match
    other_list_values: Vec<ScalarValue>,
    null_value: Option<ScalarValue>,  // None = skip null tests
}

struct PrimitiveTestCaseData<T> {
    value_in: T,
    value_not_in: T,
    other_list_values: Vec<T>,
}
```

Helper functions create test cases with generic data:
- `primitive_test_case(name, constructor, data)` -- with null support
- `primitive_test_case_no_nulls(name, constructor, data)` -- without null support

The `run_test_cases()` function executes 2-4 sub-tests per type:
1. `a IN (list)` -> `[true, false, null]`
2. `a NOT IN (list)` -> `[false, true, null]`
3. `a IN (list, NULL)` -> `[true, null, null]` (only with null_value)
4. `a NOT IN (list, NULL)` -> `[false, null, null]` (only with null_value)

Type coverage in single test functions:
```rust
#[test]
fn in_list_int_types() -> Result<()> {
    let int_data = PrimitiveTestCaseData { value_in: 0, value_not_in: 2, other_list_values: vec![1, 3, 5] };
    run_test_cases(vec![
        primitive_test_case("int8", ScalarValue::Int8, int_data.clone()),
        primitive_test_case("int16", ScalarValue::Int16, int_data.clone()),
        primitive_test_case("int32", ScalarValue::Int32, int_data.clone()),
        primitive_test_case("int64", ScalarValue::Int64, int_data.clone()),
        primitive_test_case("uint8", ScalarValue::UInt8, int_data.clone()),
        primitive_test_case("uint16", ScalarValue::UInt16, int_data.clone()),
        primitive_test_case("uint32", ScalarValue::UInt32, int_data.clone()),
        primitive_test_case("uint64", ScalarValue::UInt64, int_data.clone()),
        primitive_test_case_no_nulls("int32_no_nulls", ScalarValue::Int32, int_data),
    ])
}
```

Similar tests exist for: string types (Utf8, LargeUtf8, Utf8View), binary types
(Binary, LargeBinary, BinaryView), date types (Date32, Date64), decimal (Decimal128),
timestamp types (nanosecond, millisecond with timezone), interval types, and boolean.

### 2.4 Cancellation and Cleanup Testing

Several operators use `BlockingExec` to test proper resource cleanup:

```rust
// Pattern from sort_preserving_merge.rs, coalesce_partitions.rs, analyze.rs, etc.
#[tokio::test]
async fn test_drop_cancel() -> Result<()> {
    let blocking_exec = Arc::new(BlockingExec::new(schema, 1));
    let refs = Arc::downgrade(&blocking_exec);
    let sort_exec = Arc::new(SortPreservingMergeExec::new(sort, blocking_exec));

    let mut fut = Box::pin(common::collect(sort_exec.execute(0, task_ctx)?));

    assert_is_pending(&mut fut);      // operator is blocked waiting for input
    drop(fut);                          // simulate consumer cancellation
    assert_strong_count_converges_to_zero(refs).await;  // all Arcs freed
    Ok(())
}
```

`assert_strong_count_converges_to_zero` polls up to 2 seconds checking `Weak::strong_count()`,
verifying no resource leaks after cancellation.


## 3. Key Design Insights

1. **TestMemoryExec is the universal data source mock.** Every operator test ultimately
   feeds from `TestMemoryExec` (or a helper that wraps it). It supports partitions,
   projections, sort information, and fetch limits -- everything needed to test operator
   behavior without touching files or network.

2. **Two assertion paradigms coexist.** Older tests use `assert_batches_eq!` with
   hand-written expected string arrays. Newer tests use `insta::assert_snapshot!` with
   inline snapshots (`@r"..."`), which auto-update on test failure. The insta approach is
   lower-friction and is the current preferred style.

3. **Spill testing is integrated into operator tests, not separated.** Rather than having
   a separate "spill test suite", each stateful operator's test module includes spill
   variants alongside normal tests. The pattern is consistent: configure
   `RuntimeEnvBuilder::with_memory_limit()`, optionally configure `DiskManagerBuilder`,
   then verify either error (disabled spill) or correct results + spill metrics (enabled).

4. **rstest provides the parameterization framework.** The hash join tests demonstrate
   the pattern: a `#[template]` defines parameter combinations, and `#[apply(template)]`
   expands each test. This replaces the need for hand-written test-per-configuration.

5. **Expression tests use macros for cross-type coverage.** Rather than rstest,
   expression tests define custom macros (`test_coercion!`, `generic_test_cast!`,
   `in_list!`) and data-driven test case structures (`InListPrimitiveTestCase`) to
   run identical logic across 10+ types. The `ScalarValue` enum's constructor functions
   serve as type-erased factories.

6. **Cancellation testing is explicit.** `BlockingExec` + `assert_is_pending` +
   `assert_strong_count_converges_to_zero` is a deliberate three-step protocol ensuring
   operators correctly handle `Drop` on their output streams without leaking resources.

7. **Metrics are part of the test contract.** Tests don't just verify output data --
   they check `spill_count`, `spilled_rows`, `spilled_bytes`, `output_rows`,
   `elapsed_compute`, `join_time`, `build_time`, etc. This ensures the instrumentation
   remains accurate as code evolves.

8. **TaskContext is the control plane for test configuration.** Memory limits, batch
   sizes, session config options, and runtime environment are all injected via
   `TaskContext`. Tests build custom contexts with `TaskContext::default()
   .with_session_config(config).with_runtime(runtime)`.


## 4. Porting Considerations (DataFusion Patterns -> Trino Rust Worker)

### 4.1 Adopt the TestMemoryExec Pattern

Create a `TestMemorySource` in the Trino Rust worker that:
- Accepts `Vec<Vec<RecordBatch>>` (partitions of batches)
- Implements the worker's operator/source trait
- Supports configurable sort ordering metadata
- Provides helper constructors: `from_i32_table(a, b, c)`, `from_batches(batches)`,
  `scan_partitioned(n)`, etc.

This is the single most important test infrastructure investment -- every operator test
depends on it.

### 4.2 Use Insta for Snapshot Testing

Insta (`cargo insta`) is well-established in the Rust ecosystem and eliminates the
pain of maintaining hand-written expected output strings. Combined with
`batches_to_string` / `batches_to_sort_string`, it provides readable, auto-updatable
test output.

For order-insensitive operators (hash joins, hash aggregations), always use
`batches_to_sort_string` to avoid flaky tests.

### 4.3 Parameterize Tests with rstest

The hash join pattern (`#[template]` + `#[apply]`) is directly portable. For the
Trino worker, apply it to:
- Batch sizes (1, target, max)
- Join types (Inner, Left, Right, Full, Semi, Anti)
- Null handling modes (NullEqualsNull vs NullEqualsNothing)
- Hash join variant (standard vs perfect hash)

### 4.4 Build a Type-Coverage Test Harness for Expressions

The `InListPrimitiveTestCase` pattern is the right model. Define:
```rust
struct ExpressionTestCase {
    name: &'static str,
    input_scalars: Vec<ScalarValue>,
    expected_output: Vec<Option<ScalarValue>>,
    // ... expression constructor closure
}
```

Then test each expression against all supported Trino types in a single test function.
This catches type-handling regressions efficiently.

### 4.5 Spill Testing Strategy

Spill tests require:
1. **Memory pool with configurable limits** -- DataFusion uses `GreedyMemoryPool` and
   `FairSpillPool`. The Trino worker needs an equivalent `MemoryPool` trait.
2. **Disk manager with enable/disable** -- DataFusion's `DiskManagerBuilder` with
   `DiskManagerMode::Disabled` vs `OsTmpDirectory` is the pattern.
3. **Spill metric verification** -- Always check `spill_count > 0` when spill is
   expected, and `spill_count == 0` when not.
4. **Result equivalence** -- Run the same query with and without spill, compare results.
   DataFusion does this within `overallocation_single_batch_spill` tests.

### 4.6 Cancellation Testing

The `BlockingExec` + `assert_is_pending` + `Weak::strong_count` pattern is essential for
any operator that spawns background tasks or holds resources. Port this as:
1. A `BlockingSource` that never produces data
2. `assert_is_pending()` using `futures::task::noop_waker()`
3. `assert_cleanup_completes()` polling `Weak::strong_count()` to zero

### 4.7 Key File Reference

| File | Role |
|------|------|
| `datafusion/common/src/test_util.rs` | Assertion macros, record_batch!, create_array!, IntoArrayRef |
| `datafusion/physical-plan/src/test.rs` | TestMemoryExec, build_table_i32, make_partition, assert_is_pending, assert_join_metrics! |
| `datafusion/physical-plan/src/test/exec.rs` | MockExec, BarrierExec, BlockingExec, TestStream, assert_strong_count_converges_to_zero |
| `datafusion/physical-plan/src/joins/test_utils.rs` | compare_batches, partitioned_*_join_with_filter, build_sides_record_batches, join_expr_tests! macro |
| `datafusion/physical-plan/src/joins/sort_merge_join/tests.rs` | ~4000 lines covering all join types, filters, null handling, spill paths |
| `datafusion/physical-plan/src/joins/hash_join/exec.rs` | rstest parameterized tests (batch_size x hash_join_variant), spill tests |
| `datafusion/physical-plan/src/aggregates/mod.rs` | check_aggregates, check_grouping_sets (both with spill=true/false), new_spill_ctx |
| `datafusion/physical-plan/src/aggregates/row_hash.rs` | Race condition regression tests |
| `datafusion/physical-plan/src/sorts/sort.rs` | test_in_mem_sort, test_sort_spill, memory reservation tests |
| `datafusion/physical-plan/src/sorts/sort_preserving_merge.rs` | K-way merge tests, memory limit tests, cancellation tests |
| `datafusion/physical-expr/src/expressions/binary.rs` | Binary expression tests, test_coercion! macro |
| `datafusion/physical-expr/src/expressions/cast.rs` | generic_test_cast! and generic_decimal_to_other_test_cast! macros |
| `datafusion/physical-expr/src/expressions/in_list.rs` | Multi-type parameterized IN LIST tests, run_test_cases pattern |
| `datafusion/physical-expr/src/expressions/case.rs` | CASE expression tests with dictionary/null/coercion edge cases |
