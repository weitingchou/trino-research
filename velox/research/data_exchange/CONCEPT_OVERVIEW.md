# Phase 4: Data Exchange — Inter-Node Shuffle & Storage I/O (Velox)

Velox is a library, not a server — it has no REST endpoint, no HTTP layer, no built-in wire protocol for inter-node communication. Instead, it defines precise C++ API boundaries where the host application (Prestissimo, Spark Gluten, etc.) plugs in. Data exchange spans four boundaries: the `exec::Task` control API where the host pushes plan fragments and splits, the `OutputBuffer` system where serialized results queue for downstream consumers, the `ExchangeClient` fetch coordinator where upstream data is pulled through a host-provided transport abstraction, and the `Connector` SPI where storage I/O and predicate pushdown meet the file reader. The unifying design principle is **separation of coordination from transport** — Velox handles all scheduling, memory budgeting, backpressure, and serialization logic internally, but delegates network transport and storage access to pluggable host-provided implementations.

## 1. The Host Control Boundary: `exec::Task` as Sole Entry Point

### Library-as-a-Service, Not Server-as-a-Service

Unlike Trino, which exposes a REST API (`/v1/task/{taskId}`) where the coordinator pushes splits over HTTP, Velox is a C++ library with in-process function calls:

```
Trino:     Coordinator --HTTP POST /v1/task/{taskId}/split--> Worker (Netty) --> TaskManager --> SqlTask
Velox:     Prestissimo (HTTP handler) --C++ call--> Task::addSplit(planNodeId, Split(HiveConnectorSplit(...)))
```

Zero serialization at the API boundary — the host constructs C++ objects directly.

### Seven Host-Facing Methods

| Method | Purpose |
|--------|---------|
| `Task::create()` | Construct task from `PlanFragment` (plan tree + execution strategy) |
| `Task::start()` | Trigger parallel multi-threaded execution on executor pool |
| `Task::next()` | Pull results one batch at a time (serial mode) |
| `Task::addSplit()` / `addSplitWithSequence()` | Push data-location descriptors incrementally |
| `Task::noMoreSplits()` | Signal end of input for a plan node |
| `Task::updateOutputBuffers()` | Adjust downstream consumer count dynamically |
| `Task::requestCancel()` / `requestAbort()` | Terminate execution |

Key design facts:

- **Splits arrive before or after `start()`** — the promise-based future mechanism (`ContinuePromise`/`ContinueFuture`) provides rendezvous between host push and driver pull. `ensureSplitGroupsAreBeingProcessedLocked()` handles both orderings.
- **Sequence-based deduplication** (`addSplitWithSequence`) enables at-least-once delivery semantics for distributed coordinators that may retry split assignments. The sequence ID and max-sequence acknowledgement are separate, allowing batch acknowledgement.
- **Grouped execution** creates drivers on-demand as split groups arrive, bounded by `concurrentSplitGroups_`. Unlike Trino's fixed driver count at scheduling time, Velox's driver population grows dynamically.
- **Barrier processing** enables streaming micro-batch within serial mode — the host adds splits, requests a barrier, drains results via `next()`, then repeats. Unique to Velox; neither Trino nor DataFusion have an equivalent.
- **Single `std::timed_mutex`** protects all Task state. The "collect promises under lock, fulfill outside lock" pattern (via `EventCompletionNotifier` RAII helper) avoids blocking driver threads while holding the mutex.

### Task State Machine

```
                 +----------+
    create() --> | kRunning  |
                 +-----+----+
                       |
          +------------+----------+----------+
          |            |          |          |
     (all drivers  (user cancel) (ext err)  (exception)
      finish)         |          |          |
          |            v          v          v
    +-----v-----+ +--------+ +--------+ +--------+
    | kFinished  | |kCanceled| |kAborted| | kFailed|
    +-----------+ +--------+ +--------+ +--------+
```

