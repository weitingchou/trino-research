# Implementation Plan: Rust-Based Trino Worker

## Table of Contents

- [Guiding Principles](#guiding-principles)
- [Crate Architecture](#crate-architecture)
- [Phased Roadmap](#phased-roadmap)
  - [Phase 0: Project Scaffold](#phase-0-project-scaffold)
  - [Phase 1: Data Model & Wire Format](#phase-1-data-model--wire-format)
  - [Phase 2: Control Plane (REST API)](#phase-2-control-plane-rest-api)
  - [Phase 3: Plan Fragment Parsing](#phase-3-plan-fragment-parsing)
  - [Phase 4: Execution Engine Core](#phase-4-execution-engine-core)
  - [Phase 5: Core Operators & Storage Plane](#phase-5-core-operators--storage-plane)
  - [Phase 6: Data Plane (Shuffle)](#phase-6-data-plane-shuffle)
  - [Phase 7: Memory Management & Arbitration](#phase-7-memory-management--arbitration)
  - [Phase 8: Advanced Operators](#phase-8-advanced-operators)
  - [Phase 9: Integration & Hardening](#phase-9-integration--hardening)
- [Knowledge Gaps & Required Research](#knowledge-gaps--required-research)
- [Risk Register](#risk-register)

---

## Guiding Principles

These principles derive from `GOAL.md` Section 3 ("Best of Breed") and Section 4 ("Decision-Making Framework"):

1. **Protocol fidelity over internal fidelity.** The coordinator must not know or care that the worker is Rust. Every REST endpoint, JSON schema, wire byte, and HTTP header must be bit-compatible with Java Trino 480. Internal architecture is unconstrained.
2. **Async-native execution.** Replace Trino's custom `Driver` yield loop and `ListenableFuture` blocking with Tokio async streams (`poll_next`). The DataFusion model — where each operator is an async `Stream<RecordBatch>` and Tokio's work-stealing runtime replaces the MLFQ scheduler — is the starting point.
3. **RAII memory safety.** Every memory reservation is a Rust struct whose `Drop` implementation returns bytes to the pool. No manual `free()` calls. No accounting leaks. (DataFusion pattern.)
4. **Hierarchical memory governance.** A process-wide memory arbitrator (Velox pattern) coordinates cross-query spill and abort. Individual operator reservations (DataFusion pattern) feed into a query-level tracking tree (Trino pattern).
5. **Arrow as the internal data model, Trino as the external data model.** Computation uses Arrow `RecordBatch`/`ArrayRef`. Trino's `Page`/`Block` wire format is a serialization concern at the network boundary only.
6. **No guessing.** When a design decision depends on undocumented Trino behavior, a new research task is created and executed before the decision is made (GOAL.md Section 4.1).

---

## Crate Architecture

```
trino-rust-worker/
+-- Cargo.toml                  (workspace root)
+-- GOAL.md
+-- IMPLEMENTATION_PLAN.md
+-- crates/
|   +-- server/             HTTP server, REST endpoints (axum)
|   +-- protocol/           Trino wire format, JSON serde, plan node types
|   +-- execution/          Task lifecycle, stream orchestration, scheduling
|   +-- operators/          Operator implementations (scan, filter, join, etc.)
|   +-- memory/             MemoryPool, MemoryReservation, Arbitrator
|   +-- connectors/         Connector trait, Hive/Iceberg Parquet reader
|   +-- exchange/           Output buffers, exchange client, shuffle
|   +-- common/             Shared types, errors, config, metrics
```

### Dependency DAG

```
server
  +-- protocol
  +-- execution
  +-- exchange

execution
  +-- protocol
  +-- operators
  +-- memory
  +-- common

operators
  +-- memory
  +-- connectors
  +-- common
  +-- arrow / datafusion-physical-expr  (external)

exchange
  +-- protocol
  +-- memory
  +-- common

connectors
  +-- memory
  +-- common
  +-- object_store / parquet  (external)

memory
  +-- common
```

### Key External Dependencies

| Crate | Purpose | Replaces (from Trino Java) |
|-------|---------|---------------------------|
| `arrow` / `arrow-array` | Internal columnar data model | `Block`, `Page`, `Slice` |
| `datafusion-physical-expr` | Expression evaluation (PhysicalExpr, compute kernels) | `PageProcessor`, bytecode-compiled expressions |
| `parquet` (arrow-rs) | Parquet reader with async I/O + predicate pushdown | `ParquetReader`, `PageReader`, DWIO |
| `object_store` | S3/GCS/local file abstraction | Airlift `S3FileSystem` |
| `tokio` | Async runtime, task scheduling, mpsc channels | `TimeSharingTaskExecutor`, `MultilevelSplitQueue`, `PrioritizedSplitRunner` |
| `axum` | HTTP framework for REST endpoints | Airlift HTTP + JAX-RS (`TaskResource`) |
| `serde` / `serde_json` | JSON serialization for control plane | Jackson |
| `bytes` | Zero-copy byte buffer management | `io.airlift.slice.Slice` |
| `lz4_flex` / `zstd` | Block-level compression for wire format | `LZ4Compressor` / `ZstdCompressor` |

---

## Phased Roadmap

### Phase 0: Project Scaffold

**Objective:** Bootable Rust workspace with CI, linting, and the crate skeleton compiled.

**Tasks:**
- [ ] Initialize Cargo workspace with all 8 crates (empty `lib.rs` per crate)
- [ ] Configure CI (cargo check, clippy, fmt, test)
- [ ] Add core dependencies (arrow, tokio, axum, serde, bytes, parquet, object_store)
- [ ] Create a `server` binary that starts an axum HTTP server on a configurable port
- [ ] Implement `GET /v1/info` (returns static server info JSON) as smoke test

**Milestone:** `cargo build` succeeds; `curl localhost:8080/v1/info` returns a JSON response.

---

### Phase 1: Data Model & Wire Format

**Objective:** Define the internal data representation and implement Trino's binary serialization so the worker can produce bytes that the coordinator (and Java workers) can consume.

**Why first:** Everything downstream — operators, exchange, connectors — produces or consumes data. The data model is the foundation.

#### 1A. Internal Data Model (`common`)

Arrow `RecordBatch` is the internal unit of computation. No custom columnar types.

| Trino Concept | Rust Equivalent | Notes |
|---------------|-----------------|-------|
| `Page` | `RecordBatch` | Same role: columnar batch with shared row count |
| `Block` (ValueBlock) | `ArrayRef` (`Arc<dyn Array>`) | Primitive, String, Binary, etc. |
| `DictionaryBlock` | `DictionaryArray` | Arrow-native equivalent |
| `RunLengthEncodedBlock` | `RunEndEncodedArray` or expand on read | Evaluate perf trade-off |
| `Slice` | `Buffer` / `Bytes` | Arrow `Buffer` for aligned memory, `bytes::Bytes` for I/O |
| `positionCount` | `RecordBatch::num_rows()` | Identical semantics |

**Decision rationale:** Arrow arrays are the industry standard for Rust columnar compute. DataFusion, Polars, and the entire arrow-rs ecosystem operate on them natively. Building custom types would duplicate effort and forfeit access to hundreds of optimized compute kernels. Trino's `Block` types map cleanly to Arrow array types. The wire format layer (Phase 1B) handles the translation at the boundary.

**SIMD alignment:** Arrow's 64-byte memory alignment (cache-line aligned) is the foundation for SIMD acceleration. All `Buffer` allocations go through aligned allocators, enabling LLVM auto-vectorization of tight compute loops without explicit intrinsics. This alignment must be preserved throughout the pipeline — no unaligned copies between the wire format boundary and the compute layer.

**SIMD strategy (resolved from KG-14):** The three engines use fundamentally different approaches:

* **Trino:** Three-tier hybrid — (1) explicit Java Vector API for exchange serde compress/expand (AVX-512 `VPCOMPRESS`/`VPEXPAND`) and Parquet bit-unpacking, (2) SWAR 8-way byte matching in `FlatHash` for hash probing, (3) JVM auto-vectorization for compute (branchless selection loops, flat `boolean[]` null representation).
* **DataFusion:** Pure auto-vectorization — zero explicit SIMD in either DataFusion or arrow-rs (v58.1.0). Relies on Arrow's 64-byte alignment + LTO (`lto = true`, `codegen-units = 1`) + `-C target-cpu=native` for LLVM to emit AVX2/AVX-512 instructions. Aggregation loops use `unsafe { get_unchecked_mut }` and 64-bit chunk null processing.
* **Velox:** Aggressive explicit SIMD — xsimd 10.0.0 abstraction layer (~1,950 lines) + raw intrinsics. Swiss-table tag probing with 4-way interleaved prefetch, SIMD filter evaluation (`testValues(xsimd::batch<T>)`), SIMD bloom filter, SIMD substring search, CRC32 hardware acceleration. Compile-time dispatch only (no runtime CPUID). **Intentionally avoids AVX-512** due to frequency throttling.

**Our approach — hybrid auto-vectorization + targeted explicit SIMD:**

| Subsystem | Strategy | Rationale |
|-----------|----------|-----------|
| Arithmetic, comparison, projection | LLVM auto-vectorization (Arrow kernels) | DataFusion pattern — works well for simple loops over contiguous slices |
| Hash table probing | `hashbrown` native SIMD (SSE2/NEON) | Already Swiss-table with 16-way tag matching; consider 4-way interleaved probing from Velox |
| Wire format compress/expand | `std::arch` AVX-512 intrinsics or Arrow `filter` kernel | Trino pattern — hardware compress/expand is critical for null-heavy serde |
| Null bit packing (MSB↔LSB) | Arrow bitmap utilities + explicit SIMD for bulk conversion | Bit-reversal at boundary is a fixed cost; SIMD packing amortizes it |
| Parquet bit-unpacking | Arrow Parquet reader (auto-vectorized) or `bitpacking` crate | Arrow reader already handles this; explicit SIMD only if profiling shows need |
| Partition hash computation | `xxhash-rust` (scalar compatibility) + batch SIMD hashing for bulk ops | Must be bit-compatible with Trino's xxHash64 mix |

Build configuration: always compile with `lto = true`, `codegen-units = 1`, `-C target-cpu=native` for release. Use `#[cfg(target_feature = "avx2")]` for compile-time dispatch of explicit SIMD paths, with scalar fallbacks for portability.

#### 1B. Trino Wire Format (`protocol`)

Implement serialization/deserialization of Trino's binary page format so data can flow between Java and Rust workers.

**Known from research (Trino Phase 4, Tasks 4.2.A, 4.2.D [KG-1], 4.2.E [KG-2]):**

The page binary format is a four-layer nesting:

```
HTTP Response Envelope (16 bytes)
  magic: 0xfea4f001 (4B) + xxHash64 checksum (8B) + pageCount (4B)
  |
  +-- Page (12-byte header per page)
       positionCount (4B) + uncompressedSize (4B) + compressedSize (4B)
       |
       +-- channelCount (4B)
       +-- Block × channelCount (self-describing)
            encoding name: length-prefixed UTF-8 (e.g., "LONG_ARRAY")
            block payload: type-specific binary data
```

**Block encoding types (13):** `LONG_ARRAY`, `INT_ARRAY`, `SHORT_ARRAY`, `BYTE_ARRAY`, `INT128_ARRAY`, `FIXED12`, `VARIABLE_WIDTH`, `DICTIONARY`, `RUN_LENGTH`, `ROW`, `ARRAY`, `MAP`, `VARIANT`.

**Critical format details:**
- **Null bitmaps** are bit-packed, **MSB-first** (position 0 → bit 7 of byte 0). This differs from Apache Arrow's LSB-first convention — requires a bit-reversal step during Arrow↔Trino conversion.
- **Null compaction:** Primitive blocks write only non-null values, preceded by a `nonNullCount` int32. This is the primary wire-format size optimization.
- **All multi-byte integers are little-endian.**
- **Compression** is applied to **64 KB chunks** of the raw page payload (not per-block). Each chunk has a 4-byte marker (MSB = isCompressed, lower 31 bits = size).
- **Encryption** (optional): per-chunk AES-256-CBC with random 16-byte IV prepended.
- No codec identifier in the wire format — both sides must agree via session properties.

**Implementation:**
- [ ] HTTP response envelope: magic validation, xxHash64 checksum, pageCount parsing
- [ ] Page header: 12-byte header + channelCount parsing
- [ ] Block deserializer: dispatch on encoding name string → type-specific reader
- [ ] `PageSerializer`: `RecordBatch` → Trino binary bytes (Arrow LSB→Trino MSB null bitmap conversion)
- [ ] `PageDeserializer`: Trino binary bytes → `RecordBatch` (Trino MSB→Arrow LSB null bitmap conversion)
- [ ] Null compaction: expand non-null-only values to full array with null slots during deserialization
- [ ] LZ4/ZSTD 64KB chunk compression/decompression
- [ ] Round-trip property tests: generate random RecordBatches, serialize, deserialize, assert equality
- [ ] Cross-language integration test against Java `PagesSerde`

**Milestone:** A Rust binary can serialize a `RecordBatch` into bytes that a Java Trino worker can deserialize, and vice versa. Validated via a cross-language integration test.

---

### Phase 2: Control Plane (REST API)

**Objective:** Implement the REST endpoints so the Trino coordinator can create tasks, poll status, and manage the worker lifecycle.

**Known from research (Trino Phase 4, Tasks 4.1.A-C):**
- `POST /v1/task/{taskId}` — create/update task (accepts `TaskUpdateRequest` JSON)
- `GET /v1/task/{taskId}/status` — long-polling with version-based change detection
- `DELETE /v1/task/{taskId}` — abort task
- `GET /v1/task/{taskId}/results/{bufferId}/{token}` — data pull (Phase 6)
- `GET /v1/memory` — returns `MemoryInfo` for coordinator aggregation
- `GET /v1/info` — server info (done in Phase 0)
- Status uses randomized wait to prevent thundering herd on restart

#### 2A. Service Discovery & Worker Registration

**Known from research (Task 4.1.D [KG-3]):**

Workers register via the **ANNOUNCE** protocol (default of three pluggable backends: `ANNOUNCE`, `DNS`, `AIRLIFT_DISCOVERY`):
- Worker POSTs its URI to the coordinator's `/v1/announce` endpoint every **5 seconds**
- Coordinator maintains a **30-second TTL cache** — if a worker misses ~6 consecutive announcements, it drops out
- Coordinator independently polls each worker's `GET /v1/info` every 5 seconds, verifying environment name, Trino version, and node lifecycle state (`ACTIVE`, `DRAINING`, `DRAINED`, `SHUTTING_DOWN`). Mismatched workers are marked `INVALID` and excluded from scheduling.
- **Workers are topology-blind:** `WorkerInternalNodeManager` is a stub — workers only know their own identity and the coordinator URI.

**Implementation:**
- [ ] Background Tokio task: POST worker URI to coordinator `/v1/announce` every 5 seconds
- [ ] `GET /v1/info` endpoint returning `ServerInfo` JSON (environment, version, node state, uptime)
- [ ] Graceful shutdown: transition node state through `ACTIVE` → `DRAINING` → `DRAINED` → `SHUTTING_DOWN`
- [ ] Configurable coordinator URI, worker bind address, environment name

#### 2B. Task Lifecycle Endpoints (`server`)

- [ ] `POST /v1/task/{taskId}` — accept `TaskUpdateRequest`, delegate to `TaskManager`
- [ ] `GET /v1/task/{taskId}/status` — long-poll with version tracking (tokio `watch` channel)
- [ ] `DELETE /v1/task/{taskId}` — trigger abort/cancel state transition
- [ ] `GET /v1/memory` — return `MemoryInfo` JSON snapshot

#### 2C. Task State Machine (`execution`)

Implement the state machine from research (Trino Phase 2, Task 2.1.B):

```
RUNNING -> FLUSHING -> FINISHED
   |
   +-> CANCELING -> CANCELED
   +-> ABORTING  -> ABORTED
   +-> FAILING   -> FAILED
```

Two-phase termination: intermediate states wait for all active streams to complete before transitioning to terminal state. Use Tokio `CancellationToken` for propagation (analogous to Trino's `DriverAndTaskTerminationTracker`).

#### 2D. Task Status Reporting

- [ ] Version-monotonic `TaskStatus` struct (version bumps on any state change)
- [ ] Long-poll via `tokio::sync::watch` — coordinator holds connection until version advances or timeout
- [ ] Include memory usage, output buffer status, pipeline stats

**Known from research (Tasks 4.1.E [KG-4], 4.1.H [KG-5]):**

**TaskUpdateRequest (10 fields):** A single idempotent message for both create and update. `fragment` (PlanFragment) is sent only on the first update; the coordinator uses `needsPlan` feedback to stop re-sending. `session` (SessionRepresentation, 29 fields including `queryId`, `transactionId`, `user`, `catalog`, `schema`, `systemProperties`, `catalogProperties`) travels with every request. `sources` (split assignments) are incremental, with `sequenceId` for idempotent delivery.

**TaskStatus (22 fields):** Lightweight heartbeat record polled at ~5s cadence. Version-monotonic — `Long.MAX_VALUE` on terminal states forces immediate coordinator notification. Custom serialization: **Duration** as airlift format `"15.50ms"`, `"2.00s"` (not ISO-8601); **DataSize** as human-readable `"3.50MB"`, `"12B"`.

**TaskInfo:** Full response (superset of TaskStatus) with nested `TaskStats` (44 fields) → `PipelineStats` (39 fields) → `OperatorStats` (40 fields) → `DriverStats` (27 fields). Hierarchical `summarize()` strips detail levels for bandwidth control.

**Implementation notes:**
- [ ] `TaskUpdateRequest` serde struct: 10 fields with `Option<PlanFragment>` for conditional fragment
- [ ] `SessionRepresentation` serde struct: 29 fields including properties maps
- [ ] `TaskStatus` serde struct: 22 fields with custom Duration/DataSize serializers
- [ ] `TaskInfo` with nested stats hierarchy and summarization levels
- [ ] `needsPlan` flag in task status to signal coordinator to skip re-sending plan

**Milestone:** A real Trino coordinator can send a `TaskUpdateRequest` to the Rust worker, poll status, and see the task transition through `RUNNING -> FINISHED`. (The task doesn't execute anything yet — it just accepts and acks.)

---

### Phase 3: Plan Fragment Parsing

**Objective:** Deserialize the physical plan JSON from the coordinator into an internal plan tree that the execution engine can compile.

**Known from research (Trino Phase 2, Task 2.1.A; Phase 3, Task 3.1.C):**
- Coordinator sends a `PlanFragment` as JSON inside `TaskUpdateRequest`
- The fragment contains a tree of plan nodes (TableScan, Filter, Project, HashJoin, Aggregation, Exchange, Output, etc.)
- Each plan node has typed properties (column handles, partition functions, expressions, etc.)
- 47 plan node types in Trino 480, polymorphically tagged via `@type` discriminator

#### 3A. Plan Node Type System (`protocol`)

**Known from research (Tasks 4.1.F [KG-6], 3.3.B [KG-7]):**

**PlanFragment (12 fields):** Top-level envelope with `id`, `root` (recursive plan node tree), `symbols` (custom key encoding: `"<len>|<type>|<name>"`, e.g., `"6|bigint|col1"`), `partitioning` (`PartitioningHandle`), `outputPartitioningScheme`, `partitionedSources`, `activeCatalogs`.

**47 PlanNode types** are polymorphically tagged via `@JsonTypeInfo` with an `"@type"` discriminator field (e.g., `"tableScan"`, `"join"`, `"filter"`, `"aggregation"`, `"exchange"`). Nodes reference children as nested JSON objects.

**Connector handles** (table, column, split) use `AbstractTypedJacksonModule` with `@type` values in `"classLoaderId:fullyQualifiedClassName"` format (e.g., `"hive:io.trino.plugin.hive.HiveTableHandle"`).

**Expressions are JSON AST (not bytecode).** The coordinator sends a sealed hierarchy of **18 `Expression` subtypes** (`Reference`, `Constant`, `Call`, `Comparison`, `Cast`, `Logical`, `Case`, `Switch`, `Between`, `In`, `IsNull`, `Coalesce`, `NullIf`, `Array`, `Row`, `Lambda`, `Bind`, `FieldReference`), all as Java records with `@type` discriminator. Key details:
- **Constants** are serialized as **base64-encoded Trino Block binary** (not JSON primitives). A BIGINT `10` becomes a base64-encoded `LongArrayBlock`.
- **Function references** are fully resolved: each `Call` carries a `ResolvedFunction` with `FunctionId`, `BoundSignature`, `CatalogHandle`, and behavioral flags. Workers never perform function lookup.
- **Types** serialize as flat SQL-parseable strings (`"bigint"`, `"varchar(256)"`, `"row(x bigint, y integer)"`).

**Implementation:**
- [ ] `PlanFragment` serde struct with 12 fields
- [ ] `PlanNode` enum with 47 variants, tagged by `#[serde(tag = "@type")]`
- [ ] Custom Symbol map key deserializer: parse `"<len>|<type>|<name>"` format
- [ ] `Expression` enum with 18 variants, tagged by `#[serde(tag = "@type")]`
- [ ] `Constant` deserializer: base64-decode → Block binary parse → extract native value
- [ ] `ResolvedFunction` struct with `FunctionId`, `BoundSignature`, `CatalogHandle`, nullability
- [ ] Connector handle deserializer: parse `"classLoaderId:FQCN"` format for table/column/split handles
- [ ] Property tests: capture real plan fragment JSON from Trino coordinator, assert round-trip fidelity

#### 3B. Plan-to-Execution Compilation (`execution`)

Convert the parsed plan tree into executable async stream pipelines (DataFusion-style):

```
PlanFragment JSON
  -> PlanNode tree (protocol, Phase 3A)
  -> Pipeline of Arc<dyn ExecutionPlan> (execution, Phase 3B)
  -> SendableRecordBatchStream per partition (Phase 4)
```

Each Trino plan node maps to one or more `ExecutionPlan` nodes:

| Trino PlanNode | Rust Execution Plan | Source |
|----------------|---------------------|--------|
| `TableScanNode` | `ConnectorScanExec` | Custom (wraps connector) |
| `FilterNode` | `FilterExec` | DataFusion |
| `ProjectNode` | `ProjectionExec` | DataFusion |
| `HashJoinNode` | `HashJoinExec` | DataFusion (adapted) |
| `AggregationNode` | `AggregateExec` | DataFusion (adapted) |
| `ExchangeNode` | `ExchangeSourceExec` | Custom (Trino exchange protocol) |
| `OutputNode` | `OutputBufferSinkExec` | Custom (feeds output buffer) |
| `TopNNode` | `TopKExec` | DataFusion |
| `SortNode` | `SortExec` | DataFusion |
| `LimitNode` | `GlobalLimitExec` / `LocalLimitExec` | DataFusion |

**Known from research (Task 4.1.G [KG-8]):**

Types serialize as flat SQL-parseable strings via `@JsonValue` on `Type.getTypeId()`. Deserialization uses `TypeDeserializer` which calls `typeManager.getType(TypeId.of(value))` — essentially a SQL type signature parser.

| Trino Type String | Arrow DataType |
|-------------------|---------------|
| `"boolean"` | `DataType::Boolean` |
| `"tinyint"` | `DataType::Int8` |
| `"smallint"` | `DataType::Int16` |
| `"integer"` | `DataType::Int32` |
| `"bigint"` | `DataType::Int64` |
| `"real"` | `DataType::Float32` |
| `"double"` | `DataType::Float64` |
| `"decimal(p,s)"` | `DataType::Decimal128(p, s)` |
| `"varchar"` / `"varchar(n)"` | `DataType::Utf8` |
| `"varbinary"` | `DataType::Binary` |
| `"date"` | `DataType::Date32` |
| `"timestamp(p)"` | `DataType::Timestamp(Microsecond, None)` |
| `"timestamp(p) with time zone"` | `DataType::Timestamp(Microsecond, Some(tz))` |
| `"array(T)"` | `DataType::List(T)` |
| `"map(K,V)"` | `DataType::Map(K, V)` |
| `"row(f1 T1, f2 T2)"` | `DataType::Struct(fields)` |

- [ ] Type string parser: parse Trino type signatures into Arrow `DataType`
- [ ] Expression translator: Trino `Expression` AST → DataFusion `PhysicalExpr` (map `Call` nodes to Arrow compute kernels via `FunctionId` registry)
- [ ] Function registry: map Trino `FunctionId` strings (e.g., `"$operator$add(bigint,bigint):bigint"`) to Arrow compute functions
- [ ] Partitioning strategy: map Trino's pipeline taxonomy (split-lifecycle, task-lifecycle, remote-source) to Tokio task spawning

**Milestone:** The Rust worker can receive a `TaskUpdateRequest` containing a simple `TableScan -> Filter -> Project -> Output` plan, parse it into an internal plan tree, and log the execution plan without actually executing it.

---

### Phase 4: Execution Engine Core

**Objective:** Build the async stream orchestration that drives operator execution.

**Design choice (from GOAL.md Section 3 — DataFusion model):**

Replace Trino's `Driver` + `TimeSharingTaskExecutor` + `MultilevelSplitQueue` with Tokio's work-stealing runtime + async `Stream` chains. Each partition becomes a Tokio task; each operator is a `Stream<Item = RecordBatch>`.

| Trino Concept | Rust Replacement | Rationale |
|---------------|------------------|-----------|
| `Driver` (cooperative yield loop) | Tokio task + `Stream::poll_next()` | Rust's `async/await` provides cooperative yielding natively — `.await` is the yield point |
| `TimeSharingTaskExecutor` (MLFQ) | `tokio::runtime` (work-stealing) | Tokio's work-stealing scheduler provides automatic load balancing across CPU cores |
| `PrioritizedSplitRunner` | Tokio task priority (future: custom scheduler) | Start with Tokio's default FIFO; add priority scheduling later if needed |
| `DriverYieldSignal` (1s quantum) | Natural `.await` yield points | Tokio tasks yield at every `.await`; no explicit quantum tracking needed |
| `ListenableFuture` (blocking) | `Poll::Pending` + waker | Rust's native async mechanism |
| `OperatorContext` | `TaskContext` (DataFusion) | Carries memory pool, config, metrics |

#### 4A. Task Execution Manager (`execution`)

- [ ] `TaskManager`: maps task IDs to running tasks, handles create/update/cancel
- [ ] `TaskExecution`: owns the compiled plan, spawns Tokio tasks per partition
- [ ] `TaskContext`: per-partition context carrying `MemoryPool`, config, metrics
- [ ] Cancellation propagation via `tokio_util::sync::CancellationToken`

#### 4B. Stream Orchestration

- [ ] Each `ExecutionPlan::execute(partition, ctx)` returns a `SendableRecordBatchStream`
- [ ] Pipeline breakers (joins, aggregations) use internal state machines within `poll_next()` (DataFusion pattern)
- [ ] Split delivery: coordinator sends splits incrementally via `POST /v1/task/{taskId}`; splits are fed to source operators via `tokio::sync::mpsc` channel

#### 4C. Metrics & Observability

- [ ] Per-operator timing (wall time, CPU time, rows in/out, bytes processed)
- [ ] Per-task aggregate metrics (for `TaskStatus` reporting to coordinator)
- [ ] Pipeline-level stats matching Trino's `PipelineStats` JSON structure

**Known from research (Task 4.1.I [KG-9]):**

**OperatorStats (40 fields):** Includes `operatorId`, `planNodeId`, `operatorType`, timing fields (`addInputWall`, `getOutputWall`, `finishWall`), data fields (`rawInputDataSize`, `rawInputPositions`, `inputDataSize`, `inputPositions`, `outputDataSize`, `outputPositions`), memory fields (`userMemoryReservation`, `revocableMemoryReservation`, `peakUserMemoryReservation`), and spill fields. Polymorphic `OperatorInfo` (8 subtypes via `@type` discriminator).

**Metrics** use `@JsonTypeInfo(use = Id.CLASS)` — full Java class names as discriminators (e.g., `"io.trino.operator.CounterMetric"`). The Rust worker must serialize these with matching class names.

**Merging key:** `(pipelineId << 32 | operatorId)` for combining stats across drivers.

- [ ] `OperatorStats` struct with 40 fields and custom Duration/DataSize serializers
- [ ] `PipelineStats` struct (39 fields) aggregating operator stats
- [ ] `DriverStats` struct (27 fields)
- [ ] Metrics with Java class name discriminators in `@type` field
- [ ] Hierarchical `summarize()` that strips detail levels for bandwidth control

**Milestone:** The worker can receive a trivial plan (e.g., `VALUES (1, 'hello')`), execute it as a Tokio stream, and produce a `RecordBatch` result internally.

---

### Phase 5: Core Operators & Storage Plane

**Objective:** Implement the minimum operator set to run a real query: scan, filter, project, and the connector interface for reading Parquet from object storage.

#### 5A. Connector Trait (`connectors`)

```rust
#[async_trait]
pub trait Connector: Send + Sync {
    /// Parse a split JSON into a typed split descriptor.
    fn deserialize_split(&self, json: &Value) -> Result<Arc<dyn ConnectorSplit>>;

    /// Create a page source that reads data from the split.
    async fn create_page_source(
        &self,
        split: Arc<dyn ConnectorSplit>,
        columns: &[ColumnHandle],
        predicate: Option<&PhysicalExpr>,
        memory_reservation: MemoryReservation,
    ) -> Result<SendableRecordBatchStream>;
}
```

Design draws from all three engines:
- **Trino:** `ConnectorPageSource` interface, `ConnectorSplit` as opaque handle
- **DataFusion:** `TableProvider::scan()` returning async streams, `object_store` for I/O
- **Velox:** `DataSource` with `ScanSpec` tree for unified filter+projection pushdown

#### 5B. Hive/Iceberg Parquet Connector

**Known from research (Task 4.3.C [KG-10]):**

Splits are wrapped in a four-level nesting: `SplitAssignment` (plan node ID + splits + `noMoreSplits` flag) → `ScheduledSplit` (`sequenceId` for idempotent delivery) → `Split` (`CatalogHandle` string + polymorphic `ConnectorSplit`) → concrete split.

- **HiveSplit (17 fields):** file path, start offset, length, file size, partition keys, bucket number, storage format, S3 region. Addresses excluded via `@JsonIgnore`.
- **IcebergSplit (15 fields):** file path, offset, length, file format (PARQUET/ORC), delete files, partition data as pre-serialized JSON, `fileStatisticsDomain` as `TupleDomain`.
- **Polymorphic `@type`:** `"hive:io.trino.plugin.hive.HiveSplit"` (classloader ID + FQCN).

**Implementation:**
- [ ] `SplitAssignment` → `ScheduledSplit` → `Split` deserialization with 4-level nesting
- [ ] `HiveSplit` struct: 17 fields with file path, offset, length, partition keys
- [ ] `IcebergSplit` struct: 15 fields with file format, delete files, partition data
- [ ] Connector split dispatcher: route by `@type` prefix (e.g., `"hive:"` → Hive connector)
- [ ] Async Parquet reading via `parquet` crate + `object_store`
- [ ] Row-group pruning via min/max statistics
- [ ] Predicate pushdown to Parquet page index (where supported)
- [ ] Column projection pushdown (only read requested columns)
- [ ] Memory-aware: register allocations against `MemoryReservation`

#### 5C. Core Operators (`operators`)

| Operator | Implementation Strategy |
|----------|------------------------|
| `ScanFilterProjectExec` | Fused operator (Trino pattern): connector scan + filter + projection in one stream. Leverage `datafusion-physical-expr` for expression evaluation on Arrow arrays. |
| `FilterExec` | Standalone: evaluate `PhysicalExpr` -> `BooleanArray`, apply `arrow::compute::filter_record_batch`. |
| `ProjectionExec` | Standalone: evaluate list of `PhysicalExpr` -> new `RecordBatch` with projected columns. |
| `LimitExec` | Track row count, terminate stream after limit reached. |
| `TopNExec` | Heap-based top-N with spill support. |
| `OutputBufferSinkExec` | Terminal operator: serializes `RecordBatch` into Trino wire format and writes to output buffer (Phase 6). |
| `ExchangeSourceExec` | Source operator: pulls pages from upstream workers via HTTP (Phase 6). |

**Milestone:** The worker can execute `SELECT col1, col2 FROM hive.schema.table WHERE col1 > 10 LIMIT 100` against a Parquet file on S3, with the coordinator orchestrating the full lifecycle.

---

### Phase 6: Data Plane (Shuffle)

**Objective:** Implement output buffering and the exchange protocol so data can flow between workers (and back to the coordinator).

**Known from research (Trino Phase 4, Tasks 4.2.A-C):**
- Producer side: `PartitionedOutputBuffer` with per-partition `ClientBuffer`
- Consumer side: `DirectExchangeClient` + `HttpPageBufferClient` with token-based HTTP pull
- Flow control: client-side capacity gating (buffer remaining bytes)
- Eager acknowledgment: async ACK after receiving pages to free producer memory

#### 6A. Output Buffer (`exchange`)

**Known from research (Tasks 4.2.F [KG-11], 4.2.G [KG-12]):**

**Partition hash function** is a four-stage pipeline (NOT murmur3):
1. **Per-type hash:** Fixed-width types use a custom single-round xxHash64 mix: `rotateLeft(value * 0xC2B2AE3D27D4EB4F, 31) * 0x9E3779B185EBCA87`. Variable-width types (VARCHAR, VARBINARY) use the full xxHash64 algorithm. `NULL_HASH_CODE = 0`.
2. **Multi-column combining:** `hash = 31 * previousHash + columnHash` (polynomial, initial seed = 0).
3. **Hash-to-partition:** Lemire "fastrange" multiply-shift: `(Integer.toUnsignedLong(Long.hashCode(rawHash)) * partitionCount) >>> 32`. Avoids modulo bias without division.
4. **Bucket-to-partition indirection:** A `bucketToPartition` array decouples hash buckets from output partitions.

**Exchange HTTP protocol** — three endpoints, seven custom headers:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/task/{taskId}/results/{bufferId}/{token}` | GET | Fetch pages starting at sequence token |
| `/v1/task/{taskId}/results/{bufferId}/{token}/acknowledge` | GET | Fire-and-forget ACK for memory release |
| `/v1/task/{taskId}/results/{bufferId}` | DELETE | Destroy buffer |

| Header | Direction | Value |
|--------|-----------|-------|
| `X-Trino-Max-Size` | Request | Max response body size in bytes |
| `X-Trino-Internal-Bearer` | Request | JWT authentication (HKDF-SHA256 derived) |
| `X-Trino-Task-Instance-Id` | Response | UUID identifying the task instance |
| `X-Trino-Page-Sequence-Id` | Response | Starting sequence token of returned pages |
| `X-Trino-Page-End-Sequence-Id` | Response | Token for next fetch request |
| `X-Trino-Buffer-Complete` | Response | Boolean: all pages have been produced |
| `X-Trino-Task-Failed` | Response | Boolean: task has failed |

Empty buffer returns HTTP 204 (no content); non-empty returns 200 with binary body (16-byte envelope + pages).

**Implementation:**
- [ ] `OutputBuffer` trait with `PartitionedOutputBuffer` and `BroadcastOutputBuffer`
- [ ] Partition hash: implement Trino's exact xxHash64 mix + polynomial combining + Lemire fastrange
- [ ] `bucketToPartition` indirection array support
- [ ] Per-partition page queue with sequence tokens
- [ ] Memory-bounded: block producers (via async backpressure) when buffer exceeds limit
- [ ] `GET /v1/task/{taskId}/results/{bufferId}/{token}` — serve pages with 16-byte envelope, 4 response headers
- [ ] `GET .../acknowledge` — fire-and-forget ACK for memory release
- [ ] `DELETE /v1/task/{taskId}/results/{bufferId}` — destroy buffer
- [ ] `X-Trino-Internal-Bearer` JWT validation (HKDF-SHA256)
- [ ] `X-Trino-Max-Size` request header handling (cap response body size)

#### 6B. Exchange Client (`exchange`)

- [ ] `ExchangeClient`: pulls pages from multiple upstream workers concurrently
- [ ] Token-based idempotent GET requests with retry and backoff
- [ ] Client-side capacity gating (configurable buffer size, concurrency factor)
- [ ] Deserialize incoming Trino wire format pages into `RecordBatch`
- [ ] Feed deserialized batches into `ExchangeSourceExec` via async channel

#### 6C. Dynamic Filter Propagation

- [ ] Collect dynamic filters from build-side operators
- [ ] Include collected domains in `TaskStatus` for coordinator polling
- [ ] Accept incoming dynamic filters from `TaskUpdateRequest` and inject into scan operators

**Milestone:** A two-stage query (e.g., `SELECT ... FROM t1 JOIN t2 ON ...`) with scan on one worker, join on another, data shuffled between them via the Rust exchange protocol, consumed by the Java coordinator.

---

### Phase 7: Memory Management & Arbitration

**Objective:** Implement hierarchical memory tracking with RAII reservations and a process-wide arbitrator.

**Design synthesis:**

| Layer | Pattern Source | Implementation |
|-------|---------------|----------------|
| Reservation (leaf) | DataFusion | `MemoryReservation` struct with `Drop` impl that returns bytes |
| Tracking tree | Trino | Query -> Task -> Pipeline -> Operator hierarchy |
| Pool + limits | Trino + Velox | Single process-wide pool with per-query capacity limits |
| Arbitration | Velox | `MemoryArbitrator` coordinates cross-query spill/abort |
| Spill trigger | Velox (sync) | Synchronous on requesting thread (not async scheduler like Trino) |

#### 7A. Memory Pool & Reservation (`memory`)

- [ ] `MemoryPool` trait with `try_grow(bytes)` and `shrink(bytes)`
- [ ] `GreedyMemoryPool`: first-come-first-served, deny when full
- [ ] `FairPool`: per-consumer tracking, even distribution (DataFusion `FairSpillPool`)
- [ ] `MemoryReservation`: RAII wrapper, `try_grow()` returns `Result<()>`, `Drop` calls `pool.shrink()`
- [ ] `MemoryConsumer`: trait for operators that can spill when asked
- [ ] Per-query limits enforced at the query-level node in the tree

#### 7B. Tracking Tree

```
ProcessPool (single, total worker capacity)
  +-- QueryPool (per query, capacity set by coordinator's per-query limit)
       +-- TaskPool (per task)
            +-- OperatorReservation (leaf, RAII)
```

- [ ] Atomic counters at each level (no locks on the hot path)
- [ ] Quantized reservation at leaf level (Velox pattern: 1/4/8 MB tiers) to reduce parent updates
- [ ] Reporting: aggregate per-query usage for `GET /v1/memory` endpoint

#### 7C. Memory Arbitrator

- [ ] Process-wide `MemoryArbitrator` (Velox `SharedArbitrator` pattern)
- [ ] Multi-phase cascade: self-reclaim -> free capacity harvest -> cross-query spill -> abort
- [ ] Victim selection: capacity bucket x age (oldest queries served first)
- [ ] Spill dispatch to a dedicated Tokio task pool
- [ ] Abort escalation after configurable timeout

#### 7D. Operator Spill Integration

- [ ] `Spillable` trait for operators that support memory revocation
- [ ] Spill serialization: Arrow IPC to temp files (DataFusion pattern)
- [ ] Spill read-back: merge-sorted streams for aggregations, hash rebuild for joins
- [ ] `DiskManager` for temp file lifecycle management

**Milestone:** Under memory pressure, the worker spills hash join build data to disk, rebuilds on probe, and completes the query correctly. The arbitrator can abort a low-priority query to free memory for a higher-priority one.

---

### Phase 8: Advanced Operators

**Objective:** Complete the operator catalog to cover the majority of Trino's 47 plan node types.

#### Priority 1 (Required for most queries)

| Operator | Notes |
|----------|-------|
| `HashJoinExec` | Two-pipeline build/probe with spill. Vectorized hash lookup. |
| `AggregateExec` | Partial + Final modes. `GroupByHash` with adaptive partial disable. |
| `SortExec` | External merge sort with spill. |
| `TopNExec` | Heap-based, bounded memory. |
| `WindowExec` | Window functions with partition-by and order-by. |
| `MarkDistinctExec` | For `COUNT(DISTINCT ...)`. |
| `UnionExec` | Concatenate multiple streams. |

#### Priority 2 (Common but less critical)

| Operator | Notes |
|----------|-------|
| `SemiJoinExec` | IN subquery optimization. |
| `CrossJoinExec` | Nested loop join. |
| `TableWriterExec` | INSERT / CTAS via connector `PageSink`. |
| `SampleExec` | TABLESAMPLE implementation. |
| `AssignUniqueIdExec` | Monotonic ID generation. |
| `RowNumberExec` / `RankExec` | Ranking window functions. |

#### Priority 3 (Specialized)

| Operator | Notes |
|----------|-------|
| `SpatialJoinExec` | Geospatial joins. |
| `DynamicFilterSourceExec` | Collect domains from build side. |
| `GroupIdExec` | GROUPING SETS / CUBE / ROLLUP. |
| `MergeWriterExec` | MERGE statement. |
| `DeleteExec` | DELETE via connector. |

Each operator implementation follows the decision framework from GOAL.md Section 4 — a design document referencing how Trino, DataFusion, and Velox implement it, with trade-off analysis.

**Milestone:** The worker can execute TPC-H queries 1-6 (covering scan, filter, project, aggregation, join, sort, limit, and top-N).

---

### Phase 9: Integration & Hardening

**Objective:** End-to-end correctness and production readiness.

**Known from research (KG-13: Trino Phase 6 Tasks 6.1–6.6 + DataFusion Phase 6 Tasks 6.1–6.4):**

Trino uses four test tiers (unit/integration/product/benchmark) with no annotation-based categorization — all selective execution via Maven profiles. `DistributedQueryRunner` is fundamentally in-process (no `addExternalWorker()` method). However, the worker registration protocol is language-agnostic (HTTP `POST /v1/announce`), so a standalone coordinator with `workerCount=0` + `node-scheduler.include-coordinator=false` forces all execution to an externally registered Rust worker. Product tests use Testcontainers with the Tempto framework — `EnvSinglenodeCompatibility` already runs mixed-version clusters.

**Critical gap:** No byte-level golden fixtures exist anywhere in Trino. All tests verify semantic equality, not exact bytes. No golden hash values exist — all hash tests compare two Java implementations. This is entirely new testing infrastructure the Rust project must build.

DataFusion provides the idiomatic Rust testing reference: sqllogictest (474 `.slt` files), `insta` snapshot testing, `rstest` parametrization, inline `#[cfg(test)]` modules, single integration test binary pattern, and feature-flag-gated test tiers.

#### 9A. Golden File Conformance Suite (new, no equivalent in either engine)

Run the Java Trino implementation to capture reference outputs, hard-code in Rust tests:

- [ ] **Block encoding golden files:** Serialized bytes for all 13 encoding types with known seeds (port `BlockAssertions` seed 633969769)
- [ ] **Hash function golden values:** Partition hash outputs for known inputs across all types (BIGINT, VARCHAR, DECIMAL, etc.)
- [ ] **JSON wire type snapshots:** Captured `TaskUpdateRequest`, `TaskStatus`, `TaskInfo`, `PlanFragment`, `SessionRepresentation`, `SplitAssignment` JSON from a running coordinator
- [ ] **Page binary golden files:** Serialized pages with all compression codecs (none, LZ4, ZSTD)
- [ ] **Exchange protocol recordings:** Full HTTP request/response pairs (headers + binary body) for fetch/ACK/destroy flows

#### 9B. Hybrid Test Harness (Java coordinator + Rust worker)

- [ ] **Coordinator launcher:** Start a Java Trino coordinator via Docker or `TpchQueryRunner.main()` with `workerCount=0` + `node-scheduler.include-coordinator=false`
- [ ] **Worker registration:** Rust worker process registers via `POST /v1/announce`, serves `GET /v1/info` with matching version/environment
- [ ] **TPC-H correctness:** Run Q1–Q22 via Trino REST API, compare results against all-Java baseline
- [ ] **TPC-DS correctness:** Extended query coverage
- [ ] **Mixed-cluster testing:** Java and Rust workers in the same cluster; verify shuffle interoperability
- [ ] **sqllogictest adapter:** Two-engine setup — Java Trino as oracle (seed expected results via `--complete`), Rust worker as target

#### 9C. Operator-Level Test Infrastructure (`test-utils` crate)

Port key test utilities from both engines:

- [ ] **Operator test driver:** Rust equivalent of Trino's `OperatorAssertion.toPages()` (implements `needsInput → addInput → getOutput` protocol)
- [ ] **Page builder:** Rust equivalent of `RowPagesBuilder` — fluent API for synthetic input pages
- [ ] **In-memory spiller:** Rust equivalent of `DummySpillerFactory` — stores pages in `Vec`, counts spill operations
- [ ] **Mock data source:** Rust equivalent of DataFusion's `TestMemoryExec` — configurable partitions and projections
- [ ] **Assertion macros:** `assert_batches_eq!`, `assert_batches_sorted_eq!` (from DataFusion)
- [ ] **Snapshot testing:** `insta` for plan display and error messages
- [ ] **Parametrized tests:** `rstest` for batch size × join type × spill on/off × data type combinations
- [ ] **Systematic spill matrix:** Every stateful operator tested with `(spillEnabled, memoryLimit)` combinations
- [ ] **Cancellation testing:** `BlockingExec` + `assert_is_pending` + `Weak::strong_count` convergence (DataFusion pattern)

#### 9D. Product Test Integration

- [ ] **`EnvMultinodeRustWorker` Docker environment:** Reuse existing coordinator container and Tempto test runner, replace worker container with Rust binary
- [ ] **Health check:** Replace `jps | grep TrinoServer` with HTTP-based `GET /v1/info` check
- [ ] **Connector smoke tests:** Run existing Hive/Iceberg/TPCH product test suites unchanged against Rust worker

#### 9E. CI Test Tiers

- [ ] **Tier 1 (every PR):** Unit tests, wire format round-trips, golden file conformance, operator tests — `cargo test`
- [ ] **Tier 2 (post-merge):** Integration tests requiring Java coordinator, TPC-H correctness, fuzz tests — `cargo test --features integration_tests,extended_tests`
- [ ] **Tier 3 (nightly):** Product tests via Docker, mixed-cluster shuffle tests, performance benchmarks

#### 9F. Infrastructure

- [ ] **Graceful shutdown:** Drain active tasks before stopping the process
- [ ] **Logging & tracing:** Structured logging with `tracing` crate; distributed tracing integration

**Milestone:** A multi-node cluster with Rust workers passes TPC-H Q1–Q22 correctness tests, golden file conformance for all wire formats, and product test suites for Hive/Iceberg connectors.

---

## Knowledge Gaps & Required Research

All 13 knowledge gaps have been **resolved** via research tasks in the TRINO_TRACING_GUIDE and DATAFUSION_TRACING_GUIDE. The findings are integrated into the phase descriptions above.

| ID | Gap | Phase | Status | Research Task |
|----|-----|-------|--------|---------------|
| KG-1 | Block-level binary wire encoding | Phase 1 | **RESOLVED** | Task 4.2.D |
| KG-2 | Page envelope & block type tags | Phase 1 | **RESOLVED** | Task 4.2.E |
| KG-3 | Worker discovery/registration protocol | Phase 2 | **RESOLVED** | Task 4.1.D |
| KG-4 | `TaskUpdateRequest` full JSON schema | Phase 2 | **RESOLVED** | Task 4.1.E |
| KG-5 | `TaskStatus` full JSON schema | Phase 2 | **RESOLVED** | Task 4.1.H |
| KG-6 | Plan fragment JSON structure | Phase 3 | **RESOLVED** | Task 4.1.F |
| KG-7 | Expression serialization format | Phase 3 | **RESOLVED** | Task 3.3.B |
| KG-8 | Trino type system JSON serialization | Phase 3 | **RESOLVED** | Task 4.1.G |
| KG-9 | Pipeline/operator stats JSON schema | Phase 4 | **RESOLVED** | Task 4.1.I |
| KG-10 | HiveSplit / IcebergSplit JSON format | Phase 5 | **RESOLVED** | Task 4.3.C |
| KG-11 | Partition hash function for shuffle | Phase 6 | **RESOLVED** | Task 4.2.F |
| KG-12 | Exchange protocol HTTP headers | Phase 6 | **RESOLVED** | Task 4.2.G |
| KG-13 | Trino & DataFusion test structure | Phase 9 | **RESOLVED** | Trino Phase 6 (Tasks 6.1–6.6) + DF Phase 6 (Tasks 6.1–6.4) |
| KG-14 | SIMD vectorization across engines | Phase 1, 5 | **RESOLVED** | Trino Task 3.6.A + DF Task 3.6.A + Velox Task 3.6.A |

New knowledge gaps discovered during implementation should follow the same process: formulate a research task, append to the tracing guide, execute, then integrate findings before proceeding.

---

## Risk Register

| Risk | Impact | Mitigation | Status |
|------|--------|------------|--------|
| **Plan fragment JSON is undocumented and complex** | Blocks Phase 3 entirely | KG-6/KG-7 resolved: 47 PlanNode types with `@type` tags, 18 Expression subtypes as JSON AST. Capture live JSON via Trino proxy for validation. | **Mitigated** |
| **Wire format has undocumented edge cases** | Silent data corruption in mixed clusters | KG-1/KG-2 resolved: 13 block encodings, MSB-first null bitmaps, 64KB chunk compression documented. Round-trip fuzz testing + cross-validate with Java `PagesSerde`. | **Mitigated** |
| **Coordinator expects specific operator stats fields** | Coordinator may reject task status or make bad scheduling decisions | KG-9 resolved: OperatorStats 40 fields, Metrics with Java class name discriminators. Integration test against real coordinator from Phase 2. | **Mitigated** |
| **Expression format is bytecode (not AST)** | Cannot use DataFusion `PhysicalExpr`; need custom evaluator | **ELIMINATED.** KG-7 confirmed expressions are JSON AST — maps directly to DataFusion `PhysicalExpr`. | **Eliminated** |
| **MSB-first null bitmap conversion** | Off-by-one or bit-order bugs cause silent data corruption | Arrow uses LSB-first, Trino uses MSB-first. Requires careful bit-reversal at the serialization boundary. Comprehensive property-based testing required. | **New** |
| **Constant base64 Block deserialization** | Incorrect constant values in expressions | Constants are base64-encoded Trino Block binary, not JSON primitives. Must implement Block binary parser for each of the 13 encoding types just to read a literal `10`. | **New** |
| **Partition hash compatibility** | Rows routed to wrong partitions in mixed clusters | KG-11 resolved: custom xxHash64 mix (not murmur3) + Lemire fastrange. Must match bit-for-bit. Property-test against Java implementation. | **Mitigated** |
| **Tokio scheduler lacks MLFQ fairness** | Long queries starve short queries under load | Acceptable for initial phases. Mitigate later with custom Tokio runtime hooks or cooperative task budgets. | **Accepted** |
| **Arrow compute kernels lack Trino function parity** | Incorrect results for edge-case SQL functions | Implement custom kernels for missing functions. Use Trino's test suite as correctness oracle. | **Open** |
| **Parquet reader performance gap** | Rust worker slower than Java on scan-heavy queries | The `parquet` crate is actively optimized. Profile early; contribute upstream fixes if needed. | **Open** |
| **No cross-language golden fixtures exist** | Rust wire output may be semantically correct but byte-incompatible | KG-13 resolved: Trino tests verify semantic equality, not bytes. Must build golden file suites from Java (block encodings, hash values, JSON schemas, page binaries). | **Mitigated** |
| **`DistributedQueryRunner` is in-process only** | Cannot plug Rust worker into existing integration tests | KG-13 resolved: Use standalone coordinator + HTTP announce protocol with `workerCount=0`. Product tests can inject via `EnvMultinodeRustWorker` Docker environment. | **Mitigated** |
| **SIMD strategy unclear** | Suboptimal compute performance if relying solely on auto-vectorization | KG-14 resolved: hybrid approach — LLVM auto-vectorization for general compute (DataFusion pattern), explicit SIMD via `std::arch` for hash probing, wire format compress/expand, and bitmap operations (Velox pattern). Build with LTO + `-C target-cpu=native`. | **Mitigated** |
