# Trino Source Code Tracing Manifest (Hyper-Granular)

**Instructions for Claude:**
This is an atomic task list for analyzing the Trino source code. **DO NOT attempt to execute multiple tasks at once.** The user will specify a Task ID (e.g., "Claude, execute Task 1.1.A"). 
1. Read the specified source files using your CLI tools to understand the implementation.
2. **"Target Files" are a suggested starting point only** — they are not exhaustive. Automatically expand your research scope to any related dependencies, callers, implementations, or utilities that are relevant to the Focus. Follow the code wherever it leads.
3. Analyze the code deeply based on the specific "Focus" provided.
4. **Provide code snippets** for key concepts in your explanation. Quote relevant source code directly to support your analysis rather than describing it abstractly.
5. Generate the output exactly matching the `RESEARCH_TEMPLATE.md` structure.
6. Stop and wait for the user to verify the output and provide the next command.

---

## Phase 1: Foundation Tracing Guide (Memory Layout)

### Slice
* **Task 1.1.A: The `Slice` Memory Interface & Metadata**
    * **Target Files:** `io.airlift.slice.Slice`, `io.airlift.slice.Slices`
    * **Focus:** Analyze the internal metadata of a `Slice` (typically a base object reference, an address offset, and a size). Trace the APIs that utilize `Unsafe` memory access. Contrast `HeapSlice` with `DirectSlice`.

### Block
* **Task 1.2.A: The `Block` Interface & Internal Metadata**
    * **Target Files:** `io.trino.spi.block.Block`, `io.trino.spi.block.VariableWidthBlock`, `io.trino.spi.block.LongArrayBlock`
    * **Focus:** Open `VariableWidthBlock` and trace its internal fields (`Slice`, `int[] offsets`, `boolean[] valueIsNull`). Analyze the read-only contract of the `Block` interface. Specifically trace the `getRegion()` method to understand how zero-copy columnar slicing is implemented without duplicating the underlying memory.

### Page
* **Task 1.3.A: The `Page` Interface & Zero-Copy Mutations**
    * **Target Files:** `io.trino.spi.Page`
    * **Focus:** Inspect the internal fields of a `Page` (`Block[]`, `positionCount`). Analyze `Page` as a passive data container. Trace how `Page.getColumns()` and `Page.prependColumn()` create new `Page` instances via shallow copies of `Block` references. 

### Physical Data Mapping
* **Task 1.4.A: Physical Data Mapping (The S3 Bridge)**
    * **Target Files:** `io.trino.spi.connector.ConnectorPageSource`, `io.trino.parquet.ParquetDataSource` (or ORC equivalent), `io.trino.plugin.iceberg.IcebergPageSource`
    * **Focus:** Trace the path of an S3 read. How does the data source pull a range of bytes from the object store into a `Slice`? How does the format reader extract values to populate the `BlockBuilder` and ultimately emit a `Page`?

---

## Phase 2: Worker Scheduling & The Driver (Execution Model)
**Objective:** Map the hierarchical relationship between Tasks and Drivers and trace their full lifecycles to design a compatible async/await model.

### 2.1 Task Architecture & Lifecycle
* **Task 2.1.A: Task Creation & Resource Wiring**
  * **Target Files:** `io.trino.execution.SqlTask`, `io.trino.execution.SqlTaskExecution`, `io.trino.execution.TaskManagerConfig`
  * **Focus:** How is a `SqlTask` initialized when a `TaskUpdateRequest` hits the worker? What triggers its creation? Trace how it wires up the `OutputBuffer`, `QueryContext`, and `TaskStateMachine` at birth.
* **Task 2.1.B: The Task State Machine & Terminal Transitions**
  * **Target Files:** `io.trino.execution.TaskStateMachine`
  * **Focus:** Trace the full state transitions: `PLANNED` -> `RUNNING` -> `FLUSHING` -> `FINISHED` (`FAILED`, `CANCELED`). Exactly what logic determines the transition from `FLUSHING` to `FINISHED`?
* **Task 2.1.C: The Task-Driver Relationship (Concurrency Scaling)**
  * **Target Files:** `io.trino.execution.SqlTaskExecution`, `io.trino.operator.PipelineFactory`, `io.trino.operator.PipelineContext`
  * **Focus:** Analyze how a single `SqlTask` creates multiple `PipelineContexts`. Trace the logic that decides how many `Driver` instances to spawn for a single pipeline based on available splits and concurrency settings.