## 2. Output Buffers: Serialization on the Driver Thread

### Three-Layer Architecture

```
RowVector (columnar in-memory)
    |
    v
PartitionedOutput operator          -- per-driver: hashes rows, assigns to destinations
    |  (one detail::Destination per partition)
    v
OutputBufferManager (global singleton) -- taskId lookup
    |
    v
OutputBuffer (per-task)             -- holds N DestinationBuffer queues
    |
    v
DestinationBuffer                   -- per-downstream-consumer page queue
    |
    v
Network layer (Prestissimo HTTP)    -- getData() callback-driven pull
```

### Three Distribution Modes

| Kind | Behavior | Memory Optimization |
|------|----------|---------------------|
| `kPartitioned` | Hash or round-robin: each row to exactly one destination | Per-destination serialization buffers |
| `kBroadcast` | Every page cloned to all destinations | `shared_ptr` sharing; last acknowledge frees memory |
| `kArbitrary` | Shared pool; first active consumer takes | Single `ArbitraryBuffer` queue |

### Serialization Design

- **Serialization happens on the driver thread**, inside the `PartitionedOutput` operator — not in the buffer layer. This is the key difference from Trino, where serialization occurs in the `OutputBuffer`.
- Three serde formats: `Presto` (columnar, default for network shuffle), `CompactRow` (row-oriented, for spilling/local exchange), `UnsafeRow` (Spark compatibility).
- **Staggered flush thresholds**: Each `Destination` randomizes its `targetSizePct_` to 70-120% of nominal, preventing bursty traffic patterns where all destinations flush simultaneously.
- **Hash partitioning** uses `VectorHasher` with two mapping strategies: simple modulo for remote shuffle, bit-range extraction for power-of-two partitions. Local exchange applies `localExchangeHash()` (reverse + XXH32) to decorrelate from remote hash bits.

### Backpressure

Two-level backpressure with hysteresis:

1. **OutputBuffer level**: When `bufferedBytes_ >= maxSize_` (32 MB), producer driver blocks via `ContinueFuture`. Unblocks when bytes drop below `continueSize_` (90% of max).
2. **PartitionedOutput level**: Per-destination page size bounded by `min(1MB, maxBufferedBytes / numDestinations)` with 60 KB floor. Blocked destinations report `kWaitForConsumer` to the driver loop.

Consumers pull via callback-based `getData(destination, maxBytes, sequence, notify)` — sequence numbers provide TCP-style sliding window acknowledgement.

## 3. Exchange Client: Fetch Coordination With Pluggable Transport

### The Factory Pattern as the Transport Boundary

```cpp
using Factory = std::function<std::shared_ptr<ExchangeSource>(
    const std::string& remoteTaskId, int destination,
    std::shared_ptr<ExchangeQueue> queue, memory::MemoryPool* pool)>;
```

Velox never imports HTTP, gRPC, or any transport library. The host registers a factory that creates concrete `ExchangeSource` implementations — Prestissimo registers HTTP-based sources, tests register `LocalExchangeSource`. The `ExchangeSource` contract is minimal: `shouldRequestLocked()`, `request()`, `requestDataSizes()`, `pause()`, `close()`.

### Two-Phase Fetch Protocol

1. **Size probing** — Sources with unknown data availability receive `requestDataSizes()` (zero-cost metadata fetch) to learn what pages are buffered.
2. **Data transfer** — Sources that responded with non-empty `remainingBytes` move to `producingSources_`. The client requests actual data bounded by available queue capacity: `maxQueuedBytes_ - queue_->totalBytes() - totalPendingBytes_`.
3. **Deadlock prevention** — If no normal request fits but the queue is starving (below `minOutputBatchBytes_`), a forced out-of-band transfer of a single large page is issued.
4. **Single-source optimization** — When only one source exists, the size-probing phase is skipped entirely.

### Shared Client, Independent Consumers

