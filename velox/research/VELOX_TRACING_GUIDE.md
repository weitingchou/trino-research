# Velox Source Code Tracing Manifest (Hyper-Granular)

**Instructions for Claude:**
This is an atomic task list for analyzing the Meta Velox source code. **DO NOT attempt to execute multiple tasks at once.** The user will specify a Task ID (e.g., "Claude, execute Task 1.1.A").
1. Read the specified source files using your CLI tools to understand the implementation.
2. **"Target Files" are a suggested starting point only** — they are not exhaustive. Automatically expand your research scope to any related dependencies, callers, implementations, or utilities that are relevant to the Focus. Follow the code wherever it leads.
3. Analyze the code deeply based on the specific "Focus" provided.
4. **Provide code snippets** for key concepts in your explanation. Quote relevant source code directly to support your analysis rather than describing it abstractly.
5. Generate the output exactly matching the `RESEARCH_TEMPLATE.md` structure.
6. Stop and wait for the user to verify the output and provide the next command.

---

## Phase 1: Foundation Tracing Guide (Memory Layout)
**Objective:** Understand Velox's physical memory layout in C++ to contrast with Trino's Java-based `Slice`/`Block` and DataFusion's Arrow `Buffer`/`Array`.

### Task 1.1: Buffer
* **Task 1.1: The `Buffer` Memory Wrapper**
  * **Target Files:** `velox/buffer/Buffer.h`, `velox/buffer/BufferView.h`
  * **Focus:** Analyze how `velox::Buffer` wraps contiguous raw memory. Trace its use of `std::shared_ptr` for reference counting (C++ ownership) vs. Trino's GC model. How are allocations requested from the `MemoryPool`?

### Task 1.2: Vector (The Columnar Accessor)
* **Task 1.2: Columnar Construction and Bitmaps (`BaseVector`)**
  * **Target Files:** `velox/vector/BaseVector.h`, `velox/vector/FlatVector.h`, `velox/vector/DictionaryVector.h`
  * **Focus:** Analyze `BaseVector`. Trace how `FlatVector` manages its data `Buffer` and its `nulls` `Buffer` (bitmap). Trace `DictionaryVector` to understand how it uses an `indices` buffer to achieve zero-copy wrapping, comparing it directly to Trino's `DictionaryBlock`.

### Task 1.3: RowVector (The Envelope)
* **Task 1.3: The `RowVector` Container**
  * **Target Files:** `velox/vector/ComplexVector.h` (focus on `RowVector`)
  * **Focus:** Analyze `RowVector`. How does it group multiple `BaseVector` columns together? Compare its structural overhead and zero-copy projection capabilities (`SelectivityVector`) to Trino's `Page` and DataFusion's `RecordBatch`.

### Task 1.4: Physical Data Mapping
* **Task 1.4: Physical Data Mapping (The Ingestion Bridge)**
  * **Target Files:** `velox/dwio/parquet/reader/ParquetReader.h`, `velox/dwio/common/Reader.h`
  * **Focus:** Trace the path of an external read. How does Velox's DWIO (Data Warehouse I/O) framework pull bytes from storage and decode Parquet into a `RowVector`? Trace how memory is pre-allocated from the pool during decoding.

---

## Phase 2: Worker Scheduling & The Driver (Execution Model)
**Objective:** Trace how Velox implements cooperative multitasking and driver scheduling in C++, comparing its `Driver` and `Task` structures directly against Trino's JVM-based equivalents.

### Task 2.1: Task Architecture
* **Task 2.1: Task Creation & Pipeline Instantiation**
  * **Target Files:** `velox/exec/Task.h`, `velox/exec/Task.cpp`, `velox/core/PlanNode.h`
  * **Focus:** How does Velox convert a physical `PlanNode` tree into executable pipelines? Trace the initialization of a `Task`. How does it decide the degree of concurrency and spawn multiple `Driver`s for a given pipeline?

### Task 2.2: The Scheduling Engine
* **Task 2.2: The Thread Pool and Execution Queue**
  * **Target Files:** `velox/exec/Task.cpp` (focus on `Task::start()`), standard `folly::Executor` integrations.
  * **Focus:** Velox relies heavily on Meta's `folly` library. Trace how `Driver` instances are queued into a `folly::Executor` (or similar thread pool). How does Velox ensure CPU fairness between concurrent tasks?

### Task 2.3: Driver Lifecycle
* **Task 2.3.A: Driver Initialization & Pipeline Plumbing**
  * **Target Files:** `velox/exec/Driver.h`, `velox/exec/DriverCtx.h`
  * **Focus:** How is a `Driver` instantiated with a sequence of `Operator`s? Trace how `DriverCtx` links the executing thread back to the `Task`'s memory pool and metrics.
* **Task 2.3.B: The Cooperative Yield Loop**
  * **Target Files:** `velox/exec/Driver.cpp` (focus on the main `Driver::runInternal()` or `enqueue()` loops).
  * **Focus:** This is the core engine heart. Trace the cooperative loop. Exactly what causes a C++ `Driver` to yield the thread (`BlockingReason`)? How does it resume execution via futures/promises when I/O or memory becomes available?

---

## Phase 3: Operator Internals and Data Processing (Physical Plan Execution)
**Objective:** Trace the push-pull execution model. Compare Velox's pre-compiled vectorized expression evaluation to Trino's JIT compilation and DataFusion's Arrow compute kernels.

