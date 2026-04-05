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

## Phase 1: The Foundation & Memory SPI (Data Layout)
**Objective:** Understand raw columnar data representation before it enters the execution engine. This is essential for mapping to Apache Arrow.

* **Task 1.1.A: The `Slice` Memory Wrapper**
  * **Target Files:** `io.airlift.slice.Slice`, `io.airlift.slice.Slices`
  * **Focus:** How does Trino wrap off-heap `byte[]` arrays or direct memory buffers? Analyze bounds checking and unsafe memory access patterns.
* **Task 1.1.B: The `Block` & `BlockBuilder` Interfaces**
  * **Target Files:** `io.trino.spi.block.Block`, `io.trino.spi.block.BlockBuilder`
  * **Focus:** Analyze the contract for columnar data. How are nulls (validity bitmaps) conceptually represented at this interface level?
* **Task 1.1.C: Variable-Width Storage**
  * **Target Files:** `io.trino.spi.block.VariableWidthBlock`, `io.trino.spi.block.VariableWidthBlockBuilder`
  * **Focus:** Trace exactly how variable-length data (like VARCHAR) calculates its `offsets` array and manages its underlying `Slice` memory.
* **Task 1.1.D: Primitive Storage & Dictionary Compression**
  * **Target Files:** `io.trino.spi.block.LongArrayBlock`, `io.trino.spi.block.DictionaryBlock`
  * **Focus:** Compare primitive storage to variable-width. How does `DictionaryBlock` map an array of `ids` to a separate dictionary `Block`?
* **Task 1.2.A: The `Page` Construct**
  * **Target Files:** `io.trino.spi.Page`, `io.trino.spi.PageBuilder`
  * **Focus:** How are multiple `Blocks` bound together into a `Page` (representing a batch of rows)? Trace the `getSizeInBytes()` and `getRetainedSizeInBytes()` calculations.
* **Task 1.3.A: The Ingestion Boundary**
  * **Target Files:** `io.trino.spi.connector.ConnectorPageSource`
  * **Focus:** Understand the interface workers use to pull `Page` batches from underlying storage plugins.

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

# Phase 3: Operator Internals and Data Processing (Physical Plan Execution)
**Objective:** Trace the physical data processing nodes to understand Trino's columnar, streaming, Volcano-style execution engine. This phase ignores scheduling and focuses purely on how data is transformed in memory.

## Task 3.1: The Operator Engine Fundamentals
Before looking at specific operations, we need to understand the contract every Operator must follow to allow cooperative multitasking.

* **Task 3.1.A: The Operator State Machine**
    * **Target Files:** `io.trino.operator.Operator`, `io.trino.operator.OperatorFactory`
    * **Focus:** Analyze the non-blocking Volcano model. Trace the exact sequence of `needsInput()`, `addInput()`, `getOutput()`, `isFinished()`, and `isBlocked()`. How does an Operator tell the Driver "I need more data" vs. "I need to wait for memory"?
* **Task 3.1.B: Operator Resource Context**
    * **Target Files:** `io.trino.operator.OperatorContext`
    * **Focus:** How does an individual Operator track its CPU time, wall time, and memory allocations? How do these localized metrics bubble up to the `DriverContext`?

## Task 3.2: The Data Payload (Pages & Blocks)
Operators don't process rows; they process columnar batches. Understanding these data structures is mandatory for tracing Operator logic.

* **Task 3.2.A: The Columnar Memory Model**
    * **Target Files:** `io.trino.spi.Page`, `io.trino.spi.block.Block`
    * **Focus:** How is a `Page` structured? Trace how a `Block` represents a single column of data in memory (e.g., `DictionaryBlock`, `RunLengthEncodedBlock`, `VariableWidthBlock`). How do Operators read from these structures without copying data?

## Task 3.3: Stateless & Linear Pipelines
Tracing the simplest data flow where one input page directly results in one or more output pages.