All drivers in a pipeline share a single `ExchangeClient` (and `ExchangeQueue`). Only driver 0 manages splits (`processSplits_ = true`). The queue wakes consumers proportionally — it calculates unassigned bytes after accounting for already-unblocked consumers to prevent thundering herd.

### Self-Sustaining Request Chain

Each async response (via `folly::SemiFuture<Response>`) triggers evaluation of whether more data should be fetched. The callback re-categorizes sources (empty → producing → completed) and issues the next round of requests, maintaining the pipeline without driver thread involvement.

### Progressive Fetching

The `Exchange` operator uses `folly::collectAny` on both split futures and data futures — it begins consuming data before all upstream tasks are known. Task ID randomization (`std::shuffle`) before adding to the client prevents multiple downstream tasks from hammering the same upstream simultaneously.

## 4. Connector SPI: Single-Shot Pushdown With SIMD-Native Filters

### The Narrow Contract

```cpp
virtual std::unique_ptr<DataSource> createDataSource(
    const RowTypePtr& outputType,               // columns to produce
    const ConnectorTableHandlePtr& tableHandle,  // filters + table metadata
    const connector::ColumnHandleMap& columnHandles, // column mapping
    ConnectorQueryCtx* connectorQueryCtx) = 0;
```

Three things in, row batches out. The engine provides the output schema, a table handle carrying filters, and per-split metadata.

### Three-Tier Filter Architecture

| Level | Type | Where Applied | Example |
|-------|------|--------------|---------|
| `SubfieldFilters` | `unordered_map<Subfield, FilterPtr>` | DWIO decode layer, per-value SIMD tests | `a > 10`, `b IN ('x','y')` |
| `MetadataFilter` | Boolean expression tree over leaf filters | Row-group/stripe statistics pruning | Same predicates, evaluated against column stats |
| `RemainingFilter` | Arbitrary `ExprSet` expression | Post-reader row-by-row evaluation | `a < b`, cross-column comparisons |

### The ScanSpec Tree: Unified Filter + Projection

Velox's most distinctive abstraction — merges filter and projection into a single tree mirroring column nesting:

```
ScanSpec("root")
  +-- ScanSpec("a")  [channel=0, projectOut=true, filter=BigintRange(10,100)]
  +-- ScanSpec("b")  [channel=1, projectOut=true]
  |     +-- ScanSpec("keys")    [filter=IsNotNull]
  |     +-- ScanSpec("values")  [projectOut=true]
  +-- ScanSpec("c")  [no channel, projectOut=false, filter=BytesValues("x","y")]
```

Each `SelectiveColumnReader` receives its `ScanSpec` node and applies the filter at decode time. Filter-only columns (`projectOut_ == false`) are tested but never materialized.

### Single-Shot vs. Multi-Round Pushdown

**Velox**: The `HiveTableHandle` arrives fully formed with all `SubfieldFilters` and the `remainingFilter` already decided by the upstream planner. The `HiveDataSource` constructor processes everything in one pass — clones filters, extracts more from the remaining filter via `extractFiltersFromRemainingFilter()`, builds the `ScanSpec` tree. No negotiation.

**Trino**: Multi-round protocol where `ConnectorMetadata.applyFilter()` / `applyProjection()` are called during planning. The connector has agency — it can accept or reject pushdowns.

This is deliberate: Velox is an execution-only library; planning-time pushdown negotiation belongs to the coordinator.

### Dynamic Filter Injection

Dynamic filters from hash join build sides arrive via `addDynamicFilter(channel, filter)`:
1. Merged with static filters via `Filter::mergeWith()` (e.g., BigintRange AND BloomFilter).
2. Installed into the `ScanSpec` tree via `setFilter()`.
3. `resetCachedValues(true)` triggers filter reordering by runtime selectivity.
4. `moveAdaptationFrom()` transfers filter order and dynamic filters between splits, preserving runtime optimizations.