### Task 3.1: The Operator Contract
* **Task 3.1: The Operator State Machine**
  * **Target Files:** `velox/exec/Operator.h`, `velox/exec/Operator.cpp`
  * **Focus:** Analyze the Volcano-style contract. Trace `needsInput()`, `addInput()`, `getOutput()`, `isFinished()`, and `isBlocked()`. Document how strictly this mirrors Trino's Java interface and how blocking signals are passed as `folly::Future`.

### Task 3.2: Physical Expressions & Compute
* **Task 3.2: Vectorized Expression Evaluation**
  * **Target Files:** `velox/expression/Expr.h`, `velox/expression/EvalCtx.h`, `velox/functions/`
  * **Focus:** Trace how a `PlanNode` expression is compiled into an `Expr` tree. Because Velox does not JIT compile, trace the vectorized loop inside `Expr::eval()`. How does it use `SelectivityVector` to skip evaluating nulls or filtered rows across a batch?

### Task 3.3: Stateless Pipelines
* **Task 3.3: Stateless Operations (Filter & Project)**
  * **Target Files:** `velox/exec/FilterProject.h`, `velox/exec/FilterProject.cpp`
  * **Focus:** Trace the `addInput` and `getOutput` methods. How does it apply the `Expr` evaluator to an incoming `RowVector` and produce an output `RowVector` without row-by-row iteration overhead?

### Task 3.4: Stateful Pipelines
* **Task 3.4: Complex Pipelines (Hash Join)**
  * **Target Files:** `velox/exec/HashBuild.h`, `velox/exec/HashProbe.h`, `velox/exec/HashTable.h`
  * **Focus:** Trace the split pipeline execution of Hash Join. How does `HashBuild` ingest vectors into the `HashTable`? How does `HashProbe` block until the build is complete, and then stream matches?

### Task 3.5: Disk Spilling
* **Task 3.5: Disk Spilling Mechanism**
  * **Target Files:** `velox/exec/Spiller.h`, `velox/exec/Spill.h`
  * **Focus:** Trace the triggering mechanism for spilling. When memory limits are reached, how does an operator serialize its state to disk using Velox's serialization format?

---

## Phase 4: Communication Interfaces (Data Exchange)
**Objective:** Map out how Velox interacts with external storage (Connectors) and shuffles data locally and over the network, noting its design as an embeddable library.

### Task 4.1: The Control Plane (Host Interface)
* **Task 4.1: The Host Control Boundary**
  * **Target Files:** `velox/core/PlanNode.h`, `velox/exec/Task.h` (focus on `Task::updateBroadcastOutputBuffers` or split additions).
  * **Focus:** Velox does not have a REST server; the host application (e.g., Prestissimo) feeds it. Trace the C++ API boundary where the host pushes `PlanNode` fragments and incrementally adds `exec::Split`s to a running `Task`.

### Task 4.2: The Data Plane (Shuffle)
* **Task 4.2.A: Result Buffering (Local/Network Exchange Server)**
  * **Target Files:** `velox/exec/OutputBufferManager.h`, `velox/exec/PartitionedOutputBufferManager.h`
  * **Focus:** Trace how `RowVector` objects are serialized and queued in output buffers. How does the hashing logic distribute rows to different downstream partition queues?
* **Task 4.2.B: The Exchange Client (Fetching Data)**
  * **Target Files:** `velox/exec/Exchange.h`, `velox/exec/Task.h`
  * **Focus:** Trace how a downstream Velox task fetches data. Look at the `Exchange` operator. Since network RPC is host-dependent, trace the boundary where Velox expects the host to populate its internal exchange queues.

### Task 4.3: The Storage Plane
* **Task 4.3: The Connector SPI & Predicate Pushdown**
  * **Target Files:** `velox/connectors/Connector.h`, `velox/connectors/hive/HiveConnector.h`
  * **Focus:** Trace the `Connector` interface. How does Velox pass subfield projections and complex filter predicates (via `SubfieldFilters`) down to the `DataSource`? Compare this to Trino's pushdown channels.

---

## Phase 5: Memory Tracking & Arbitration (Memory Management)
**Objective:** Map Velox's explicit C++ memory accounting system and its global arbitration mechanisms, comparing it to both Trino's hierarchical tree and DataFusion's RAII model.

### Task 5.1: Memory Pools and the Tracking Tree
* **Task 5.1: The `MemoryPool` Hierarchy**
  * **Target Files:** `velox/common/memory/MemoryPool.h`, `velox/common/memory/MemoryManager.h`
  * **Focus:** Analyze how Velox structures memory pools (Root -> Query -> Task -> Node). Trace how bytes are reserved and tracked. How does it handle thread-safe aggregation of memory usage statistics without high lock contention?

### Task 5.2: Memory Allocation
* **Task 5.2: Allocation Mechanisms**
  * **Target Files:** `velox/common/memory/MemoryAllocator.h`, `velox/common/memory/MmapAllocator.h`
  * **Focus:** Trace the actual byte allocation mechanisms. How does Velox differentiate between contiguous allocations and non-contiguous `Allocation` chunks? How are these chunks provided to `Buffer`s?

### Task 5.3: Memory Arbitration & OOM Handling
* **Task 5.3: The `MemoryArbitrator` (Spill Triggering)**
  * **Target Files:** `velox/common/memory/MemoryArbitrator.h`, `velox/common/memory/SharedArbitrator.h`
  * **Focus:** Trace the exact path when a `MemoryPool` exceeds its limit. How does the `MemoryArbitrator` coordinate across the entire process to find victim queries/tasks? How does it signal operators to pause execution and invoke the `Spiller`? Compare this to Trino's `MemoryRevokingScheduler`.