### 2.2 The Scheduling Engine
* **Task 2.2.A: The Thread Pool Executor**
  * **Target Files:** `io.trino.execution.executor.TaskExecutor`
  * **Focus:** Analyze the core thread pool (Runner threads). How does it accept tasks? How does it manage the priority queue of `PrioritizedSplitRunner` objects?
* **Task 2.2.B: Split Prioritization & Time Quanta**
  * **Target Files:** `io.trino.execution.executor.PrioritizedSplitRunner`, `io.trino.execution.executor.TaskHandle`
  * **Focus:** How does the executor track CPU "quanta" (time slices)? Trace how it calculates the priority of a split to ensure fair scheduling across multiple queries.

### 2.3 Driver Lifecycle & The Execution Loop
* **Task 2.3.A: Driver Initialization & Pipeline Plumbing**
  * **Target Files:** `io.trino.operator.Driver`, `io.trino.operator.DriverContext`
  * **Focus:** How is a `Driver` instantiated as a sequence of `Operator` instances? Trace how the `DriverContext` is used to track memory and CPU at the driver level.
* **Task 2.3.B: The Cooperative Yield Loop (The Engine Heart)**
  * **Target Files:** `io.trino.operator.Driver` (focus on `processFor()` and `processInternal()`)
  * **Focus:** This is the core cooperative loop. Document exactly what causes a `Driver` to suspend (yield) and return a `ListenableFuture` to the executor (e.g., blocked on an operator, or quantum exhausted).
* **Task 2.3.C: Driver Termination & Cleanup**
  * **Target Files:** `io.trino.operator.Driver` (focus on `close()` and `isFinished()`)
  * **Focus:** When is a `Driver` considered done? Trace the cleanup path: closing operators, releasing memory back to the `PipelineContext`, and signaling the `TaskStateMachine`.

---

## Phase 3: Operator Internals and Data Processing (Physical Plan Execution)
**Objective:** Trace the physical data processing nodes to understand Trino's columnar, streaming, Volcano-style execution engine. This phase ignores scheduling and focuses purely on how data is transformed in memory.

### Task 3.1: The Operator Engine Fundamentals
Before looking at specific operations, we need to understand the contract every Operator must follow to allow cooperative multitasking.

* **Task 3.1.A: The Operator State Machine**
    * **Target Files:** `io.trino.operator.Operator`, `io.trino.operator.OperatorFactory`
    * **Focus:** Analyze the non-blocking Volcano model. Trace the exact sequence of `needsInput()`, `addInput()`, `getOutput()`, `isFinished()`, and `isBlocked()`. How does an Operator tell the Driver "I need more data" vs. "I need to wait for memory"?
* **Task 3.1.B: Operator Resource Context**
    * **Target Files:** `io.trino.operator.OperatorContext`
    * **Focus:** How does an individual Operator track its CPU time, wall time, and memory allocations? How do these localized metrics bubble up to the `DriverContext`?
* **Task 3.1.C: The Complete Operator Catalog**
    * **Target Files:** `io.trino.operator.*Operator.java`, `io.trino.operator.*OperatorFactory` (all ~43 Operator implementations in `io.trino.operator/` and sub-packages)
    * **Focus:** Enumerate every `Operator` implementation in Trino 480. For each operator, classify it along these dimensions: (1) **Category** — source, sink, transform, join, aggregation, window, exchange, metadata, or DML; (2) **Pipeline role** — does it start a pipeline (source), end a pipeline (sink), or sit in the middle (transform)? (3) **State model** — stateless (streaming, one page in / one page out), stateful-blocking (accumulates all input before producing output), or stateful-streaming (maintains state but can produce output incrementally)? (4) **Spill support** — does this operator implement `startMemoryRevoke()`/`finishMemoryRevoke()`? (5) **Brief function** — one-sentence description of what the operator does. The output should be a comprehensive reference table covering every operator, grouped by category.

### Task 3.2: The Data Payload (Pages & Blocks)
Operators don't process rows; they process columnar batches. Understanding these data structures is mandatory for tracing Operator logic.