The `PushdownFilters` struct is wrapped in `folly::Synchronized` with read/write lock separation — scans hold a read lock, filter updates hold a write lock.

## 5. Comparison with Trino and DataFusion

### Host Control Boundary

| Dimension | Velox | Trino | DataFusion |
|-----------|-------|-------|------------|
| API style | C++ library calls (`Task::addSplit`) | REST over HTTP (`/v1/task/{taskId}`) | Rust library calls (`execute()`) |
| Split delivery | Incremental, async, push-based | Incremental, HTTP push-based | Upfront via `TableProvider` at plan time |
| Execution trigger | `start()` (parallel) or `next()` (serial) | `SqlTask.updateTask()` | `collect()` or `execute()` on physical plan |
| Split deduplication | Sequence-based (`addSplitWithSequence`) | Sequence-based (REST task update) | N/A (no distributed split assignment) |
| Dynamic drivers | Grouped execution creates drivers on-demand | Fixed driver count at scheduling time | Fixed partition count at plan time |

### Output Buffering

| Dimension | Velox | Trino | DataFusion |
|-----------|-------|-------|------------|
| Serialization location | Driver thread (in operator) | Buffer layer | N/A (Arrow IPC if used) |
| Pull model | Callback-based (`DataAvailableCallback`) | Future-based (`ListenableFuture`) | Async stream (`RecordBatchStream`) |
| Distribution modes | Partitioned, Broadcast, Arbitrary | Partitioned, Broadcast, Arbitrary, ScaledWriter | Repartition via `RepartitionExec` |
| Backpressure | Two-level with hysteresis (90% continue) | Memory-tracking based (`revocableMemory`) | Tokio channel backpressure |
| Flush stagger | Randomized 70-120% per destination | None (deterministic flush) | N/A |
| Serde formats | Presto columnar, CompactRow, UnsafeRow | Presto columnar only | Arrow IPC |

### Exchange Client

| Dimension | Velox | Trino | DataFusion |
|-----------|-------|-------|------------|
| Transport abstraction | `ExchangeSource::Factory` (fully pluggable) | `HttpPageBufferClient` (HTTP-coupled) | No built-in exchange (host responsibility) |
| Fetch protocol | Two-phase: size probe → data transfer | Single-phase: request with fixed `maxResponseSize` | N/A |
| Client sharing | Single `ExchangeClient` per pipeline | Single `ExchangeClient` per exchange | N/A |
| Queue model | Shared `ExchangeQueue` with proportional wake | Per-destination buffers at source side | N/A |
| Backpressure to source | `ExchangeSource::pause()` | Implicit via HTTP flow control | N/A |
| Progressive fetching | `collectAny` on split + data futures | Waits for all sources registered | N/A |

### Connector SPI & Predicate Pushdown

| Dimension | Velox | Trino | DataFusion |
|-----------|-------|-------|------------|
| Pushdown negotiation | Single-shot (execution-time) | Multi-round (planning-time) | Single-shot (`TableProviderFilterPushDown` enum) |
| Filter representation | `SubfieldFilters` (nested `Subfield` → `Filter`) | `TupleDomain<ColumnHandle>` (flat column → domain) | `Expr` list (expressions) |
| Nested pushdown | Yes (`Subfield` paths: `a.b[1].c`) | No (flat columns only) | No (flat columns only) |
| Filter evaluation | SIMD-accelerated `testValues(batch)` at DWIO layer | Java `Domain.includes()` per-value | Arrow compute kernels |
| Unified filter+projection | `ScanSpec` tree (merged specification) | Separate `TupleDomain` + `ProjectionApplicationResult` | Separate `projection` indices + `filters` list |
| Dynamic filters | `addDynamicFilter()` with runtime reordering | `DynamicFilter` framework | Limited support |
| Filter adaptation | `ScanSpec::newRead()` reorders by runtime selectivity | None | None |
