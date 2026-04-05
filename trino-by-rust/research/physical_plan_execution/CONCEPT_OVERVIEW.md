# Phase 3: Operator Internals and Data Processing Overview (Physical Plan Execution)

If Phase 2 is about how Trino orchestrates work across a cluster and schedules time on a CPU, Phase 3 is about what actually happens during that CPU time. This phase zooms into the very bottom of the execution hierarchy: the **Operators**. This is the physical data processing engine where bytes are read, transformed, joined, and aggregated.

## 1. The Volcano Engine and the Operator Contract

Trino uses a columnar, batch-oriented execution model inspired by the Volcano model, but adapted for cooperative, non-blocking multitasking.

* **The Non-Blocking State Machine:** In a traditional database, an operator might block a thread while waiting for disk I/O. In Trino, an `Operator` is strictly non-blocking. It operates via a rigid API contract:
    * `needsInput()`: Can this operator accept more data right now?
    * `addInput(Page)`: Push a batch of data into the operator.
    * `getOutput()`: Pull a transformed batch of data out of the operator.
    * `isFinished()`: Has this operator completely exhausted its work?
    * `isBlocked()`: If the operator cannot proceed (e.g., waiting for memory or a downstream buffer to clear), it returns a `ListenableFuture`. The Driver yields the thread until this future resolves.

    This contract is cooperative, not enforced — the `Operator` interface has no runtime checks that `needsInput()` is called before `addInput()`. An operator that blocks in `getOutput()` would freeze the Driver's thread. The Driver enforces the protocol through its call ordering in `processInternal()`.

* **The Resource Context:** Every operator is bound to an `OperatorContext`. This is its tether to reality, tracking exactly how many CPU nanoseconds it burns and how many bytes of memory it allocates, propagating these metrics up to the Driver and Task levels for real-time monitoring and resource enforcement. Memory is tracked in two pools: **user memory** (non-revocable, the operator needs it to function) and **revocable memory** (can be spilled to disk under pressure). This distinction is foundational to how spilling works.

* **The WorkProcessor Pattern:** Many key operators (including the scan-filter-project pipeline and the lookup join) don't implement the push-pull Operator contract directly. Instead, they are built as pull-based `WorkProcessor<Page>` chains — lazy, composable stream transformations with built-in yield and blocking support. A `WorkProcessorOperatorAdapter` bridges this internal pull model to the push-pull Operator interface the Driver expects: `addInput()` pushes data into a `PageBuffer`, and `getOutput()` advances the WorkProcessor pipeline which may pull from that buffer. The Driver sees a standard Operator, but inside, computation is structured as a lazy stream.

## 2. The Currency of Computation: Pages and Blocks

A critical distinction of Trino is that it does not process data row-by-row. It is a **columnar** engine in memory, which allows for tight loops and CPU cache efficiency.

* **Blocks:** A `Block` represents a single column of data. At the base level, `ValueBlock` implementations store raw primitive arrays — `LongArrayBlock` holds `long[]`, `VariableWidthBlock` holds a byte `Slice` plus an `int[]` offsets array. On top of this, two wrapper types provide compression and indirection without copying data:
    * **`DictionaryBlock`**: wraps any block with an `int[]` of position IDs, enabling sparse selection and deduplication with zero data copying. This is the primary mechanism by which filtered pages avoid copying column data.
    * **`RunLengthEncodedBlock`**: wraps a single value repeated N times (e.g., an all-null column), reducing an entire column to one value plus a count.

    The `Block` interface is sealed to exactly these three types (`ValueBlock | DictionaryBlock | RunLengthEncodedBlock`), enabling exhaustive pattern matching throughout the codebase.

* **Pages:** A `Page` is simply a collection of `Blocks` that share the same row count. It is the fundamental payload passed between Operators. When `addInput()` or `getOutput()` is called, they are passing entire `Pages`, not individual rows. Column selection (`Page.getColumns(...)`) is zero-copy — it creates a new Page by reshuffling Block references without copying any data.

* **The Zero-Copy Backbone:** Trino avoids data copying at nearly every level:
    * `Block.getRegion()` creates a view with an adjusted `arrayOffset` into the same backing arrays — O(1) time, O(1) memory.
    * `Block.getPositions()` defaults to wrapping the block in a `DictionaryBlock` — sparse filter results never copy the underlying column data.
    * Nested DictionaryBlocks are automatically flattened to prevent unbounded indirection.
    * `DictionaryAwarePageProjection` caches projected dictionary results across batches — if the same dictionary object appears in the next batch (common within an ORC stripe), the projection is skipped entirely.
    * Data is only physically copied when it reaches a `BlockBuilder` (e.g., to consolidate small pages via `MergePages`, or to construct hash table output).

## 3. Pipeline Anatomy: Stateless vs. Stateful

Inside a Driver, Operators are chained together. The complexity of this chain depends entirely on whether the operations need to "remember" anything.

### Stateless Operations (Stream-Through)

For simple pipelines (like `Scan -> Filter -> Project`), Trino fuses all three operations into a single `ScanFilterAndProjectOperator`. There is no separate FilterOperator or ProjectOperator — fusion eliminates intermediate Page allocations and allows the filter to control which positions the projections operate on.

Inside this fused operator, a `PageProcessor` does the actual work:
1. It applies filter predicates to produce a `SelectedPositions` selection vector (either a contiguous range for the fast path, or a sparse list of surviving positions).
2. It projects only the surviving positions in adaptive batches (self-tuning between 1 and 8192 positions based on output page size).
3. Between projection batches, it checks the `DriverYieldSignal` and voluntarily yields if the time quantum has expired, preserving partial results in a cache for the next call.

