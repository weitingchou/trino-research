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
|   +-- trw-server/             HTTP server, REST endpoints (axum)
|   +-- trw-protocol/           Trino wire format, JSON serde, plan node types
|   +-- trw-execution/          Task lifecycle, stream orchestration, scheduling
|   +-- trw-operators/          Operator implementations (scan, filter, join, etc.)
|   +-- trw-memory/             MemoryPool, MemoryReservation, Arbitrator
|   +-- trw-connectors/         Connector trait, Hive/Iceberg Parquet reader
|   +-- trw-exchange/           Output buffers, exchange client, shuffle
|   +-- trw-common/             Shared types, errors, config, metrics
```

### Dependency DAG

```
trw-server
  +-- trw-protocol
  +-- trw-execution
  +-- trw-exchange

trw-execution
  +-- trw-protocol
  +-- trw-operators
  +-- trw-memory
  +-- trw-common

trw-operators
  +-- trw-memory
  +-- trw-connectors
  +-- trw-common
  +-- arrow / datafusion-physical-expr  (external)

trw-exchange
  +-- trw-protocol
  +-- trw-memory
  +-- trw-common

trw-connectors
  +-- trw-memory
  +-- trw-common
  +-- object_store / parquet  (external)

trw-memory
  +-- trw-common
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
- [ ] Create a `trw-server` binary that starts an axum HTTP server on a configurable port
- [ ] Implement `GET /v1/info` (returns static server info JSON) as smoke test

**Milestone:** `cargo build` succeeds; `curl localhost:8080/v1/info` returns a JSON response.

---

### Phase 1: Data Model & Wire Format

**Objective:** Define the internal data representation and implement Trino's binary serialization so the worker can produce bytes that the coordinator (and Java workers) can consume.

**Why first:** Everything downstream — operators, exchange, connectors — produces or consumes data. The data model is the foundation.

#### 1A. Internal Data Model (`trw-common`)

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

#### 1B. Trino Wire Format (`trw-protocol`)

Implement serialization/deserialization of Trino's binary page format so data can flow between Java and Rust workers.

**Known from research (Trino Phase 4, Task 4.2.A):**
- 12-byte page header: `[positionCount:i32][uncompressedSize:i32][compressedSize:i32]`
- Compression (LZ4/ZSTD) and optional AES-CBC encryption applied per-block
- `PagesSerde` handles serialization; `CompressingEncryptingPageSerializer` wraps it

**Implementation:**
- [ ] `PageSerializer`: `RecordBatch` -> Trino binary bytes
- [ ] `PageDeserializer`: Trino binary bytes -> `RecordBatch`
- [ ] LZ4/ZSTD block-level compression support
- [ ] Round-trip property tests: generate random RecordBatches, serialize, deserialize, assert equality

> **KNOWLEDGE GAP [KG-1]: Block-Level Binary Encoding**
> The research covers the page-level 12-byte header and the existence of `PagesSerde`, but does not document the exact byte-level encoding for each Block type (how `LongArrayBlock` lays out its `long[]` + `boolean[]` in the wire format, how `VariableWidthBlock` encodes offsets + data, how `DictionaryBlock` serializes the dictionary + ids, null bitmap encoding, etc.). This is critical for bit-level compatibility.
> **Required research task:** → Append to `TRINO_TRACING_GUIDE.md`

> **KNOWLEDGE GAP [KG-2]: Block Type Tags**
> When `PagesSerde` serializes a Page, how does it encode which Block type each column is? Is there a type tag byte? An enum? How does the deserializer know whether to read a `LongArrayBlock` vs. `VariableWidthBlock`?
> **Required research task:** → Append to `TRINO_TRACING_GUIDE.md`

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

> **KNOWLEDGE GAP [KG-3]: Worker Discovery Protocol**
> How does a Trino worker register itself with the coordinator on startup? The research covers task-level communication but not the initial worker registration. Is it based on Airlift's discovery service? Does the worker POST to a coordinator endpoint? Is there a heartbeat loop?
> **Required research task:** → Append to `TRINO_TRACING_GUIDE.md`

