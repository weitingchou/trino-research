# Phase 4: Communication Interfaces & Data Exchange Overview (DataFusion)

To build a worker or understand DataFusion's networking and storage boundaries, we must map out its three interface planes: Storage (Connector), Local Data (Intra-node Shuffle), and Distributed Network (Arrow Flight / Ballista).

Unlike Trino, DataFusion core is an embeddable engine. Its "Control Plane" and "Distributed Data Plane" are often delegated to external schedulers like Apache/DataFusion-Ballista.

## 1. The Storage Plane (Core <-> `object_store` / Connectors)

This is DataFusion's equivalent to Trino's SPI. It defines how the engine interacts with external storage like S3, Parquet files, or custom databases.

### The TableProvider Contract

* **Two-Trait Split:** `TableSource` (planning-time, in `datafusion-expr`) vs `TableProvider` (execution-time, in `datafusion-catalog`). `DefaultTableSource` bridges them. This lets projects use DataFusion's optimizer without the execution engine.
* **`scan()` Method:** Receives `projection` (column indices), `filters` (pushed-down predicates as `Expr`s), and `limit` (row count hint), returning an `Arc<dyn ExecutionPlan>`. Unlike Trino's `ConnectorPageSource` which is a simple single-reader, `scan()` returns a complete plan subtree — the provider controls its own parallelism.
* **`scan_with_args()` (new API):** Extensible replacement using `ScanArgs` struct. `ListingTable` overrides this directly; legacy `scan()` delegates to it.
* **Filter Pushdown Negotiation:** `supports_filters_pushdown()` returns per-filter `Exact` (trusted, filter removed from plan), `Inexact` (pushed to scan AND retained above as safety check — critically prevents LIMIT pushdown), or `Unsupported`. Compared to Trino's `applyFilter()` which returns a new `TableHandle` with baked-in constraints.
* **Concrete Implementations:** `ListingTable` (file-based, discovers files via `list_files_for_scan()`, builds `FileScanConfig` -> `DataSourceExec`), `MemTable` (in-memory with full DML), `ViewTable` (re-optimizes its LogicalPlan at scan time — full optimizer pass per query), `StreamingTable` (unbounded streaming).

### The Write Path

* **`DataSink` trait:** `write_all()` receives a `SendableRecordBatchStream` and writes to storage. `DataSinkExec` enforces single-partition input (coalesce required) and drives the sink.
* **`INSERT INTO` flow:** `LogicalPlan::Dml(WriteOp::Insert)` -> physical planner downcasts `DefaultTableSource` -> calls `TableProvider::insert_into()` -> returns `DataSinkExec` wrapping a `DataSink`.
* **`TableProviderFactory`:** Enables `CREATE EXTERNAL TABLE` to dynamically create table providers at runtime.

### The Parquet Scan Pipeline

The Parquet scan is the most complex storage path, orchestrated by a **10-state explicit state machine** (`ParquetOpenState`) that interleaves CPU-only pruning with async I/O:

* **Progressive Access Plan Refinement:** A `ParquetAccessPlan` starts as "scan all" and is narrowed through 6 stages:
  1. User-provided external index -> initial plan
  2. `prune_by_range` -> skip row groups outside file range
  3. `prune_by_statistics` -> skip by min/max (via `PruningPredicate`)
  4. `prune_by_bloom_filters` -> skip by bloom filter
  5. `prune_by_limit` -> skip partially-matched groups when fully-matched groups suffice (uses inverted predicate `NOT(pred)` to detect full matches)
  6. `prune_plan_with_page_index` -> add `RowSelection` within row groups

* **Row-Level Filtering:** `build_row_filter` splits conjuncts, sorts by `required_bytes` cost (cheap columns first = late materialization), and compiles to `DatafusionArrowPredicate`s. The arrow-rs decoder applies these during decode, only materializing expensive columns for surviving rows.

* **Push-Based Decoder:** `ParquetPushDecoder` returns `NeedsData(ranges)` for demand-driven I/O — only the exact byte ranges needed for the current batch are fetched from `object_store`.

* **Partition Column Substitution:** Hive-style partition columns are substituted as literals into predicates at file-open time (e.g., `region = 'us-west-2'` on a file from `region=us-east-1` becomes `FALSE`), enabling file-level pruning before metadata is loaded.

* **Dynamic Filter Early Stopping:** `EarlyStoppingStream` checks a `FilePruner` after every batch, enabling mid-file termination when dynamic filters (TopK, hash join) prove no more rows can match.

* **No Default Metadata Caching:** `DefaultParquetFileReaderFactory` reads the footer on every file open. `CachedParquetFileReaderFactory` exists but must be explicitly configured — caching policy is left to the integrator.

## 2. The Local Data Plane (Intra-Node Shuffle)

*Fully covered in Phase 2, Task 2.4.B (`23_datafusion_48_2.4.B_local_repartitioning.md`).*