* **Task 3.3.A: Simple Projection & Filtering**
    * **Target Files:** `io.trino.operator.ScanFilterAndProjectOperator`, `io.trino.operator.project.PageProcessor`
    * **Focus:** Trace a raw block of data coming from a connector, passing through a filter, and yielding a transformed `Page`. How does Trino compile these expressions into bytecode for faster execution?

## Task 3.4: Stateful & Complex Pipelines
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

## Task 3.5: Resilience and Memory Management
What happens when a single Operator demands more memory than the worker can provide?

* **Task 3.5.A: The Disk Spilling Mechanism**
    * **Target Files:** `io.trino.spill.Spiller`, `io.trino.operator.SpillContext`
    * **Focus:** Trace the spilling trigger mechanism. When memory limits are reached, how does a stateful Operator (like an Aggregation or Hash Join) pause, serialize its state to disk, free up RAM, and later read it back?

---

## Phase 4: Network & Data Exchange (Shuffle)
**Objective:** Understand how workers shuffle data over the wire, defining the API boundaries for the Rust worker.

* **Task 4.1.A: The Exchange Client**
  * **Target Files:** `io.trino.operator.exchange.ExchangeClient`
  * **Focus:** How does a worker manage connections to multiple upstream workers? How does it buffer incoming data?
* **Task 4.1.B: HTTP Polling**
  * **Target Files:** `io.trino.operator.HttpPageBufferClient`
  * **Focus:** Trace the actual HTTP request cycle. How does it handle retries, backoff, and deserializing the wire format back into a `Page`?
* **Task 4.2.A: Holding Results (Output Buffers)**
  * **Target Files:** `io.trino.execution.buffer.OutputBuffer`, `io.trino.execution.buffer.ClientBuffer`
  * **Focus:** How does a `Driver` write its final `Page` outputs? How are they queued in memory waiting for downstream workers?
* **Task 4.2.B: Partitioning Data**
  * **Targetাক্ষ Files:** `io.trino.execution.buffer.PartitionedOutputBuffer`
  * **Focus:** Trace the hashing logic. How does it ensure specific rows are routed to the correct downstream task (e.g., during a distributed Group By)?
* **Task 4.2.C: The Network API Endpoint**
  * **Target Files:** `io.trino.server.TaskResource`
  * **Focus:** Analyze the REST endpoints. What URL schemas and HTTP verbs are used by downstream workers to pull data from the `OutputBuffer`?

---

## Phase 5: Memory Tracking & Arbitration (Memory Management)
**Objective:** Map out the strict hierarchical memory accounting system to replicate it in Rust.

* **Task 5.1.A: Global Memory Pools**
  * **Target Files:** `io.trino.memory.MemoryPool`
  * **Focus:** How does the worker track total JVM memory? Analyze the concept of reserved vs. general pools (if applicable to the current Trino version).
* **Task 5.1.B: Query Level Tracking**
  * **Target Files:** `io.trino.memory.QueryContext`
  * **Focus:** How does the worker isolate memory tracking per query to ensure one heavy query doesn't crash the worker?
* **Task 5.1.C: The Tracking Tree**
  * **Target Files:** `io.trino.memory.context.MemoryTrackingContext`
  * **Focus:** Analyze the hierarchical tree structure (System -> Query -> Task -> Pipeline -> Driver -> Operator).
* **Task 5.2.A: Operator Allocation Reporting**
  * **Target Files:** `io.trino.memory.context.LocalMemoryContext`
  * **Focus:** Trace the exact mechanism an `Operator` uses to report an allocation (e.g., `setBytes()`). What happens synchronously when this call exceeds a limit?
* **Task 5.2.B: Memory Revocation**
  * **Target Files:** `io.trino.memory.context.MemoryRevokingContext`
  * **Focus:** How does the engine signal an operator that it needs to free memory (triggering the spilling mechanism identified in Task 3.5.A)?
