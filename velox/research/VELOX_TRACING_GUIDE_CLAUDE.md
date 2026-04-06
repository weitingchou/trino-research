# Velox Source Code Tracing Manifest (Hyper-Granular)

**Instructions for Claude:**
This is an atomic task list for analyzing the Meta Velox source code (C++). **DO NOT attempt to execute multiple tasks at once.** The user will specify a Task ID (e.g., "Claude, execute Task 1.1.A").
1. Read the specified source files using your CLI tools to understand the implementation.
2. **"Target Files" are a suggested starting point only** — they are not exhaustive. Automatically expand your research scope to any related dependencies, callers, implementations, or utilities that are relevant to the Focus. Follow the code wherever it leads.
3. Analyze the code deeply based on the specific "Focus" provided.
4. **Provide code snippets** for key concepts in your explanation. Quote relevant source code directly to support your analysis rather than describing it abstractly.
5. Generate the output exactly matching the `RESEARCH_TEMPLATE.md` structure.
6. Stop and wait for the user to verify the output and provide the next command.

**Version:** v2026.03.13.00
**Submodule path:** `velox/velox/`
**Primary source root:** `velox/velox/velox/`

---

## Phase 1: The Foundation & Data Layout (Vector Model)
**Objective:** Understand how Velox represents columnar data in memory. Velox's "Vector" abstraction is its equivalent to Arrow's `Array` — a central concept that all operators, expressions, and I/O components interact with.

* **Task 1.1.A: The `Buffer` Primitive**
  * **Target Files:** `velox/buffer/Buffer.h`, `velox/buffer/Buffer.cpp`
  * **Focus:** How does Velox manage raw memory regions? Analyze the `Buffer` class's reference-counting model, CoW (copy-on-write) semantics, and how it integrates with `MemoryPool`. Compare to Arrow's `Buffer` and Trino's `Slice`.
* **Task 1.1.B: The `BaseVector` Hierarchy**
  * **Target Files:** `velox/vector/BaseVector.h`, `velox/vector/BaseVector.cpp`
  * **Focus:** Analyze the `BaseVector` abstract class contract. What are the mandatory interface methods? How does Velox encode type information (`TypePtr`), nulls (nulls buffer), and vector size? How does the encoding enum (`VectorEncoding::Simple`) determine the concrete subtype?
* **Task 1.1.C: `FlatVector` — The Dense Representation**
  * **Target Files:** `velox/vector/FlatVector.h`, `velox/vector/FlatVector-inl.h`, `velox/vector/FlatVector.cpp`
  * **Focus:** How does `FlatVector<T>` store fixed-width primitive values and variable-length strings (via `StringView`)? Trace the layout of the values buffer and the nulls buffer. Compare to `LongArrayBlock` in Trino and `PrimitiveArray` in Arrow.
* **Task 1.1.D: Encoded Vectors — Dictionary & Constant**
  * **Target Files:** `velox/vector/DictionaryVector.h`, `velox/vector/ConstantVector.h`
  * **Focus:** How does `DictionaryVector` wrap an indices buffer and a dictionary `BaseVector`? How does `ConstantVector` represent a single repeated value without allocating a full values buffer? When are these encodings preserved vs. flattened?
* **Task 1.2.A: The `DecodedVector` — Uniform Access**
  * **Target Files:** `velox/vector/DecodedVector.h`, `velox/vector/DecodedVector.cpp`
  * **Focus:** `DecodedVector` is the key abstraction for reading any encoding uniformly. Trace how `decode()` peels layers of encoding to produce a flat `data` pointer and an `indices` array. Why does this design avoid materializing copies during expression evaluation?
* **Task 1.2.B: The `Type` System**
  * **Target Files:** `velox/type/Type.h`, `velox/type/Type.cpp`
  * **Focus:** How does Velox represent SQL types? Trace the `Type` class hierarchy (scalar, complex: `ROW`, `ARRAY`, `MAP`). How are types used to parameterize `BaseVector` and how are they compared/unified?

---

## Phase 2: Execution Model (Task / Driver / Operator)
**Objective:** Understand how Velox physically executes a query plan — from a `Task` object holding the plan down to `Driver` threads processing `Operator` pipelines.

* **Task 2.1.A: `Task` — The Query Execution Unit**
  * **Target Files:** `velox/exec/Task.h`, `velox/exec/Task.cpp`, `velox/exec/TaskStructs.h`
  * **Focus:** How is a `Task` created from a `core::PlanNode` tree? Trace the lifecycle states (e.g., `kRunning`, `kFinished`, `kFailed`). How does `Task` manage splits (work units from connectors), and how does it coordinate Driver creation per pipeline/partition?
* **Task 2.1.B: `Driver` — The Execution Thread**
  * **Target Files:** `velox/exec/Driver.h`, `velox/exec/Driver.cpp`
  * **Focus:** A `Driver` is a thread that runs a linear pipeline of `Operator`s. Trace the `Driver::run()` loop: how does it call `getOutput()` on operators, handle blocking reasons (`BlockingReason`), and yield back to the `Executor`? Compare to DataFusion's `partition` streams and Trino's driver loop.
* **Task 2.2.A: `Operator` — The Pipeline Stage Interface**
  * **Target Files:** `velox/exec/Operator.h`, `velox/exec/Operator.cpp`
  * **Focus:** Analyze the `Operator` interface: `addInput()`, `getOutput()`, `needsInput()`, `isBlocked()`. How does the push/pull duality work — is Velox push-based or pull-based? Trace how a simple `FilterProject` operator implements this contract.
