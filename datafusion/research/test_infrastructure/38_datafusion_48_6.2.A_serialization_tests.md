# Module Teardown: Arrow Serialization & Round-Trip Tests (Task 6.2.A) [KG-13]

## 0. Research Focus

This document analyzes DataFusion's serialization test infrastructure, covering:
- How Arrow data types, ScalarValues, and schemas are tested for protobuf round-trip fidelity
- How logical and physical execution plans are serialized/deserialized and verified
- How nested/complex Arrow types (List, Struct, Map, Dictionary) are encoded via IPC within protobuf
- Whether property-based or fuzz testing is used for serialization
- What test utilities exist for constructing test RecordBatch instances
- How extension codecs (UDFs, custom plans, FFI) are tested for serialization

**Key source locations:**
- `datafusion/proto/tests/cases/roundtrip_logical_plan.rs` (~3050 lines)
- `datafusion/proto/tests/cases/roundtrip_physical_plan.rs` (~3189 lines)
- `datafusion/proto/tests/cases/serialize.rs` (~299 lines)
- `datafusion/proto/tests/cases/mod.rs` (shared UDF/UDAF/UDWF test types)
- `datafusion/proto/src/bytes/mod.rs` (Serializeable trait + plan byte helpers)
- `datafusion/proto-common/src/to_proto/mod.rs` (IPC encoding for nested ScalarValues)
- `datafusion/proto-common/src/from_proto/mod.rs` (IPC decoding for nested ScalarValues)
- `datafusion/proto-common/proto/datafusion_common.proto` (protobuf schema)
- `datafusion/physical-plan/src/test.rs` (RecordBatch construction utilities)
- `datafusion/physical-plan/src/spill/mod.rs` (IPC StreamWriter round-trip tests)
- `datafusion/ffi/src/proto/logical_extension_codec.rs` (FFI codec round-trip tests)
- `datafusion-examples/examples/proto/` (composed codec + expression deduplication examples)

## 1. High-Level Overview

DataFusion's serialization test infrastructure is organized around a single core pattern: **construct an object, serialize it to protobuf bytes, deserialize it back, and assert equality**. This pattern is applied at every layer of abstraction:

```
Layer 1: Primitive types     DataType, Field, Schema, DFSchema -> protobuf -> assert_eq
Layer 2: Scalar values       ScalarValue (all variants) -> protobuf -> assert_eq
Layer 3: Expressions         Expr (logical) / PhysicalExpr -> protobuf bytes -> assert_eq
Layer 4: Plans               LogicalPlan / ExecutionPlan -> protobuf bytes -> assert_eq(Debug)
Layer 5: Full queries        SQL string -> optimize -> physical plan -> bytes -> compare Debug
```

There are **no property-based or fuzz tests** (no proptest, quickcheck, or Arbitrary usage in the proto crate). All serialization testing is exhaustive enumeration of known types and edge cases.

### Test Organization

```
datafusion/proto/tests/
  proto_integration.rs          -- just `mod cases;`
  cases/
    mod.rs                      -- shared test UDF/UDAF/UDWF definitions
    roundtrip_logical_plan.rs   -- ~80 test functions for logical plan/expr roundtrips
    roundtrip_physical_plan.rs  -- ~60 test functions for physical plan roundtrips
    serialize.rs                -- ~12 test functions for Expr byte serialization
  testdata/
    test.arrow                  -- test data files used by some tests
    test.csv
```

## 2. Detailed Analysis

### 2.1 Core Roundtrip Test Patterns

**Pattern A: Logical Expression Roundtrip** (`roundtrip_logical_plan.rs`)

The central helper function is `roundtrip_expr_test`:

```rust
// File: datafusion/proto/tests/cases/roundtrip_logical_plan.rs:122-141
fn roundtrip_expr_test(initial_struct: Expr, ctx: SessionContext) {
    let extension_codec = DefaultLogicalExtensionCodec {};
    roundtrip_expr_test_with_codec(initial_struct, ctx, &extension_codec);
}

fn roundtrip_expr_test_with_codec(
    initial_struct: Expr,
    ctx: SessionContext,
    codec: &dyn LogicalExtensionCodec,
) {
    let proto: protobuf::LogicalExprNode = serialize_expr(&initial_struct, codec)
        .unwrap_or_else(|e| panic!("Error serializing expression: {e:?}"));
    let round_trip: Expr = from_proto::parse_expr(&proto, &ctx, codec).unwrap();
    assert_eq!(format!("{:?}", &initial_struct), format!("{round_trip:?}"));
    roundtrip_json_test(&proto);  // also JSON roundtrip if "json" feature enabled
}
```

Key observation: **equality is checked via Debug format string comparison**, not via `PartialEq`. This is because some Expr variants contain Arc references or closures that don't implement PartialEq. The JSON roundtrip (when the `json` feature is enabled) uses direct `assert_eq!` on the protobuf struct.

**Pattern B: Physical Plan Roundtrip** (`roundtrip_physical_plan.rs`)

```rust
// File: datafusion/proto/tests/cases/roundtrip_physical_plan.rs:136-173
fn roundtrip_test(exec_plan: Arc<dyn ExecutionPlan>) -> Result<()> {
    let ctx = SessionContext::new();
    let codec = DefaultPhysicalExtensionCodec {};
    let proto_converter = DefaultPhysicalProtoConverter {};
    roundtrip_test_and_return(exec_plan, &ctx, &codec, &proto_converter)?;
    Ok(())
}

fn roundtrip_test_and_return(
    exec_plan: Arc<dyn ExecutionPlan>,
    ctx: &SessionContext,
    codec: &dyn PhysicalExtensionCodec,
    proto_converter: &dyn PhysicalProtoConverterExtension,
) -> Result<Arc<dyn ExecutionPlan>> {
    let bytes = physical_plan_to_bytes_with_proto_converter(
        Arc::clone(&exec_plan), codec, proto_converter,
    )?;
    let result_exec_plan = physical_plan_from_bytes_with_proto_converter(
        bytes.as_ref(), ctx.task_ctx().as_ref(), codec, proto_converter,
    )?;
    pretty_assertions::assert_eq!(
        format!("{exec_plan:?}"),
        format!("{result_exec_plan:?}")
    );
    Ok(result_exec_plan)
}
```

Physical plan roundtrips use `pretty_assertions` for better diff output. This file also includes `roundtrip_test_with_context` (for plans needing registered tables) and `roundtrip_test_sql_with_context` (end-to-end SQL -> physical plan -> bytes -> plan).

**Pattern C: Expr Byte Serialization** (`serialize.rs`)

A lighter-weight pattern using the `Serializeable` trait:

```rust
// File: datafusion/proto/tests/cases/serialize.rs:102-105
fn roundtrip_expr(expr: &Expr) -> Expr {
    let bytes = expr.to_bytes().unwrap();
    Expr::from_bytes(&bytes).unwrap()
}
```

This is used for tests that focus on expression structure fidelity without needing a SessionContext.

### 2.2 Serializeable Trait and Bytes Module

