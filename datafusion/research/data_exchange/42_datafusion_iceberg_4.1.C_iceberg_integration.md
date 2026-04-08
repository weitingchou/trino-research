# Module Teardown: DataFusion Iceberg Integration & Delete Handling (Task 4.1.C)

## Table of Contents
- [0. Research Focus](#0-research-focus)
- [1. High-Level Overview](#1-high-level-overview)
- [2. Detailed Analysis](#2-detailed-analysis)
  - [2.1 TableProvider and Scan Translation](#21-tableprovider-and-scan-translation)
  - [2.2 Filter Translation: DataFusion Expr to Iceberg Predicate](#22-filter-translation-datafusion-expr-to-iceberg-predicate)
  - [2.3 Delete File Handling (MOR Pipeline)](#23-delete-file-handling-mor-pipeline)
  - [2.4 Schema Evolution](#24-schema-evolution)
  - [2.5 Multi-Level Predicate Pushdown](#25-multi-level-predicate-pushdown)
  - [2.6 Snapshot Isolation and Time Travel](#26-snapshot-isolation-and-time-travel)
- [3. Key Design Insights](#3-key-design-insights)
- [4. Comparison with Trino's Java Iceberg Connector](#4-comparison-with-trinos-java-iceberg-connector)
- [5. Implications for Rust Worker Design](#5-implications-for-rust-worker-design)

---

## 0. Research Focus

This teardown examines the `apache/iceberg-rust` project's DataFusion integration, focusing on:
1. How the Iceberg `TableProvider` translates `scan()` into Iceberg table scans
2. Delete file handling: positional deletes, equality deletes, and MOR architecture
3. Schema evolution strategy (column ID vs name-based mapping)
4. Multi-level predicate pushdown pipeline
5. Snapshot isolation and time-travel mechanisms
6. Maturity comparison vs Trino's Java Iceberg connector

**Repository:** `apache/iceberg-rust` on GitHub, branch `main`
**Key paths:**
- `crates/integrations/datafusion/src/` -- DataFusion TableProvider + physical plan
- `crates/iceberg/src/scan/` -- Scan planning (mod.rs, context.rs, task.rs)
- `crates/iceberg/src/arrow/` -- ArrowReader, delete filters, schema evolution
- `crates/iceberg/src/expr/visitors/` -- Multi-level predicate evaluators

---

## 1. High-Level Overview

The `iceberg-rust` crate provides a complete Rust-native Iceberg implementation with a DataFusion integration layer. The architecture is cleanly split:

```
                DataFusion Layer                          Iceberg Core
          (crates/integrations/datafusion)              (crates/iceberg)
 +------------------------------------+      +------------------------------------+
 | IcebergTableProvider               |      | Table                              |
 | IcebergStaticTableProvider         |      | TableScan / TableScanBuilder       |
 |   scan() -> IcebergTableScan       | ---> | ArrowReader                        |
 |                                    |      | CachingDeleteFileLoader            |
 | expr_to_predicate.rs               |      | DeleteFilter / DeleteFileIndex     |
 |   DataFusion Expr -> Iceberg Pred  |      | ManifestEvaluator                  |
 +------------------------------------+      | ExpressionEvaluator                |
                                             | RowGroupMetricsEvaluator           |
                                             | PageIndexEvaluator                 |
                                             | RecordBatchTransformer             |
                                             +------------------------------------+
```

**Two TableProvider implementations:**
- `IcebergTableProvider` -- Catalog-backed, reloads metadata on every scan/write. For write operations or when latest state is needed.
- `IcebergStaticTableProvider` -- Holds a cached `Table` instance. For read-only queries, time-travel, and avoiding catalog overhead.

The DataFusion integration is intentionally thin. `IcebergTableScan` (the `ExecutionPlan`) simply calls `table.scan()` from the core crate, which does the heavy lifting of manifest planning, file pruning, delete application, and Parquet reading. DataFusion's role is limited to (a) translating `Expr` filters to Iceberg predicates, (b) wrapping the result stream, and (c) applying LIMIT post-stream.

---

## 2. Detailed Analysis

### 2.1 TableProvider and Scan Translation

**File:** `crates/integrations/datafusion/src/table/mod.rs`

Both providers implement DataFusion's `TableProvider::scan()` with the same pattern:

```rust
async fn scan(
    &self,
    _state: &dyn Session,
    projection: Option<&Vec<usize>>,
    filters: &[Expr],
    limit: Option<usize>,
) -> DFResult<Arc<dyn ExecutionPlan>> {
    // IcebergTableProvider reloads table from catalog here
    // IcebergStaticTableProvider uses cached table
    Ok(Arc::new(IcebergTableScan::new(
        table,
        self.snapshot_id,   // None for current, Some(id) for time-travel
        self.schema.clone(),
        projection,
        filters,
        limit,
    )))
}
```

**Projection handling:** Column indices from DataFusion are mapped to column names using the Arrow schema. The Iceberg scan layer then maps names to Iceberg field IDs for field-ID-based projection.

**Filter pushdown:** `supports_filters_pushdown()` returns `Inexact` for all filters, indicating they are pushed down on a best-effort basis. Any filter that cannot be converted to an Iceberg predicate is silently dropped (the scanner handles the fallback).

**Limit:** Pushed down to `IcebergTableScan`, applied as a streaming filter after `RecordBatch`es arrive from the Iceberg scan. The limit is NOT pushed into the Iceberg scan planner (no manifest/file pruning based on limit). It is applied post-stream by slicing record batches.

**File:** `crates/integrations/datafusion/src/physical_plan/scan.rs`

The `IcebergTableScan::execute()` method:
1. Calls `get_batch_stream()` which constructs a `TableScanBuilder`
2. Applies snapshot_id, column selection, and filter predicate
3. Calls `table_scan.to_arrow()` to get a stream of RecordBatches
4. Wraps the stream with a limit-applying combinator if `limit` is set

```rust
// The core flow in get_batch_stream():
let scan_builder = table.scan().snapshot_id(snapshot_id);
let scan_builder = scan_builder.select(column_names);
scan_builder = scan_builder.with_filter(pred);
let table_scan = scan_builder.build()?;
let stream = table_scan.to_arrow().await?;
```

**Critical observation:** The DataFusion integration currently uses `Partitioning::UnknownPartitioning(1)` -- a single partition. This means DataFusion cannot parallelize across Iceberg files at the plan level. Parallelism happens internally within `ArrowReader::read()` using `try_buffer_unordered(concurrency_limit_data_files)`.

### 2.2 Filter Translation: DataFusion Expr to Iceberg Predicate

**File:** `crates/integrations/datafusion/src/physical_plan/expr_to_predicate.rs`

The `convert_filters_to_predicate()` function recursively walks DataFusion's `Expr` tree and maps it to Iceberg's `Predicate` type.

**Supported conversions:**

| DataFusion Expr | Iceberg Predicate |
|---|---|
| `col = literal` | `BinaryExpression(Eq, ref, datum)` |
| `col != literal` | `BinaryExpression(NotEq, ref, datum)` |
| `col > / >= / < / <= literal` | Corresponding comparison operators |
| `col IS NULL / IS NOT NULL` | `UnaryExpression(IsNull/NotNull, ref)` |
| `col IN (list)` | `ref.is_in(datums)` |
| `col NOT IN (list)` | `ref.is_not_in(datums)` |
| `NOT expr` | Negation of inner predicate |
| `expr AND expr` | `Predicate::and()` (partial: keeps whichever side converts) |
| `expr OR expr` | `Predicate::or()` (both sides must convert or drops entirely) |
| `col LIKE 'prefix%'` | `StartsWith` / `NotStartsWith` |
| `isnan(col)` | `UnaryExpression(IsNan, ref)` |
| `CAST(expr)` (non-date) | Recurse into inner expr |
| `literal op col` | Reverse operator direction |

**Notable design choices:**
- AND is permissive: if only one side converts, the converted side is kept. This maximizes pushdown.
- OR is strict: both sides must convert, else the entire OR is dropped. This prevents incorrect results.
- Date casts are explicitly rejected (would truncate precision and create wrong predicates).
- ILIKE (case-insensitive) is NOT pushed down since Iceberg's StartsWith is case-sensitive.
- Complex expressions (arithmetic, function calls other than isnan) are NOT pushed down.

**Datum conversion** handles: Boolean, Int8-64, Float32/64, Utf8/LargeUtf8, Binary/LargeBinary, Date32/64, TimestampMicrosecond, TimestampNanosecond. TimestampSecond and TimestampMillisecond are NOT converted because DataFusion's type coercion normalizes them first.

### 2.3 Delete File Handling (MOR Pipeline)

This is the most architecturally interesting part. The delete handling pipeline involves four major components working together.

#### 2.3.1 DeleteFileIndex -- Scan-Time Delete File Assignment

**File:** `crates/iceberg/src/delete_file_index.rs`

During scan planning, a `DeleteFileIndex` is built asynchronously. Delete manifest entries are streamed through a channel and accumulated into a `PopulatedDeleteFileIndex` containing:
- `global_equality_deletes` -- Equality deletes with empty partitions (apply to all data files)
- `eq_deletes_by_partition` -- HashMap<Struct, Vec<DeleteFileContext>>
- `pos_deletes_by_partition` -- HashMap<Struct, Vec<DeleteFileContext>>

When a data file's `FileScanTask` is created, `get_deletes_for_data_file()` queries this index to find applicable deletes:
- **Equality deletes:** Global deletes apply if their sequence number > data file's sequence number. Partitioned deletes must also match the partition value AND partition spec ID.
- **Positional deletes:** Same partition/spec matching, but sequence number check uses `>=` (not `>`), per Iceberg spec.

**Key detail:** Manifest files are sorted to process delete manifests FIRST, preventing a deadlock where the data channel fills while the delete consumer is waiting. This is a subtle concurrency design.

#### 2.3.2 CachingDeleteFileLoader -- Loading and Caching Delete Files

**File:** `crates/iceberg/src/arrow/caching_delete_file_loader.rs`

This is the central orchestrator for delete file loading. Its `load_deletes()` method:

1. **Deduplication:** Uses `DeleteFilter` state to avoid loading the same delete file twice (important when multiple data files share the same delete file).
2. **Phase 1 (Load):** Opens Parquet delete files and creates RecordBatch streams.
   - Positional deletes: Uses `try_start_pos_del_load()` to coordinate via `PosDelLoadAction::Load/AlreadyLoaded/WaitFor`.
   - Equality deletes: Uses `try_start_eq_del_load()` to claim ownership of loading, or skip if another task is already loading it.
3. **Phase 2 (Parse):**
   - **Positional deletes** -> Parsed into `HashMap<String, DeleteVector>` (data_file_path -> RoaringTreemap of row positions). Each pos delete Parquet file has columns `file_path` (string) and `pos` (int64).
   - **Equality deletes** -> Parsed into an Iceberg `Predicate` via `parse_equality_deletes_record_batch_stream()`. Each row becomes `NOT (col1 = val1 AND col2 = val2 ...)`, and all row predicates are combined with AND using a balanced binary tree (to avoid stack overflow from deep nesting).
4. **Phase 3 (Merge):** Delete vectors are upserted into `DeleteFilter` state. Equality delete predicates are sent via oneshot channels.

The data flow diagram (from the source code):
```
                 FileScanTaskDeleteFile
                          |
                  Skip Started EQ Deletes
                          |
            [load recordbatch stream / puffin]
                   DeleteFileContext
                          |
    +---------------------+--------------------+
  Pos Del              Del Vec (NYI)        EQ Del
    |                     |                    |
 [parse stream]    [parse puffin]        [parse eq del]
 HashMap<path,DV>  HashMap<path,DV>     (Predicate, Sender)
    |                     |                    |
    |                     |            [persist to state]
    +---------------------+--------------------+
                          |
                   [buffer unordered]
                          |
                  [combine del vectors]
                          |
                 [persist to state]
```

#### 2.3.3 DeleteFilter -- State Management

**File:** `crates/iceberg/src/arrow/delete_filter.rs`

Thread-safe state container for loaded deletes:
- `delete_vectors: HashMap<String, Arc<Mutex<DeleteVector>>>` -- Pos delete vectors keyed by data file path
- `equality_deletes: HashMap<String, EqDelState>` -- Loading/Loaded state per eq delete file
- `positional_deletes: HashMap<String, PosDelState>` -- Loading/Loaded state per pos delete file

Uses `RwLock` for the outer state and `Notify` for signaling between concurrent tasks.

**Key method:** `build_equality_delete_predicate()` -- Collects all equality delete predicates for a task's associated delete files, ANDs them together, and binds the combined predicate to the task's schema.

#### 2.3.4 DeleteVector -- Efficient Row Position Storage

**File:** `crates/iceberg/src/delete_vector.rs`

Uses `RoaringTreemap` (from the `roaring` crate) for memory-efficient storage of deleted row positions. Supports:
- Individual insert
- Bulk append (for sorted positions)
- BitOr merge of multiple delete vectors
- Custom iterator with `advance_to()` for efficient RowSelection building

#### 2.3.5 Where Deletes Are Applied in the Reader

**File:** `crates/iceberg/src/arrow/reader.rs` (`process_file_scan_task`)

The delete application happens at two distinct stages inside `ArrowReader::process_file_scan_task()`:

**Stage A: Equality deletes become row-level filters.**
The equality delete predicate is combined with the scan's filter predicate via logical AND:
```rust
let delete_predicate = delete_filter.build_equality_delete_predicate(&task).await?;
let final_predicate = match (&task.predicate, delete_predicate) {
    (Some(filter), Some(delete)) => Some(filter.clone().and(delete)),
    (None, Some(delete)) => Some(delete.clone()),
    (Some(filter), None) => Some(filter.clone()),
    (None, None) => None,
};
```
This combined predicate then flows into:
1. **RowGroup filtering** (`get_selected_row_group_indices`) -- Row groups whose statistics prove they cannot contain matching rows are skipped.
2. **RowFilter** (`get_row_filter`) -- An Arrow predicate function is built from the combined predicate and applied as a Parquet `RowFilter`, which evaluates row-by-row during decoding.
3. **Row selection** (page-level, when `row_selection_enabled`) -- Page-level statistics are used to skip ranges of rows within surviving row groups.

**Stage B: Positional deletes become RowSelection masks.**
```rust
let positional_delete_indexes = delete_filter.get_delete_vector(&task);
if let Some(positional_delete_indexes) = positional_delete_indexes {
    let delete_row_selection = Self::build_deletes_row_selection(
        metadata.row_groups(),
        &selected_row_group_indices,
        &positional_delete_indexes,
    )?;
    // Intersect with any existing row_selection from filter predicate
    row_selection = match row_selection {
        None => Some(delete_row_selection),
        Some(filter_rs) => Some(filter_rs.intersection(&delete_row_selection)),
    };
}
```
The `build_deletes_row_selection()` method converts the `DeleteVector` (RoaringTreemap) into Parquet's `RowSelection` type -- a list of `RowSelector::select(n)` and `RowSelector::skip(n)` segments. This is applied before any data is decoded, so deleted rows are never materialized.

**Summary of MOR pipeline stages:**

| Stage | What | Mechanism | When Applied |
|---|---|---|---|
| Manifest pruning | Skip delete manifests that don't match scan filter | ManifestEvaluator | Plan time |
| Delete file index | Build index of which deletes apply to which data files | Partition + seq_num matching | Plan time |
| Delete file loading | Load & parse delete files (cached/shared) | CachingDeleteFileLoader | Before Parquet read |
| Equality deletes | Converted to filter predicate | RowFilter + RowGroup/Page pruning | During Parquet decode |
| Positional deletes | Converted to RowSelection mask | RoaringTreemap -> RowSelection | During Parquet decode |

**Critical insight:** Both delete types are applied DURING Parquet decode, not as a post-filter on RecordBatches. Equality deletes become part of the Parquet RowFilter. Positional deletes become part of the Parquet RowSelection. This means deleted rows are never fully materialized in memory -- they are skipped at the Parquet decoder level.

### 2.4 Schema Evolution

**Files:** `crates/iceberg/src/arrow/reader.rs`, `crates/iceberg/src/arrow/record_batch_transformer.rs`

The schema evolution strategy uses **field-ID-based mapping** (like Trino), not name-based mapping. This is directly aligned with the Iceberg spec's Column Projection rules.

**Three-branch resolution strategy** (matching Java's `ReadConf`):

1. **Branch 1: File has embedded field IDs** (normal case) -- Trust the Parquet metadata field IDs. Use `pruneColumns()` equivalent with field-ID-based projection.

2. **Branch 2: Name mapping present** (migration from Hive/Spark) -- Apply the table's `schema.name-mapping.default` property to assign field IDs to columns. Then use field-ID-based projection.

3. **Branch 3: Fallback** (no field IDs, no name mapping) -- Assign position-based fallback field IDs. Use position-based projection.

The code explicitly checks for missing field IDs:
```rust
let missing_field_ids = arrow_metadata.schema().fields().iter().next()
    .is_some_and(|f| f.metadata().get(PARQUET_FIELD_ID_META_KEY).is_none());
```

**Type promotion** is supported for:
- `Int -> Long`
- `Float -> Double`
- `Decimal(p1, s) -> Decimal(p2, s)` where p2 >= p1
- `Fixed(16) -> Uuid`

**Missing columns** (new columns added after file was written): The `RecordBatchTransformer` handles this by:
- Adding columns with default/NULL values for fields present in the scan schema but absent in the file
- Reordering columns to match the projected schema order
- Applying type promotion casts where needed
- Adding partition constants for identity-transformed partition fields

### 2.5 Multi-Level Predicate Pushdown

The iceberg-rust implementation has a **complete 5-level predicate pushdown pipeline**:

#### Level 1: Manifest Pruning
**File:** `crates/iceberg/src/expr/visitors/manifest_evaluator.rs`

`ManifestEvaluator` evaluates the scan predicate against each `ManifestFile`'s partition field summaries (min/max/null counts per partition field). Manifest files whose summary stats prove no matching data can exist are skipped entirely. Applied in `PlanContext::build_manifest_file_contexts()`.

#### Level 2: Partition Expression Evaluation
**File:** `crates/iceberg/src/expr/visitors/expression_evaluator.rs`

`ExpressionEvaluator` evaluates the partition-bound predicate against each `DataFile`'s partition `Struct`. Data files whose partition values don't match the filter are skipped. Applied in `TableScan::process_data_manifest_entry()`.

#### Level 3: File-Level Metrics (Inclusive Metrics)
**File:** `crates/iceberg/src/expr/visitors/inclusive_metrics_evaluator.rs`

`InclusiveMetricsEvaluator` evaluates the snapshot-bound predicate against each data file's column-level statistics (min/max values, null counts, NaN counts stored in the manifest entry). Files whose statistics prove they cannot contain matching rows are skipped. Applied in `TableScan::process_data_manifest_entry()`.

#### Level 4: Row Group Pruning
**File:** `crates/iceberg/src/expr/visitors/row_group_metrics_evaluator.rs`

`RowGroupMetricsEvaluator` evaluates the predicate against each Parquet row group's column statistics. Row groups that cannot contain matching data are excluded from the read. Enabled by default (`row_group_filtering_enabled: true`).

Uses `IN_PREDICATE_LIMIT = 200` -- IN predicates with more than 200 values are not evaluated at this level.

#### Level 5: Page Index Pruning
**File:** `crates/iceberg/src/expr/visitors/page_index_evaluator.rs`

`PageIndexEvaluator` uses the Parquet page index (column index + offset index) to create a `RowSelection` that skips ranges of rows within surviving row groups. **Disabled by default** (`row_selection_enabled: false`) because parsing the page index can outweigh gains.

The builder comments note:
> It is recommended to experiment with partitioning, sorting, row group size, page size, and page row limit Iceberg settings on the table being scanned in order to get the best performance from using row selection.

#### Level 6: Row-Level Filter
Applied via Parquet's `RowFilter` mechanism. The `get_row_filter()` method in ArrowReader builds an `ArrowPredicateFn` that evaluates the predicate against decoded columns. This happens during decode, allowing the reader to skip materializing non-matching rows.

**Summary of multi-level pushdown:**

| Level | Component | Granularity | Default |
|---|---|---|---|
| 1. Manifest | ManifestEvaluator | Entire manifest file | Always on |
| 2. Partition | ExpressionEvaluator | Individual data file | Always on |
| 3. File metrics | InclusiveMetricsEvaluator | Individual data file | Always on |
| 4. Row group | RowGroupMetricsEvaluator | Parquet row group | On by default |
| 5. Page index | PageIndexEvaluator | Page ranges within row group | Off by default |
| 6. Row filter | ArrowPredicateFn/RowFilter | Individual rows | Always on (when predicate exists) |

### 2.6 Snapshot Isolation and Time Travel

**Files:** `crates/iceberg/src/scan/mod.rs`, `crates/integrations/datafusion/src/table/mod.rs`

**Snapshot selection** happens at `TableScanBuilder::build()`:

```rust
let snapshot = match self.snapshot_id {
    Some(snapshot_id) => self.table.metadata().snapshot_by_id(snapshot_id)
        .ok_or_else(|| Error::new(ErrorKind::DataInvalid, ...))?
        .clone(),
    None => {
        let Some(current_snapshot_id) = self.table.metadata().current_snapshot() else {
            // No snapshot = empty table, return empty scan
            return Ok(TableScan { plan_context: None, ... });
        };
        current_snapshot_id.clone()
    }
};
```

The snapshot is locked at scan build time. All subsequent operations (manifest list traversal, manifest reading, file scanning) use this single consistent snapshot.

**Time-travel** is supported through `IcebergStaticTableProvider`:
```rust
pub async fn try_new_from_table_snapshot(table: Table, snapshot_id: i64) -> Result<Self> {
    let snapshot = table.metadata().snapshot_by_id(snapshot_id)
        .ok_or_else(|| Error::new(...))?;
    let table_schema = snapshot.schema(table.metadata())?;
    // Uses snapshot's schema (may differ from current schema due to evolution)
    let schema = Arc::new(schema_to_arrow_schema(&table_schema)?);
    Ok(IcebergStaticTableProvider { table, snapshot_id: Some(snapshot_id), schema })
}
```

Notable: when time-traveling, the schema is resolved from the snapshot (which records its `schema_id`), not the table's current schema. This ensures correct schema evolution behavior.

**IcebergTableProvider** (catalog-backed) always uses the current snapshot and always reloads metadata from the catalog on each scan. There is no explicit version negotiation or optimistic concurrency control in the read path -- the catalog's `load_table()` returns whatever is current at call time.

---

## 3. Key Design Insights

### 3.1 Thin DataFusion Layer, Heavy Core
The DataFusion integration is deliberately minimal (~300 lines for the physical plan scan, ~300 lines for the table provider). All complex logic lives in the `iceberg` core crate. This makes the Iceberg implementation reusable for other query engines (e.g., the Python bindings also use the same core).

### 3.2 Deletes Applied at Parquet Decoder Level
Both positional and equality deletes are converted into Parquet-native mechanisms (RowSelection and RowFilter) rather than post-filtering RecordBatches. This means deleted rows never consume memory for column data. This is a significant optimization compared to approaches that decode all rows and then filter.

### 3.3 Concurrent Delete Loading with Caching
The `CachingDeleteFileLoader` uses a clever pattern of `Notify`-based coordination to avoid duplicate work when multiple data files share the same delete files. The `try_start_pos_del_load()` / `try_start_eq_del_load()` methods implement a lock-free claim mechanism. This is important for large tables where many data files in the same partition share the same set of delete files.

### 3.4 Equality Deletes as Predicates (Not Hash Joins)
Equality deletes are converted into negated Iceberg predicates: `NOT (col1 = v1 AND col2 = v2)` for each deleted row, combined with AND. This allows them to participate in the same multi-level pushdown pipeline as the scan filter. The downside is that for large equality delete files, the predicate tree can become very large. The balanced binary tree construction mitigates stack overflow but not memory pressure.

### 3.5 Single-Partition DataFusion Plan
`IcebergTableScan` reports `Partitioning::UnknownPartitioning(1)` to DataFusion. This means DataFusion sees it as a single-partition leaf node and cannot parallelize above it. Parallelism is entirely internal to the Iceberg reader (via `try_buffer_unordered`). This is a maturity gap -- Trino, for example, distributes `FileScanTask`s across workers.

### 3.6 Delete Manifests Processed First
The scan planner sorts manifest files to process delete manifests before data manifests. This is a critical concurrency detail: the delete file index must be populated before data `FileScanTask`s can query it. Without this ordering, a deadlock can occur if the data channel fills while the delete consumer is still waiting.

---

## 4. Comparison with Trino's Java Iceberg Connector

| Aspect | iceberg-rust + DataFusion | Trino Java Iceberg Connector |
|---|---|---|
| **Architecture** | Single-process, internal parallelism | Distributed, coordinator assigns splits to workers |
| **Filter pushdown** | 6-level (manifest -> partition -> file metrics -> row group -> page index -> row) | Similar multi-level, plus dynamic filtering from joins |
| **Equality deletes** | Converted to negated predicates, applied via RowFilter | Loaded into hash sets, applied as post-filter on decoded batches (some optimizations with predicate pushdown in newer versions) |
| **Positional deletes** | RowSelection mask (rows never decoded) | RowSelection mask (similar approach in recent versions; older versions used post-filter) |
| **Deletion vectors (V3)** | Not yet implemented (TODO in code) | Supported via Puffin file reader |
| **Schema evolution** | Field-ID-based, 3-branch strategy (embedded IDs / name mapping / fallback) | Field-ID-based, similar strategy with `ParquetSchemaUtil` |
| **Type promotion** | Int->Long, Float->Double, Decimal widening, Fixed(16)->Uuid | Same set plus potentially more edge cases |
| **Snapshot isolation** | Snapshot locked at scan build time | Snapshot locked at split generation time |
| **Time travel** | `IcebergStaticTableProvider` with snapshot_id | `FOR TIMESTAMP AS OF` / `FOR VERSION AS OF` SQL syntax |
| **Parallelism model** | Internal (buffered unordered streams within single process) | External (coordinator distributes FileScanTasks as splits to workers) |
| **Partition-aware planning** | Not exposed to DataFusion (single partition) | Splits distributed by partition to enable partition-aware joins |
| **Write support** | Yes (via IcebergTableProvider + IcebergWriteExec + IcebergCommitExec) | Yes (CTAS, INSERT, MERGE, UPDATE, DELETE) |
| **DML maturity** | Append-only writes, no MERGE/UPDATE/DELETE yet | Full DML including MERGE, UPDATE, DELETE |
| **Catalog support** | REST, Hive, Glue, SQL (memory catalog built-in) | REST, Hive, Glue, Nessie, JDBC, and more |
| **Delete file caching** | Cross-task caching via CachingDeleteFileLoader + DeleteFilter | Worker-level caching of delete files |

**Maturity Assessment:**
- **Read path:** iceberg-rust is quite mature for reads. The multi-level pushdown pipeline is complete. Delete file handling (both pos and eq) is production-ready. The 0.8.0 release (Jan 2026) included major fixes for delete handling correctness.
- **Write path:** Basic append support exists but lacks MERGE, UPDATE, DELETE, compaction.
- **V3 features:** Deletion vectors from Puffin files are not yet implemented (marked TODO in delete_file_index.rs).
- **Distributed execution:** No built-in mechanism for distributing FileScanTasks across workers. This is the biggest gap for a distributed query engine.

---

## 5. Implications for Rust Worker Design

### 5.1 Reuse the Iceberg Core Crate Directly
The `iceberg` crate (not the DataFusion integration) provides everything needed: `TableScanBuilder`, `ArrowReader`, `CachingDeleteFileLoader`. A Rust worker should:
1. Use `Table::scan()` to build a `TableScan`
2. Call `plan_files()` to get `FileScanTaskStream`
3. Feed tasks to `ArrowReader::read()` for parallel file processing

The DataFusion integration layer can be bypassed entirely if the worker has its own execution framework.

### 5.2 FileScanTask as the Unit of Distribution
`FileScanTask` is serializable (`Serialize`/`Deserialize` via serde). A coordinator could:
1. Build the `TableScan` and call `plan_files()` to enumerate tasks
2. Serialize `FileScanTask`s and distribute to workers
3. Workers deserialize and use `ArrowReader` to process their assigned tasks

However, the current `FileScanTask` has some fields marked with `serialize_not_implemented` (partition, partition_spec, name_mapping), which would need to be addressed for distributed serialization.

### 5.3 Delete File Handling is Self-Contained
The `CachingDeleteFileLoader` handles all delete file loading and caching. In a distributed setting, the main consideration is that delete files may be shared across data files in the same partition. If tasks for the same partition are scheduled to the same worker, the caching benefits naturally. If spread across workers, each worker independently loads the same delete files (acceptable but not optimal).

### 5.4 Predicate Pushdown Pipeline is Reusable
The multi-level evaluators (`ManifestEvaluator`, `ExpressionEvaluator`, `InclusiveMetricsEvaluator`, `RowGroupMetricsEvaluator`, `PageIndexEvaluator`) all work with Iceberg's `BoundPredicate` type. A custom Rust worker can build `BoundPredicate` directly without going through DataFusion's `Expr` translation layer, gaining access to more predicate types.

### 5.5 Missing: Distributed Split Generation
The biggest gap for a distributed Rust worker is split generation. Trino's coordinator generates splits from manifests and distributes them. The iceberg-rust scan planner produces a stream of `FileScanTask`s, but there's no built-in mechanism to partition this stream across workers by data locality, partition affinity, or load balancing.

### 5.6 Missing: Deletion Vectors (V3)
For Iceberg V3 tables using deletion vectors (stored in Puffin files), the current iceberg-rust implementation has a TODO. This would need to be contributed upstream or implemented in the worker as an extension.

### 5.7 Schema Evolution Handled Transparently
The `RecordBatchTransformer` and `ArrowReader`'s projection mask logic handle all schema evolution (column reordering, type promotion, missing columns, name mapping) transparently. A Rust worker gets correct results without any additional schema handling code.