* **Task 3.2.A: The Columnar Memory Model**
    * **Target Files:** `io.trino.spi.Page`, `io.trino.spi.block.Block`
    * **Focus:** How is a `Page` structured? Trace how a `Block` represents a single column of data in memory (e.g., `DictionaryBlock`, `RunLengthEncodedBlock`, `VariableWidthBlock`). How do Operators read from these structures without copying data?

### Task 3.3: Stateless & Linear Pipelines
Tracing the simplest data flow where one input page directly results in one or more output pages.

* **Task 3.3.A: Simple Projection & Filtering**
    * **Target Files:** `io.trino.operator.ScanFilterAndProjectOperator`, `io.trino.operator.project.PageProcessor`
    * **Focus:** Trace a raw block of data coming from a connector, passing through a filter, and yielding a transformed `Page`. How does Trino compile these expressions into bytecode for faster execution?

### Task 3.4: Stateful & Complex Pipelines
Tracing operations that must hold state across multiple `addInput()` calls, and operations that bridge multiple Pipelines.

* **Task 3.4.A: Hash Join - The Build Pipeline**
    * **Target Files:** `io.trino.operator.HashBuilderOperator`, `io.trino.operator.join.JoinHash`
    * **Focus:** How does Trino ingest the entire "build" side of a join into memory? Trace the internal structure of the hash table. How does it handle memory limits before the probe phase begins?
* **Task 3.4.B: Hash Join - The Probe Pipeline**
    * **Target Files:** `io.trino.operator.LookupJoinOperator`
    * **Focus:** How does the "probe" side stream through and match against the built hash table without blocking? How are the output pages constructed from the matched indices?
* **Task 3.4.C: Aggregation & Accumulators**
    * **Target Files:** `io.trino.operator.aggregation.AggregationOperator`, `io.trino.operator.aggregation.Accumulator`
    * **Focus:** Trace a Group-By operation. How does the operator maintain state across multiple `addInput()` calls? Differentiate between partial (local) aggregations and final (global) aggregations.

### Task 3.5: Resilience and Memory Management
What happens when a single Operator demands more memory than the worker can provide?

* **Task 3.5.A: The Disk Spilling Mechanism**
    * **Target Files:** `io.trino.spill.Spiller`, `io.trino.operator.SpillContext`
    * **Focus:** Trace the spilling trigger mechanism. When memory limits are reached, how does a stateful Operator (like an Aggregation or Hash Join) pause, serialize its state to disk, free up RAM, and later read it back?

---

## Phase 4: Communication Interfaces (Data Exchange)
**Objective:** Understand the three critical interface boundaries of a Trino Worker: Control (Coordinator), Data (Shuffle), and Storage (Connector). This defines the full API surface required for a compatible worker implementation.

### Task 4.1: The Control Plane (Coordinator ↔ Worker)
How the "Brain" manages the "Muscle." This is primarily REST/JSON based.

* **Task 4.1.A: Task Lifecycle Management**
    * **Target Files:** `io.trino.server.TaskResource`, `io.trino.execution.SqlTask`
    * **Focus:** Analyze the REST endpoints for creating, updating, and deleting tasks. Trace the `TaskUpdateRequest` JSON structure. How does the worker receive the "Physical Plan" from the coordinator?
* **Task 4.1.B: Status & Heartbeat Reporting**
    * **Target Files:** `io.trino.execution.TaskStateMachine`, `io.trino.server.TaskStatus`
    * **Focus:** How does the worker report its health, memory usage, and execution progress back to the coordinator? Trace the asynchronous long-polling mechanism used to update task status.
* **Task 4.1.C: Dynamic Filter Coordination**
    * **Target Files:** `io.trino.server.DynamicFilterService`
    * **Focus:** Trace how a worker "collects" a filter (from a join build side) and sends it back to the coordinator, and how the coordinator then "broadcasts" it to other workers to prune scans.

### Task 4.2: The Data Plane (Worker ↔ Worker Shuffle)
How data moves between workers during a query. This is high-volume HTTP streaming.