The `Serializeable` trait (`datafusion/proto/src/bytes/mod.rs`) provides the `to_bytes`/`from_bytes` API. Notably, `to_bytes` for `Expr` includes a **built-in round-trip validation**: after encoding, it immediately tries to decode using a `PlaceHolderRegistry` to catch prost recursion-depth issues (issue #3968). This means every call to `Expr::to_bytes()` already performs a mini round-trip test.

Companion free functions:
- `logical_plan_to_bytes` / `logical_plan_from_bytes`
- `logical_plan_to_bytes_with_extension_codec` / `logical_plan_from_bytes_with_extension_codec`
- `physical_plan_to_bytes` / `physical_plan_from_bytes`
- `physical_plan_to_bytes_with_proto_converter` / `physical_plan_from_bytes_with_proto_converter`
- JSON variants: `logical_plan_to_json` / `logical_plan_from_json` (behind `json` feature)

### 2.3 ScalarValue Round-Trip Tests

The test `round_trip_scalar_values_and_data_types` (line 1544 of roundtrip_logical_plan.rs) is the most comprehensive, covering ~90+ ScalarValue variants:

**Null variants tested:** Boolean, Float32, Float64, Int8-Int64, UInt8-UInt64, Utf8, LargeUtf8, Date32, TimestampMicrosecond, TimestampNanosecond

**Non-null variants tested with boundary values:**
- Float32/Float64: 1.0, MAX, MIN, -2000.0
- Int8 through Int64: MIN, MAX, 0, -15
- UInt8 through UInt64: MAX, 0
- All timestamp units: with and without timezone ("UTC")
- All time32/time64 units: 0, MAX, None
- Date32/Date64: 0, MAX, None
- IntervalDayTime and IntervalMonthDayNano: with boundary values
- Utf8, LargeUtf8, Utf8View, BinaryView
- Binary, LargeBinary
- FixedSizeBinary: with data, empty (len=0), and null (len=5)

**Nested types tested:**
- `List(Float32)` with mixed null/non-null elements
- `LargeList(Float32)` with mixed null/non-null elements
- Nested `List(List(Float32))` -- 2 levels deep
- Nested `LargeList(LargeList(Float32))`
- `FixedSizeList(Int32, 3)`
- `Dictionary(Int32, Utf8)` -- both with value and null
- `RunEndEncoded(Int32, Utf8)` -- both with value and null
- `Struct { a: Int32, b: Boolean }` -- with values and null struct
- `Map { key: Int32, value: Utf8 }` -- null maps and populated MapArray

The test performs **two assertions per value**: (1) ScalarValue round-trip, (2) DataType round-trip.

Additionally, `round_trip_datatype` (line 1893) tests ~50 DataType variants including Null, all numerics, all timestamps, Duration, Interval, Binary types, Decimal128/256, recursive List/FixedSizeList, nested Struct, Union (Sparse + Dense with type IDs), Dictionary, and Map.

### 2.4 IPC Encoding for Nested ScalarValues

Nested ScalarValue types (List, FixedSizeList, LargeList, Struct, Map) cannot be represented as simple protobuf fields. DataFusion uses **Arrow IPC messages embedded within protobuf** for these:

```protobuf
// File: datafusion/proto-common/proto/datafusion_common.proto:202-213
message ScalarNestedValue {
  message Dictionary {
    bytes ipc_message = 1;
    bytes arrow_data = 2;
  }
  bytes ipc_message = 1;
  bytes arrow_data = 2;
  Schema schema = 3;
  repeated Dictionary dictionaries = 4;
}
```

Serialization (`to_proto/mod.rs:1037-1097`):
1. Wrap the Arrow array in a single-column `RecordBatch`
2. Use `IpcDataGenerator` to produce IPC messages
3. Separately encode any dictionary batches
4. Package the IPC bytes + schema into `ScalarNestedValue`

Deserialization (`from_proto/mod.rs:420-469`):
1. Reconstruct IPC dictionary IDs by re-encoding the schema through IPC (since protobuf Schema does not preserve IPC dictionary IDs)
2. Decode dictionary batches first
3. Use `read_record_batch` to reconstruct the Arrow RecordBatch from IPC message + dictionaries

This means **every nested ScalarValue roundtrip implicitly tests the Arrow IPC reader/writer path**.

### 2.5 Plan-Level Roundtrip Tests

**Logical plan tests (~80 functions)** cover:
- Basic plans: `roundtrip_logical_plan`, `roundtrip_logical_plan_aggregation`, `roundtrip_logical_plan_sort`
- DML: `roundtrip_logical_plan_dml` (INSERT INTO)
- COPY TO with all formats: Arrow, CSV, JSON, Parquet (each with specific writer options)
- Default codecs for file formats: CSV, JSON, Parquet, Arrow (`roundtrip_default_codec_*`)
- Custom tables: `roundtrip_custom_tables`, `roundtrip_custom_memory_tables`, `roundtrip_custom_listing_tables`
- Constraint preservation: tables with PRIMARY KEY, column defaults
- Views: `roundtrip_logical_plan_with_view_scan`
- Prepared statements with metadata: `roundtrip_logical_plan_prepared_statement_with_metadata`
- Recursive queries: `roundtrip_recursive_query`
- UNNEST: `roundtrip_logical_plan_unnest`
- Extension nodes: `roundtrip_logical_plan_with_extension` (custom TopK node)
- Expr API: `roundtrip_expr_api` (~70 expressions including array functions, aggregates, window functions)

**Physical plan tests (~60 functions)** cover:
- All operator types: Empty, Limit, Sort, Filter, Projection, Aggregate, Repartition, Union, Interleave, Unnest, Coalesce, Analyze
- Join variants: HashJoin (all JoinType x PartitionMode), NestedLoopJoin, SortMergeJoin, SymmetricHashJoin
- Window functions: WindowAggExec, BoundedWindowAggExec, lead with default values
- File sources/sinks: Parquet (with pruning predicates, table partition cols, custom predicate), Arrow, JSON, CSV
- UDFs: scalar UDF, aggregate UDF, window UDF -- each with extension codec
- Special expressions: HashExpr, HashTableLookupExpr (degrades to lit(true) on serialize)
- TPC-H queries: `test_serialize_deserialize_tpch_queries` and `test_round_trip_tpch_queries` -- all 22 TPC-H queries
- Memory source: `roundtrip_memory_source`
- Async function: `roundtrip_async_func_exec`
- Listing table with schema metadata: `roundtrip_listing_table_with_schema_metadata`

### 2.6 Expression Deduplication Tests

A newer feature (PR #18192) introduces `DeduplicatingProtoConverter` which caches deserialized expressions by their protobuf bytes to share `Arc`s:

```rust
// File: roundtrip_physical_plan.rs:2704-2762
fn test_expression_deduplication() -> Result<()> {
    // ... creates shared_col Arc used in InList and BinaryExpr
    let proto_converter = DeduplicatingProtoConverter {};
    // ... roundtrip with deduplicating converter
    // Verifies plan structure is preserved
}

// File: roundtrip_physical_plan.rs:2764-2831
fn test_expression_deduplication_arc_sharing() -> Result<()> {
    // Creates projection using same Arc twice
    // After roundtrip, verifies Arc::ptr_eq(&exprs[0].expr, &exprs[1].expr)
}
```

Also tested: `test_backward_compatibility_no_expr_id` (protos without expr_id still work) and `test_deduplication_within_plan_deserialization` (separate deserializations produce independent Arcs).

### 2.7 Extension Codec Testing Pattern

Custom UDFs are tested via a well-defined pattern in `cases/mod.rs`:

1. Define test UDF types (e.g., `MyRegexUdf`, `MyAggregateUDF`, `CustomUDWF`)
2. Define corresponding protobuf node types (e.g., `MyRegexUdfNode` using `#[derive(prost::Message)]`)
3. Implement `LogicalExtensionCodec` or `PhysicalExtensionCodec` with encode/decode for those types
4. Test roundtrip using the custom codec

The `TopKExtensionCodec` example (roundtrip_logical_plan.rs:1382-1450) is particularly instructive -- it shows how to serialize a completely custom logical plan node through protobuf.

### 2.8 FFI Codec Roundtrip Tests

The FFI module (`datafusion/ffi/src/proto/logical_extension_codec.rs`) tests cross-library serialization:

```rust
fn roundtrip_ffi_logical_extension_codec_table_provider() -> Result<()>  // TableProvider
fn roundtrip_ffi_logical_extension_codec_udf() -> Result<()>              // ScalarUDF
fn roundtrip_ffi_logical_extension_codec_udaf() -> Result<()>             // AggregateUDF
fn roundtrip_ffi_logical_extension_codec_udwf() -> Result<()>             // WindowUDF
```

These wrap the codec in `FFI_LogicalExtensionCodec`, set a mock foreign marker ID, and verify the round-trip produces the correct concrete type via `downcast_ref` / `is::<T>()`.

### 2.9 IPC Stream Roundtrip (Spill Tests)

The spill subsystem (`datafusion/physical-plan/src/spill/mod.rs`) provides indirect IPC round-trip testing:

```rust
async fn test_batch_spill_and_read() -> Result<()> {
    let batch1 = build_table_i32(("a2", &vec![0, 1, 2]), ...);
    let batch2 = build_table_i32(("a2", &vec![10, 11, 12]), ...);
    // Write via IPCStreamWriter, read back, verify len
}

async fn test_batch_spill_and_read_dictionary_arrays() -> Result<()> {
    // Dictionary-encoded Int32 arrays through IPC roundtrip
    // Verifies schema equality after roundtrip
}
```

These test the `IPCStreamWriter` -> `StreamReader` path with `IpcWriteOptions` configured for correct alignment based on schema.

### 2.10 RecordBatch Construction Test Utilities

**`datafusion/physical-plan/src/test.rs`** provides reusable helpers:

| Function | Signature | Purpose |
|----------|-----------|---------|
| `build_table_i32` | `((&str, &Vec<i32>), ...) -> RecordBatch` | 3-column Int32 batch |
| `build_table_i32_two_cols` | `((&str, &Vec<i32>), ...) -> RecordBatch` | 2-column Int32 batch |
| `build_table_scan_i32` | Same as above but returns `Arc<dyn ExecutionPlan>` | Wrapped in TestMemoryExec |
| `aggr_test_schema` | `() -> SchemaRef` | 13-column schema matching aggregate test CSV |

**`datafusion/core/src/test_util/mod.rs`** has higher-level utilities including `TestTableProvider` and `TestTableFactory`.

Physical plan tests also commonly use `EmptyExec::new(schema)` as a lightweight input source when only schema matters.

### 2.11 Substrait Roundtrip Tests

DataFusion also has Substrait (cross-engine plan format) roundtrip tests in `datafusion/substrait/tests/cases/roundtrip_logical_plan.rs`, testing ~40+ SQL patterns through the Substrait producer/consumer path. These follow the same pattern (SQL -> LogicalPlan -> Substrait -> LogicalPlan -> compare) and cover joins, subqueries, aggregates, set operations, and more.

### 2.12 Deeply Nested Expression Tests

`serialize.rs` includes specific stress tests for deeply nested expression trees:

- `roundtrip_deeply_nested_binary_expr`: 100-deep chain of `(a < 5) OR ...`
- `roundtrip_deeply_nested_binary_expr_reverse_order`: reverse-associativity nesting
- `roundtrip_deeply_nested`: incremental depth from 1 to 100 with alternating AND/OR

These run on threads with enlarged stack sizes (10-20MB) to avoid stack overflow in debug builds, specifically testing prost's recursion limits.

## 3. Key Design Insights

1. **Debug-format equality is the standard assertion method.** DataFusion does not require `PartialEq` on all plan nodes. Instead, `format!("{:?}", plan)` is compared before and after roundtrip. This is pragmatic but means some information loss could go undetected if it does not appear in Debug output. The test comment explicitly acknowledges this: "this often isn't sufficient to guarantee that no information is lost during serde because the string representation of a plan often only shows a subset of state."

2. **No property-based or fuzz testing exists for serialization.** The proto crate has zero dependencies on proptest, quickcheck, or Arbitrary. All tests are manually constructed. This is a notable gap -- random type/value generation could find edge cases in the IPC encoding path.

3. **Nested ScalarValues use Arrow IPC as an inner transport.** Rather than defining protobuf messages for every Arrow array type, DataFusion wraps arrays in a RecordBatch and uses IPC. This is elegant but creates a dependency chain: protobuf -> IPC bytes -> Arrow. Testing nested ScalarValue roundtrips implicitly tests a significant portion of the Arrow IPC path.

4. **The Serializeable::to_bytes() method has a built-in self-test.** After encoding an Expr, it immediately tries to decode it with a placeholder registry. This catches prost recursion issues at serialization time rather than at deserialization time.

5. **Extension codec pattern enables pluggable serialization.** The `LogicalExtensionCodec` / `PhysicalExtensionCodec` traits allow downstream crates to add serialization for custom plan nodes, UDFs, and table providers. The test suite exercises this with `TopKExtensionCodec`, `UDFExtensionCodec`, and `TestTableProviderCodec`.

6. **TPC-H queries serve as integration tests.** All 22 TPC-H queries are run through plan serialization roundtrip, providing broad coverage of real-world plan structures.

7. **Expression deduplication preserves Arc sharing across serialization boundaries.** The `DeduplicatingProtoConverter` tests verify not just value equality but `Arc::ptr_eq`, ensuring memory sharing semantics survive the roundtrip.

8. **Backward compatibility is explicitly tested.** `test_backward_compatibility_no_expr_id` verifies that protos from older versions (without the `expr_id` field) can still be deserialized.

## 4. Porting Considerations (DataFusion Patterns -> Trino Rust Worker)

### Patterns to Adopt

1. **Layered roundtrip testing.** Test serialization at every abstraction level independently: types, scalars, expressions, plans. DataFusion's layered approach means a failure at one level does not mask issues at another. For the Trino worker, this means testing Trino's type system serialization separately from plan fragment serialization.

2. **Enumeration-based ScalarValue tests.** The `round_trip_scalar_values_and_data_types` pattern of exhaustively listing every type variant (including boundary values like MIN/MAX/NULL) is effective. For Trino's type system, create an equivalent exhaustive list covering all Trino types (BOOLEAN, TINYINT through BIGINT, REAL, DOUBLE, DECIMAL, VARCHAR, VARBINARY, DATE, TIME, TIMESTAMP, INTERVAL, ARRAY, MAP, ROW, etc.).

3. **IPC for complex nested types.** DataFusion's approach of using Arrow IPC to transport nested types within protobuf is battle-tested. If the Trino worker needs to serialize Arrow data within Thrift messages (Trino's RPC format), embedding IPC messages is a proven approach.

4. **Extension codec architecture.** The pluggable codec pattern is valuable for a Trino worker that may need to serialize custom connectors' plan fragments. Design the serialization layer with a trait-based extension point from the start.

5. **SQL-based integration tests.** Running full queries through the serialization pipeline (like the TPC-H tests) catches issues that unit tests on individual nodes miss. Use TPC-H and TPC-DS queries as integration tests for the Trino worker's plan fragment serialization.

### Patterns to Improve Upon

1. **Add property-based testing.** DataFusion's biggest gap is the absence of fuzz/property testing. For the Trino worker, use proptest to generate random `ScalarValue` instances, random schemas, and random expression trees, then verify roundtrip invariants. This is especially important for the Arrow IPC path where buffer alignment and dictionary encoding have subtle corner cases.

2. **Use structural equality, not Debug-format comparison.** DataFusion's reliance on `format!("{:?}")` for plan comparison is fragile. Implement `PartialEq` on plan nodes (or a dedicated comparison trait) for the Trino worker so that roundtrip tests catch all information loss, not just what appears in Debug output.

3. **Test byte-level compatibility and versioning.** DataFusion only has one backward compatibility test (`test_backward_compatibility_no_expr_id`). For the Trino worker, maintain a corpus of serialized plan fragments from each release and verify that newer code can always deserialize older formats. This is critical for rolling upgrades.

4. **Test concurrent serialization.** DataFusion's tests are all single-threaded. The Trino worker will serialize/deserialize plan fragments across many concurrent tasks. Add stress tests that serialize/deserialize from multiple threads simultaneously to catch any accidental shared mutable state.

### Trino-Specific Considerations

1. **Trino's Thrift-based PrestoPage format** is the equivalent of DataFusion's protobuf-serialized plans. The Trino worker will need roundtrip tests for `PrestoPage` (the wire format for data exchange between workers), testing all Trino types through the encode/decode path.

2. **Trino's `Split` and `PlanFragment` serialization** uses JSON over HTTP. Roundtrip tests should cover the JSON serialization of plan fragments, including all node types (TableScan, Filter, Project, Join, Aggregate, Exchange, etc.).

3. **Dictionary encoding in Arrow IPC** requires special attention. DataFusion's spill test `test_batch_spill_and_read_dictionary_arrays` specifically tests dictionary-encoded arrays through IPC. This is relevant because Trino's VARCHAR columns will naturally be dictionary-encoded in Arrow representation.

4. **The `Serializeable::to_bytes()` self-test pattern** is worth adopting. Having the serializer immediately verify its own output catches encoding bugs at the point of creation rather than at the (potentially remote) point of consumption.
