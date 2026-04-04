# Trino Source Code Tracing Manifest (Hyper-Granular)

**Instructions for Claude:**
This is an atomic task list for analyzing the Trino source code. **DO NOT attempt to execute multiple tasks at once.** The user will specify a Task ID (e.g., "Claude, execute Task 1.1.A"). 
1. Read the specified source files using your CLI tools to understand the implementation.
2. **"Target Files" are a suggested starting point only** — they are not exhaustive. Automatically expand your research scope to any related dependencies, callers, implementations, or utilities that are relevant to the Focus. Follow the code wherever it leads.
3. Analyze the code deeply based on the specific "Focus" provided.
4. Generate the output exactly matching the `RESEARCH_TEMPLATE.md` structure.
5. Stop and wait for the user to verify the output and provide the next command.

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
**Objective:** Break down the cooperative multitasking engine to map it to a Rust Tokio async/await model.

* **Task 2.1.A: Task Lifecycle**
  * **Target Files:** `io.trino.execution.SqlTask`, `io.trino.execution.SqlTaskExecution`
  * **Focus:** How is a task initialized on a worker? What triggers its creation?
* **Task 2.1.B: Task State Machine**
  * **Target Files:** `io.trino.execution.TaskStateMachine`
  * **Focus:** Trace the state transitions (PLANNED, RUNNING, FLUSHING, FINISHED, FAILED, CANCELED) and the listeners attached to them.
* **Task 2.1.C: The Thread Pool Executor**
  * **Target Files:** `io.trino.execution.TaskExecutor`
  * **Focus:** Analyze the core thread pool. How does it accept tasks? How many threads are configured by default relative to CPU cores?
* **Task 2.1.D: Split Prioritization & Quanta**
  * **Target Files:** `io.trino.execution.executor.PrioritizedSplitRunner`, `io.trino.execution.executor.TaskHandle`
  * **Focus:** Analyze the scheduling logic. How does it determine which query fragment gets CPU time next? How does it measure execution time (quanta) to prevent starvation?
* **Task 2.2.A: Driver Initialization**
  * **Target Files:** `io.trino.operator.Driver`, `io.trino.operator.DriverContext`
  * **Focus:** How is a `Driver` instantiated with a pipeline of `Operator` objects? How does `DriverContext` track its lifespan?
* **Task 2.2.B: The Cooperative Yield Loop (Critical)**
  * **Target Files:** `io.trino.operator.Driver` (focus on `processFor()` and `processInternal()`)
  * **Focus:** This is the core engine loop. Document exactly what causes the `Driver` loop to suspend processing and yield control back to the `TaskExecutor`.

---

## Phase 3: The Operator Pipeline (Physical Plan Execution)
**Objective:** Trace the physical data processing nodes to understand streaming Volcano-style execution.

* **Task 3.1.A: Operator Contracts**
  * **Target Files:** `io.trino.operator.Operator`, `io.trino.operator.OperatorFactory`
  * **Focus:** Analyze the non-blocking state machine of an operator (`needsInput()`, `addInput()`, `getOutput()`, `isFinished()`).
* **Task 3.1.B: Operator Resource Context**
  * **Target Files:** `io.trino.operator.OperatorContext`
  * **Focus:** How does an operator interact with its context to track CPU time, wall time, and report its state?
* **Task 3.2.A: Simple Projection Pipeline**
  * **Target Files:** `io.trino.operator.ScanFilterAndProjectOperator`
  * **Focus:** Trace the simplest data flow: pulling from a source, applying a filter, and yielding a transformed `Page`.
* **Task 3.3.A: Hash Join - Building the Table**
  * **Target Files:** `io.trino.operator.HashBuilderOperator`, `io.trino.operator.join.JoinHash`
  * **Focus:** How does Trino ingest the "build" side of a join into memory? How is the hash table structured internally?
* **Task 3.3.B: Hash Join - Probing**
  * **Target Files:** `io.trino.operator.LookupJoinOperator`
  * **Focus:** How does the "probe" side stream through and match against the built hash table without blocking?
* **Task 3.4.A: Aggregation & State**
  * **Target Files:** `io.trino.operator.aggregation.AggregationOperator`, `io.trino.operator.aggregation.Accumulator`
  * **Focus:** How does an operator maintain group-by state across multiple `addInput()` calls?
* **Task 3.5.A: Disk Spilling Mechanism**
  * **Target Files:** `io.trino.spill.Spiller`, `io.trino.operator.SpillContext`
  * **Focus:** When memory limits are reached, how does an operator serialize its state to disk? Trace the spilling trigger mechanism.

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