* **Task 4.2.A: Result Buffering (The Server Side)**
    * **Target Files:** `io.trino.execution.buffer.OutputBuffer`, `io.trino.execution.buffer.PagesSerde`
    * **Focus:** How are `Page` objects serialized into the wire format? Trace the logic in `PartitionedOutputBuffer` that hashes rows to decide which downstream worker gets which data.
* **Task 4.2.B: The Exchange Client (The Client Side)**
    * **Target Files:** `io.trino.operator.exchange.ExchangeClient`, `io.trino.operator.HttpPageBufferClient`
    * **Focus:** How does a worker pull data from multiple upstream workers? Trace the HTTP request cycle: headers used for flow control (`X-Trino-Buffer-Remaining-Bytes`), retries, and backoff.
* **Task 4.2.C: The Coordinator as the Final Consumer**
    * **Target Files:** `io.trino.server.StatementResource`, `io.trino.execution.DataPuller` (or how `Query` manages the final exchange)
    * **Focus:** Trace how the coordinator fetches the final result set from the root task. How does the root worker's `OutputBuffer` differ (if at all) from an intermediate shuffle buffer? How are internal `Pages` finally converted into the client-facing format?

### Task 4.3: The Storage Plane (Worker ↔ Connector)
How the worker interacts with the SPI (Service Provider Interface) to get raw data.

* **Task 4.3.A: The Page Source (Reading)**
    * **Target Files:** `io.trino.spi.connector.ConnectorPageSource`, `io.trino.connector.hive.HivePageSource` (as an example)
    * **Focus:** Trace the boundary where a "Split" is converted into a stream of `Page` objects. How does the worker pass column projections and predicate pushdowns into the connector?
* **Task 4.3.B: The Page Sink (Writing)**
    * **Target Files:** `io.trino.spi.connector.ConnectorPageSink`
    * **Focus:** Trace how data is sent to a connector for `INSERT` or `CREATE TABLE AS` operations. How does the worker handle transaction commits/rollbacks via the connector?

---

## Phase 5: Memory Tracking & Arbitration (Memory Management)
**Objective:** Map out the strict hierarchical memory accounting system to understand flow control, spilling triggers, and query arbitration. This is vital for replicating safe resource management in a Rust worker.

### The Global Pool and the Tracking Tree 
* **Task 5.1.A: The Global Pool and the Tracking Tree**
    * **Target Files:** `io.trino.memory.MemoryPool`, `io.trino.memory.QueryContext`, `io.trino.memory.context.MemoryTrackingContext`
    * **Focus:** Analyze how the single global `MemoryPool` is constructed. Trace the creation of the context tree (Query -> Task -> Pipeline -> Driver -> Operator). How do deltas (additions/subtractions of memory) propagate up this tree without causing severe lock contention?

### Operator Allocation and Blocking 
* **Task 5.2.A: Operator Allocation and Blocking**
    * **Target Files:** `io.trino.memory.context.LocalMemoryContext`, `io.trino.memory.context.UserMemoryContext`
    * **Focus:** Trace the exact mechanism an Operator uses to report an allocation: `LocalMemoryContext.setBytes()`. What is the exact execution path when this call results in a limit being exceeded? Trace how the resulting `ListenableFuture` is passed back to the Driver yield loop.

### Revocable Memory and Spilling
* **Task 5.3.A: Revocable Memory and Spilling**
    * **Target Files:** `io.trino.memory.context.RevocableMemoryContext`, `io.trino.execution.MemoryRevokingScheduler`
    * **Focus:** Look at `HashBuilderOperator` or `HashAggregationOperator`. Trace how they allocate memory via the `RevocableMemoryContext`. How does the `MemoryRevokingScheduler` monitor the global pool, select a victim task, and invoke the asynchronous revoking process?

### Cluster Arbitration and the OOM Killer
* **Task 5.4.A: Cluster Arbitration and the OOM Killer**
    * **Target Files:** `io.trino.memory.ClusterMemoryManager`, `io.trino.memory.LowMemoryKiller`
    * **Focus:** This requires jumping to the Coordinator. How does the coordinator aggregate the `MemoryInfo` from all workers? Trace the logic in `ClusterMemoryManager` when a worker reports a "blocked" state. How does the `LowMemoryKiller` select a victim, and how is the "kill" signal propagated back down the Control Plane (Phase 4) to the workers?