To maximize speed, Trino uses **runtime bytecode generation**. SQL expressions are compiled into optimized JVM bytecode at query planning time, with two parallel compilation paths: a **columnar path** (`ColumnarFilterCompiler`) that processes position arrays in bulk, and a **row-at-a-time fallback** (`PageFunctionCompiler`) for expressions too complex for columnar evaluation.

### Stateful Operations (Memory-Bound)

Complex SQL requires operations that must hold state across multiple Pages. These are the danger zones for memory constraints.

* **Aggregations:** Group-By clauses use `HashAggregationOperator` (not `AggregationOperator`, which handles only global aggregations without GROUP BY). A `GroupByHash` maps input rows to dense integer group IDs, with two specializations: `BigintGroupByHash` for single-BIGINT keys (direct primitive comparison) and `FlatGroupByHash` for general keys (SWAR byte-level probing, similar to Swiss Tables). Each `GroupedAccumulator` maintains per-group state indexed by group ID, avoiding per-group object allocation.

    The optimizer splits aggregation across the network: **PARTIAL** aggregation runs locally on each worker (map-side combine), and **FINAL** aggregation runs after the exchange (reduce-side). A `PartialAggregationController` monitors the unique-row-to-input ratio and **adaptively disables** partial aggregation when it's not reducing data volume (high-cardinality GROUP BY), periodically re-enabling to check if conditions improve.

* **Hash Joins:** A Hash Join fundamentally breaks a linear stream, forcing Trino to use two separate Pipelines:
    1. **The Build Pipeline:** Reads the planner-chosen smaller side of the join, feeding a `HashBuilderOperator` that accumulates pages into a `PagesIndex` and, on `finish()`, constructs an optimized hash table (`DefaultPagesHash` with open addressing and byte-level hash filtering, or `BigintPagesHash` for single-bigint keys). The built lookup source is handed to a `PartitionedLookupSourceFactory` which resolves a `SettableFuture` to unblock the probe side.
    2. **The Probe Pipeline:** `LookupJoinOperator` blocks until the build completes, then streams input pages through a `PageJoiner`. For each page, a `JoinProbe` eagerly batch-computes all hash lookups (vectorized before the per-row probe loop), then walks match chains with cooperative yielding. Output pages combine zero-copy probe columns (via `Block.getRegion()`) with deep-copied build columns.

## 4. Resilience: The Disk Spilling Mechanism

Because Trino processes data in memory, stateful operations (like massive Hash Joins or heavy Aggregations) can hit the worker's memory limits. Trino's safety valve is **Spilling**. It requires operators to explicitly opt in by reserving memory as "revocable" rather than "user" — only revocable memory can be reclaimed.

* **The Spill Trigger:** A `MemoryRevokingScheduler` monitors the global `MemoryPool` via a listener callback plus a 1-second poll. When pool utilization exceeds 90%, it traverses tasks (oldest first) and calls `OperatorContext.requestMemoryRevoking()` on operators holding revocable memory. This sets a flag and fires a listener that wakes the Driver. On its next processing loop, the Driver calls `startMemoryRevoke()` on the operator.

* **Two-Phase Revoke Protocol:** Spilling is asynchronous. `startMemoryRevoke()` submits serialization work to a dedicated thread pool and returns a `ListenableFuture`. While the future is pending, the Driver blocks all other Operator methods. Only after completion does the Driver call `finishMemoryRevoke()`, which performs non-thread-safe cleanup (clearing state, zeroing memory counters, transitioning internal state).

* **Operator-Specific Strategies:** Different operators handle spilling differently:
    * **Hash Join Build:** The `HashBuilderOperator` first tries **compaction** of its `PagesIndex` — if it shrinks by more than 20%, spilling is avoided entirely. If compaction is insufficient, pages are serialized to disk. After spill, new input goes directly to the spiller. On unspill (triggered by the probe side), a **checksum** verifies data integrity before rebuilding the hash table.
    * **Aggregation:** The `SpillableHashAggregationBuilder` spills sorted aggregation groups, then **immediately rebuilds a fresh empty builder** so the operator can continue accepting input while old data is still being written. On output, all spilled streams are **merge-sorted** by hash value via `MergeHashSort` and re-aggregated, since multiple spill rounds can produce overlapping group keys.

* **Serialization:** Pages are written to round-robin striped temp files via `FileSingleStreamSpiller`, with optional LZ4/ZSTD compression and optional AES encryption. The striping enables parallel read-back across multiple I/O threads.

## Summary: Connecting the Dots

1.  Inside a Driver, **Operators** form a non-blocking, cooperative state machine. Many key operators are internally built as lazy `WorkProcessor` streams, adapted to the push-pull Operator interface.
2.  Operators do not process rows; they shuttle columnar **Pages** and **Blocks** for maximum CPU efficiency. **DictionaryBlock** wrapping enables zero-copy filtering and projection caching throughout the pipeline.
3.  **Stateless operations** (scan, filter, project) are **fused** into a single operator. A `PageProcessor` applies compiled bytecode filters and projections in adaptive batches with cooperative yield support.
4.  **Stateful operators** accumulate state in memory. Joins use a two-pipeline build/probe architecture with vectorized hash lookups. Aggregations use specialized `GroupByHash` implementations with adaptive partial/final splitting across the network.
5.  Operators opt into spilling by reserving **revocable memory**. A `MemoryRevokingScheduler` triggers a two-phase async protocol. Hash joins try compaction first; aggregations spill sorted groups and merge-sort on read-back.