* **`RepartitionExec`:** In-memory exchange using custom `DistributionSender`/`DistributionReceiver` pairs backed by a shared `Gate` for backpressure (not Tokio's standard MPSC channels).
* Hash/round-robin routing, spill-to-disk on memory pressure, preserve-order mode (N x M channels + `StreamingMerge`), cooperative yielding.

## 3. The Distributed Data Plane (Over-The-Wire via Arrow Flight)

When DataFusion is scaled out across multiple machines (e.g., using Ballista), it uses **Apache Arrow Flight** rather than a custom HTTP protocol.

### Encoding Pipeline (FlightDataEncoder)

* Takes a `Stream<Result<RecordBatch>>`, splits large batches to fit within a **2MB default target** (via zero-copy `RecordBatch::slice`), serializes via Arrow IPC (`IpcDataGenerator::encode`), and queues `FlightData` messages in a `VecDeque`.
* **First message:** Contains the schema as Arrow IPC `Schema` message.
* **Dictionary handling:** Two strategies — `Hydrate` (default, replaces dictionaries with underlying types) and `Resend` (sends `DictionaryBatch` messages, uses `ArrayData::ptr_eq` to skip unchanged dictionaries — common when batches share the same string interning pool).
* **64-byte alignment** (default) in IPC buffers enables AVX-512 SIMD operations without copying to aligned memory.

### Decoding Pipeline (FlightDataDecoder)

* **Confirmed zero-copy chain:** (1) prost deserializes `data_body` as `bytes::Bytes` (reference-counted), (2) `Buffer::from(Bytes)` wraps in `Arc` without copying, (3) `slice_with_length` extracts per-column buffers via pointer arithmetic + refcount increment. All column buffers in a decoded `RecordBatch` are slices of the same underlying network allocation.
* **Compression breaks zero-copy:** When IPC compression (LZ4/ZSTD) is enabled, decompression produces a new allocation.

### Flight vs. Trino Exchange

| Dimension | Arrow Flight | Trino Exchange |
|---|---|---|
| **Transport** | gRPC/HTTP2 streaming | Custom HTTP/1.1 chunk pulling |
| **Serialization** | Columnar Arrow IPC (zero-copy) | Row-scoped Pages with per-block compression |
| **Flow control** | HTTP/2 `WINDOW_UPDATE` frames (transport-level) | Application-managed buffer capacity gating |
| **Fault tolerance** | None built-in | Token-based idempotent retry |
| **Key RPC methods** | `do_get`, `do_put`, `do_exchange` | `GET /v1/task/{taskId}/results/{bufferId}/{token}` |

## 4. The Control Plane (Plan Serialization via Protobuf)

In a single-node setup, the Control Plane is simply the `SessionContext` in memory. In a distributed setup, the entire `ExecutionPlan` DAG is serialized for transmission.

### Serialization Architecture

* **`AsExecutionPlan` trait:** `try_encode()` and `try_decode()` convert between `Arc<dyn ExecutionPlan>` and protobuf bytes. The tree is the unit of serialization — `PhysicalPlanNode` is recursive with children as `Box<PhysicalPlanNode>`.
* **Downcast-chain pattern:** Serialization uses a hardcoded chain of ~25 sequential `downcast_ref` calls (not visitor/registration). Each new built-in operator requires a source-code change. The `PhysicalExtensionCodec` trait provides an escape hatch for custom operators.
* **Three extension points:** `PhysicalExtensionCodec` (what to serialize), `PhysicalProtoConverterExtension` (how to serialize — e.g., deduplication), `ComposedPhysicalExtensionCodec` (combining multiple codecs).

### Runtime State Boundary

* **Excluded from serialization:** `HashTableLookupExpr` (replaced with `lit(true)`), `CachedParquetFileReaderFactory` (rebuilt from `TaskContext` on remote), `DynamicFilterPhysicalExpr` (snapshotted via `snapshot_physical_expr()` before serialization).
* **Protobuf contains plan structure + configuration; `TaskContext` provides the runtime environment** on the receiving side.

### Plan Serialization vs. Trino

| Dimension | DataFusion (Protobuf) | Trino (REST/JSON) |
|---|---|---|
| **Wire format** | Protocol Buffers (binary, compact) | JSON over HTTP REST |
| **Operator identification** | Protobuf `oneof` discriminant (static) | Java class names in `@type` annotations (dynamic) |
| **Plan payload** | Plan structure only | `TaskUpdateRequest` with session, splits, buffer config, AND plan |
| **Custom operators** | `PhysicalExtensionCodec` with opaque bytes | Java SPI + Jackson polymorphic deserialization |
| **Transport** | None built-in (caller's choice — Ballista uses gRPC) | Built-in HTTP POST to `/v1/task/{taskId}` |

## Summary: Connecting the Dots

1. **`TableProvider::scan()`** returns a full `ExecutionPlan` subtree — the provider controls parallelism, unlike Trino where the engine manages splits. Filter pushdown is negotiated per-filter as `Exact`/`Inexact`/`Unsupported`.
2. **The Parquet scan pipeline** uses a 10-state machine with progressive pruning (statistics -> bloom filters -> limit -> page index -> row-level filter), push-based demand-driven I/O, and dynamic filter early stopping.
3. **Local data exchange** (RepartitionExec) uses custom Gate-based channels with backpressure, hash/round-robin routing, and spill-to-disk. *(See Phase 2, Task 2.4.B for full details.)*
4. **Arrow Flight** provides zero-copy network transfer via gRPC/HTTP2 streaming with Arrow IPC format. The `FlightDataEncoder` splits batches to ~2MB targets; the `FlightDataDecoder` wraps network bytes directly into `Buffer` references without copying.
5. **Plan serialization** uses protobuf with a recursive `PhysicalPlanNode` structure. Runtime state is excluded — the receiving side reconstructs it from `TaskContext`. Three extension points allow custom operators, conversion strategies, and codec composition.