* **Task 2.2.B: Exchange — Cross-Node Shuffle**
  * **Target Files:** `velox/exec/Exchange.h`, `velox/exec/ExchangeClient.h`, `velox/exec/ExchangeQueue.h`
  * **Focus:** How does Velox receive shuffled data from remote tasks? Trace the `Exchange` operator's interaction with `ExchangeClient` and `ExchangeQueue`. How is back-pressure signaled when the queue is full? How does this map to Trino's `ExchangeClient`?

---

## Phase 3: Expression Evaluation
**Objective:** Understand how Velox evaluates SQL expressions efficiently over columnar vectors, including its "peeling" optimization and vectorized function dispatch.

* **Task 3.1.A: `Expr` and `ExprSet` — The Expression Tree**
  * **Target Files:** `velox/expression/Expr.h`, `velox/expression/Expr.cpp`, `velox/expression/ExprCompiler.h`
  * **Focus:** How is a logical `core::ITypedExpr` tree compiled into a physical `ExprSet`? Trace the structure of `Expr` nodes (inputs, `VectorFunction`, result type). How does `ExprSet::eval()` drive evaluation over a `RowVector`?
* **Task 3.1.B: `EvalCtx` — Evaluation Context and Encoding Peeling**
  * **Target Files:** `velox/expression/EvalCtx.h`, `velox/expression/EvalCtx.cpp`
  * **Focus:** `EvalCtx` manages the active selection vector (`SelectivityVector`) and result reuse. Trace the "peeling" mechanism: how does Velox detect a `DictionaryVector` input and evaluate the expression only on distinct dictionary values, then re-wrap the result? Why is this the key to Velox's expression performance?
* **Task 3.2.A: `VectorFunction` — The Vectorized UDF Contract**
  * **Target Files:** `velox/expression/VectorFunction.h`, `velox/expression/VectorFunction.cpp`
  * **Focus:** Analyze the `VectorFunction::apply()` contract. How does a vectorized function receive `DecodedVector` inputs and write into a pre-allocated result `BaseVector`? Compare simple functions (scalar output per row) vs. functions that can handle encoded inputs natively.

---

## Phase 4: Memory Management
**Objective:** Understand Velox's memory architecture — from the low-level allocator to the hierarchical pool system and the spill-based arbitration model.

* **Task 4.1.A: `MemoryAllocator` — The Allocation Backend**
  * **Target Files:** `velox/common/memory/MemoryAllocator.h`, `velox/common/memory/MallocAllocator.cpp`, `velox/common/memory/MmapAllocator.cpp`
  * **Focus:** Velox has two allocator backends. Trace `MallocAllocator` (wraps `malloc`) vs `MmapAllocator` (manages huge pages via `mmap` with a size-class slab system). How does `MmapAllocator` track and reuse allocations via `Allocation` objects? Compare to DataFusion's `MemoryPool` trait.
* **Task 4.1.B: `MemoryPool` — The Hierarchical Pool**
  * **Target Files:** `velox/common/memory/MemoryPool.h`, `velox/common/memory/MemoryPool.cpp`
  * **Focus:** How does the `MemoryPool` hierarchy (root → aggregate → leaf) propagate memory usage? Trace how a `LeafMemoryPool::allocate()` call updates parent pools and triggers capacity checks. How does the pool enforce memory limits?
* **Task 4.2.A: `MemoryArbitrator` and `SharedArbitrator` — Spill-Based Reclamation**
  * **Target Files:** `velox/common/memory/MemoryArbitrator.h`, `velox/common/memory/SharedArbitrator.h`, `velox/common/memory/SharedArbitrator.cpp`
  * **Focus:** When a query exceeds its memory limit, the `SharedArbitrator` is invoked. Trace the arbitration protocol: how does it identify the best `ArbitrationParticipant` to spill, how does it request memory reclamation, and how does it grant the freed capacity to the requestor? How does this compare to DataFusion's `MemoryPool::try_grow()` / spill model?
* **Task 4.2.B: `HashStringAllocator` — In-Place State Allocation for Aggregation**
  * **Target Files:** `velox/common/memory/HashStringAllocator.h`, `velox/common/memory/HashStringAllocator.cpp`
  * **Focus:** Aggregation operators maintain per-group state (e.g., running sum, HLL sketches). `HashStringAllocator` provides a bump-pointer allocator over `MemoryPool`-backed pages. Trace how it manages free lists and how aggregation accumulators use `ByteStream` to write variable-length state.

---

## Phase 5: I/O & Connectors
**Objective:** Understand how Velox reads external data — the connector SPI, split processing, and how column pruning/predicate pushdown are expressed.

* **Task 5.1.A: The Connector SPI**
  * **Target Files:** `velox/connectors/Connector.h`, `velox/connectors/ConnectorQueryCtx.h`
  * **Focus:** Analyze the `Connector` and `DataSource` abstract interfaces. How does a connector register itself? How does `DataSource::addSplit()` and `DataSource::next()` integrate with the `TableScan` operator's pull model? What information is passed via `ConnectorQueryCtx`?
* **Task 5.1.B: Predicate Pushdown & Column Pruning**
  * **Target Files:** `velox/connectors/hive/HiveConnector.h` (or relevant connector), `velox/expression/ExprToSubfieldFilter.h`
  * **Focus:** How does Velox express column-level predicates that get pushed into the connector? Trace how a `SubfieldFilter` is constructed from an `Expr` tree and handed to the `DataSource`. How does the connector use this to skip row groups or columns at the I/O level?