#### 2B. Task Lifecycle Endpoints (`trw-server`)

- [ ] `POST /v1/task/{taskId}` — accept `TaskUpdateRequest`, delegate to `TaskManager`
- [ ] `GET /v1/task/{taskId}/status` — long-poll with version tracking (tokio `watch` channel)
- [ ] `DELETE /v1/task/{taskId}` — trigger abort/cancel state transition
- [ ] `GET /v1/memory` — return `MemoryInfo` JSON snapshot

#### 2C. Task State Machine (`trw-execution`)

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

> **KNOWLEDGE GAP [KG-4]: TaskUpdateRequest Full JSON Schema**
> We know the REST endpoint and that it carries plan fragments + splits, but the exact JSON schema — all fields, optional vs. required, nested types, version compatibility — is not fully documented in the research. This is critical for serde deserialization.
> **Required research task:** → Append to `TRINO_TRACING_GUIDE.md`

> **KNOWLEDGE GAP [KG-5]: TaskStatus Full JSON Schema**
> Same gap for the response schema of `GET /v1/task/{taskId}/status`. The coordinator's deserialization must match our serialization exactly.
> **Required research task:** → Append to `TRINO_TRACING_GUIDE.md`

**Milestone:** A real Trino coordinator can send a `TaskUpdateRequest` to the Rust worker, poll status, and see the task transition through `RUNNING -> FINISHED`. (The task doesn't execute anything yet — it just accepts and acks.)

---

### Phase 3: Plan Fragment Parsing

**Objective:** Deserialize the physical plan JSON from the coordinator into an internal plan tree that the execution engine can compile.

**Known from research (Trino Phase 2, Task 2.1.A; Phase 3, Task 3.1.C):**
- Coordinator sends a `PlanFragment` as JSON inside `TaskUpdateRequest`
- The fragment contains a tree of plan nodes (TableScan, Filter, Project, HashJoin, Aggregation, Exchange, Output, etc.)
- Each plan node has typed properties (column handles, partition functions, expressions, etc.)
- ~43 operator types in Trino 480

#### 3A. Plan Node Type System (`trw-protocol`)

- [ ] Define Rust enums/structs for each Trino plan node type
- [ ] Implement `serde::Deserialize` for the plan fragment JSON
- [ ] Handle expression deserialization (filter predicates, projection expressions)

> **KNOWLEDGE GAP [KG-6]: Plan Fragment JSON Structure**
> The research identifies `PlanFragment` at a conceptual level but does not trace the exact JSON serialization. Specifically: how are plan nodes tagged (e.g., `@type` field?), how are expressions encoded (AST? serialized bytecode? string representation?), how are type handles serialized? This is the most critical gap — without it, we cannot parse coordinator instructions.
> **Required research task:** → Append to `TRINO_TRACING_GUIDE.md`

> **KNOWLEDGE GAP [KG-7]: Expression Serialization Format**
> Trino compiles SQL expressions into bytecode at planning time (`ExpressionCompiler`, `PageProcessor`). But what does the coordinator send to the worker — the original expression AST, or pre-compiled bytecode, or something else? The answer determines whether we need a Trino expression interpreter/compiler or can map directly to DataFusion `PhysicalExpr`.
> **Required research task:** → Append to `TRINO_TRACING_GUIDE.md`

#### 3B. Plan-to-Execution Compilation (`trw-execution`)

Convert the parsed plan tree into executable async stream pipelines (DataFusion-style):

```
PlanFragment JSON
  -> PlanNode tree (trw-protocol, Phase 3A)
  -> Pipeline of Arc<dyn ExecutionPlan> (trw-execution, Phase 3B)
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

- [ ] Expression translator: Trino expression AST -> DataFusion `PhysicalExpr`
- [ ] Type mapping: Trino types -> Arrow `DataType`
- [ ] Partitioning strategy: map Trino's pipeline taxonomy (split-lifecycle, task-lifecycle, remote-source) to Tokio task spawning

> **KNOWLEDGE GAP [KG-8]: Trino Type System Serialization**
> How are Trino SQL types (`VARCHAR`, `BIGINT`, `DECIMAL(p,s)`, `ARRAY(T)`, `ROW(...)`) serialized in the plan fragment JSON? We need exact mapping to Arrow `DataType`.
> **Required research task:** → Append to `TRINO_TRACING_GUIDE.md`

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

#### 4A. Task Execution Manager (`trw-execution`)

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

> **KNOWLEDGE GAP [KG-9]: Pipeline Stats JSON Schema**
> The coordinator expects specific pipeline/operator statistics in the task status response. What fields does it expect? (`rawInputDataSize`, `processedInputPositions`, `outputPositions`, etc.)
> **Required research task:** → Append to `TRINO_TRACING_GUIDE.md`

**Milestone:** The worker can receive a trivial plan (e.g., `VALUES (1, 'hello')`), execute it as a Tokio stream, and produce a `RecordBatch` result internally.

---

### Phase 5: Core Operators & Storage Plane

**Objective:** Implement the minimum operator set to run a real query: scan, filter, project, and the connector interface for reading Parquet from object storage.

#### 5A. Connector Trait (`trw-connectors`)

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

- [ ] Parse Trino's `HiveSplit` JSON (file path, byte offset/length, partition keys)
- [ ] Async Parquet reading via `parquet` crate + `object_store`
- [ ] Row-group pruning via min/max statistics
- [ ] Predicate pushdown to Parquet page index (where supported)
- [ ] Column projection pushdown (only read requested columns)
- [ ] Memory-aware: register allocations against `MemoryReservation`

> **KNOWLEDGE GAP [KG-10]: HiveSplit / IcebergSplit JSON Format**
> What is the exact JSON structure of a `HiveSplit` or `IcebergSplit` as sent by the coordinator? Fields like file path, offset, length, partition values, bucket number, etc. This must be deserialized precisely.
> **Required research task:** → Append to `TRINO_TRACING_GUIDE.md`

#### 5C. Core Operators (`trw-operators`)

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

#### 6A. Output Buffer (`trw-exchange`)

- [ ] `OutputBuffer` trait with `PartitionedOutputBuffer` and `BroadcastOutputBuffer`
- [ ] Partition routing: hash partitioning using Trino's partition function (murmur3-based)
- [ ] Per-partition page queue with sequence tokens
- [ ] Memory-bounded: block producers (via async backpressure) when buffer exceeds limit
- [ ] `GET /v1/task/{taskId}/results/{bufferId}/{token}` — serve serialized pages
- [ ] Eager ACK endpoint to free consumed pages
- [ ] `BufferResult` response with `nextToken` and `bufferComplete` flag

> **KNOWLEDGE GAP [KG-11]: Partition Hash Function**
> Trino uses a specific hash function to assign rows to output partitions during shuffle. Which hash function? How is it applied to different types? Is it murmur3? Is it the same as `TypeOperators.hashCodeOperator()`? The exact function must match for shuffle compatibility between Java and Rust workers.
> **Required research task:** → Append to `TRINO_TRACING_GUIDE.md`

> **KNOWLEDGE GAP [KG-12]: Output Buffer HTTP Headers**
> The exchange protocol uses specific HTTP headers (`X-Trino-Buffer-Remaining-Bytes`, `X-Trino-Max-Size`, `X-Trino-Task-Instance-Id`, etc.). What are all the headers, their semantics, and which ones are required for compatibility?
> **Required research task:** → Append to `TRINO_TRACING_GUIDE.md`

#### 6B. Exchange Client (`trw-exchange`)

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

#### 7A. Memory Pool & Reservation (`trw-memory`)

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

**Objective:** Complete the operator catalog to cover the majority of Trino's ~43 operators.

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

- [ ] **Protocol conformance tests:** Record real Java worker HTTP request/response pairs; replay against Rust worker; assert byte-level match
- [ ] **TPC-H / TPC-DS correctness:** Run full benchmark suites against a hybrid cluster (Java coordinator + Rust workers); compare results against all-Java baseline
- [ ] **Mixed-cluster testing:** Java and Rust workers in the same cluster; verify shuffle interoperability
- [ ] **Failure injection:** Kill workers mid-query; verify coordinator detects failure and retries
- [ ] **Memory pressure testing:** Run queries that exceed worker memory; verify spill/abort behavior
- [ ] **Performance benchmarking:** Single-node and cluster TPC-H throughput comparison
- [ ] **Graceful shutdown:** Drain active tasks before stopping the process
- [ ] **Logging & tracing:** Structured logging with `tracing` crate; distributed tracing integration

**Milestone:** A multi-node cluster with Rust workers passes TPC-H correctness tests and handles failure scenarios gracefully.

---

## Knowledge Gaps & Required Research

The following gaps must be resolved (per GOAL.md Section 4.1 "No Guessing" rule) before implementation can proceed for the affected phases. Each gap will generate a new research task appended to the appropriate tracing guide.

| ID | Gap | Affects Phase | Target Guide |
|----|-----|---------------|-------------|
| KG-1 | Block-level binary wire encoding (per-type byte layout) | Phase 1 | TRINO_TRACING_GUIDE |
| KG-2 | Block type tags in serialized pages | Phase 1 | TRINO_TRACING_GUIDE |
| KG-3 | Worker discovery/registration protocol | Phase 2 | TRINO_TRACING_GUIDE |
| KG-4 | `TaskUpdateRequest` full JSON schema | Phase 2 | TRINO_TRACING_GUIDE |
| KG-5 | `TaskStatus` full JSON schema | Phase 2 | TRINO_TRACING_GUIDE |
| KG-6 | Plan fragment JSON structure (node tagging, nesting) | Phase 3 | TRINO_TRACING_GUIDE |
| KG-7 | Expression serialization format (AST vs bytecode vs string) | Phase 3 | TRINO_TRACING_GUIDE |
| KG-8 | Trino type system JSON serialization | Phase 3 | TRINO_TRACING_GUIDE |
| KG-9 | Pipeline/operator stats JSON schema | Phase 4 | TRINO_TRACING_GUIDE |
| KG-10 | HiveSplit / IcebergSplit JSON format | Phase 5 | TRINO_TRACING_GUIDE |
| KG-11 | Partition hash function for shuffle | Phase 6 | TRINO_TRACING_GUIDE |
| KG-12 | Exchange protocol HTTP headers | Phase 6 | TRINO_TRACING_GUIDE |

**Execution order:** KG-6 and KG-7 are the highest priority — they gate Phase 3, which gates all execution. KG-1 and KG-2 gate Phase 1B (wire format). KG-3 gates Phase 2A (worker startup). All others can be resolved just-in-time as their respective phases begin.

---

## Risk Register

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Plan fragment JSON is undocumented and complex** | Blocks Phase 3 entirely | Resolve KG-6/KG-7 immediately. Consider capturing live JSON via Trino proxy as additional source of truth. |
| **Wire format has undocumented edge cases** | Silent data corruption in mixed clusters | Extensive round-trip fuzz testing (Phase 1 milestone). Cross-validate with Java `PagesSerde` via JNI test harness. |
| **Coordinator expects specific operator stats fields** | Coordinator may reject task status or make bad scheduling decisions | Resolve KG-9 early. Run integration tests against real coordinator from Phase 2 onward. |
| **Expression format is bytecode (not AST)** | Cannot use DataFusion `PhysicalExpr`; need custom evaluator | Resolve KG-7. If bytecode, consider building a bytecode interpreter or mapping bytecode ops to Arrow kernels. |
| **Tokio scheduler lacks MLFQ fairness** | Long queries starve short queries under load | Acceptable for initial phases. Mitigate later with custom Tokio runtime hooks or cooperative task budgets. |
| **Arrow compute kernels lack Trino function parity** | Incorrect results for edge-case SQL functions | Implement custom kernels for missing functions. Use Trino's test suite as correctness oracle. |
| **Parquet reader performance gap** | Rust worker slower than Java on scan-heavy queries | The `parquet` crate is actively optimized. Profile early; contribute upstream fixes if needed. |
