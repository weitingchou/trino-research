# DataFusion Source Code Tracing Manifest (Hyper-Granular)

**Instructions for Claude:**
This is an atomic task list for analyzing the Apache DataFusion source code (and its underlying Apache Arrow components). **DO NOT attempt to execute multiple tasks at once.** The user will specify a Task ID (e.g., "Claude, execute Task 1.1.A"). 
1. Read the specified source files using your CLI tools to understand the implementation.
2. **"Target Files" are a suggested starting point only** — they are not exhaustive. Automatically expand your research scope to any related dependencies, callers, implementations, or utilities that are relevant to the Focus. Follow the code wherever it leads.
3. Analyze the code deeply based on the specific "Focus" provided.
4. **Provide code snippets** for key concepts in your explanation. Quote relevant source code directly to support your analysis rather than describing it abstractly.
5. Generate the output exactly matching the `RESEARCH_TEMPLATE.md` structure.
6. Stop and wait for the user to verify the output and provide the next command.

---

## Phase 1: The Foundation & SPI (Data Layout)
**Objective:** Understand raw columnar data representation before it enters the physical execution engine. DataFusion relies natively on the Apache Arrow memory model.

* **Task 1.1.A: The `Buffer` and `ArrayData` Foundation**
  * **Target Crates/Files:** `arrow-buffer/src/buffer/immutable.rs`, `arrow-data/src/data/mod.rs`
  * **Focus:** How does Arrow manage contiguous memory regions? Trace how `ArrayData` stores buffers, lengths, null counts, and offsets.
* **Task 1.1.B: The `Array` Trait and Primitives**
  * **Target Crates/Files:** `arrow-array/src/array/mod.rs`, `arrow-array/src/array/primitive_array.rs`
  * **Focus:** Analyze the `Array` trait contract. How does `PrimitiveArray` map standard types (like `i32`, `f64`) onto the underlying `ArrayData` buffers?
* **Task 1.1.C: Variable-Width Storage & Strings**
  * **Target Crates/Files:** `arrow-array/src/array/string_array.rs`, `arrow-array/src/array/byte_array.rs`
  * **Focus:** Trace exactly how variable-length data (like `StringArray`) calculates its offsets buffer and manages its underlying byte buffer.
* **Task 1.1.D: Nullability and Validity Bitmaps**
  * **Target Crates/Files:** `arrow-buffer/src/buffer/null_buffer.rs`
  * **Focus:** How are nulls conceptually represented? Trace how the validity bitmap is read and written using bitwise operations.
* **Task 1.2.A: The `RecordBatch` Construct**
  * **Target Crates/Files:** `arrow-array/src/record_batch.rs`
  * **Focus:** How are multiple `Array` instances bound together into a `RecordBatch` (representing a batch of rows)? Trace how row counts and schema alignments are validated.
* **Task 1.2.B: Schema and Field Metadata**
  * **Target Crates/Files:** `arrow-schema/src/schema.rs`, `arrow-schema/src/field.rs`
  * **Focus:** Analyze how `Schema` and `Field` hold datatype information and metadata.

---

## Phase 2: Worker Scheduling (Execution Model)
**Objective:** Map the relationship between Execution Plans, Partitions, and Async Streams to understand DataFusion's pull-based execution model.

### 2.1 Plan Partitioning & Task Context
* **Task 2.1.A: The `ExecutionPlan` & Partitioning**
  * **Target Crates/Files:** `datafusion-physical-plan/src/lib.rs` (focus on `ExecutionPlan` and `Partitioning`)
  * **Focus:** How does a physical plan define its level of concurrency? Trace the `output_partitioning()` method to understand how an operator declares how many parallel streams (drivers) it can produce.
* **Task 2.1.B: `TaskContext` Initialization & Resource Wiring**
  * **Target Crates/Files:** `datafusion-execution/src/task.rs` (focus on `TaskContext` and `SessionConfig`)
  * **Focus:** Trace how a `TaskContext` is wired up before execution begins. How does it link the `MemoryPool`, `DiskManager`, and session properties to the execution of a specific partition?

### 2.2 Async Task Spawning
* **Task 2.2.A: Tokio Task Spawning**
  * **Target Crates/Files:** `datafusion-physical-plan/src/lib.rs` or operator-specific files like `datafusion-physical-plan/src/repartition/mod.rs`
  * **Focus:** How are multiple partitions mapped to actual CPU threads? Find the exact mechanisms where DataFusion uses `tokio::spawn` or `tokio::task::spawn_blocking` to hand off work to the async runtime.
* **Task 2.2.B: Task Cancellation & Failure Propagation**
  * **Target Crates/Files:** `datafusion-execution/src/task.rs` (focus on `JoinHandle` or cancellation logic)
  * **Focus:** If one partition fails or the query is aborted, how is the failure propagated? Trace how dropping a Rust `Future` or aborting a Tokio task handles the teardown.

### 2.3 The Stream Lifecycle
* **Task 2.3.A: Stream Initialization**
  * **Target Crates/Files:** `datafusion-physical-plan/src/stream.rs` (focus on `RecordBatchStream`)
  * **Focus:** How is an operator instantiated into an active stream? Trace the `execute(partition: usize, context: Arc<TaskContext>)` method across a core operator (like Filter or Projection).
* **Task 2.3.B: The Pull-Based Execution Loop (`poll_next`)**
  * **Target Crates/Files:** `datafusion-physical-plan/src/stream.rs` and `std::task::Poll` implementations
  * **Focus:** This is the core engine loop. Trace how `poll_next()` is called. Document exactly where the stream hits an `.await` point (yielding control back to the Tokio executor) when waiting for upstream data or disk I/O.
* **Task 2.3.C: Stream Termination & Cleanup**
  * **Target Crates/Files:** Implementations of `RecordBatchStream` in specific operators.
  * **Focus:** When is a stream considered finished? Trace the path when `poll_next()` finally returns `Poll::Ready(None)`, and analyze how `MemoryReservation` drops automatically release memory back to the `TaskContext`.

---

## Phase 3: The Operator Pipeline (Physical Plan Execution)
**Objective:** Trace the physical data processing nodes to understand streaming query execution.

* **Task 3.1.A: Physical Expressions (`PhysicalExpr`)**
  * **Target Crates/Files:** `datafusion-physical-expr/src/physical_expr.rs`
  * **Focus:** Analyze how column references and scalar math are evaluated against a `RecordBatch`. Trace the `evaluate()` method.
* **Task 3.2.A: Simple Projection & Filtering**
  * **Target Crates/Files:** `datafusion-physical-plan/src/filter.rs`, `datafusion-physical-plan/src/projection.rs`
  * **Focus:** Trace the simplest data flow: taking an input stream, applying a boolean mask (Filter), and returning a transformed `RecordBatch` (Projection).
* **Task 3.3.A: Hash Join - Building the Table**
  * **Target Crates/Files:** `datafusion-physical-plan/src/joins/hash_join.rs` (focus on build-side logic)
  * **Focus:** How does DataFusion ingest the "build" side of a join? Analyze the `JoinHashMap` structure and how it stores indices.
* **Task 3.3.B: Hash Join - Probing**
  * **Target Crates/Files:** `datafusion-physical-plan/src/joins/hash_join.rs` (focus on probe-side logic)
  * **Focus:** How does the "probe" side stream through and match against the built hash table? Trace the batch-level joining logic.
* **Task 3.4.A: Aggregation & Accumulators**
  * **Target Crates/Files:** `datafusion-physical-plan/src/aggregates/mod.rs`, `datafusion-physical-expr/src/aggregate/accumulator.rs`
  * **Focus:** How does an `AggregateExec` maintain group-by state? Analyze the `Accumulator` trait (`update_batch`, `evaluate`).
* **Task 3.5.A: Sorting and Spilling Mechanism**
  * **Target Crates/Files:** `datafusion-physical-plan/src/sorts/sort.rs`, `datafusion-physical-plan/src/sorts/sort_preserving_merge.rs`
  * **Focus:** When sorting data that exceeds memory, how does `SortExec` serialize its state to disk? Trace the spilling trigger and temp file creation.

---

## Phase 4: Network & Data Exchange (Shuffle)
**Objective:** Understand how DataFusion shuffles data between partitions and threads locally, as well as the mechanisms for over-the-wire exchange.

* **Task 4.1.A: Local Repartitioning (The Exchange)**
  * **Target Crates/Files:** `datafusion-physical-plan/src/repartition/mod.rs`
  * **Focus:** How does `RepartitionExec` take `N` input streams and shuffle them into `M` output streams?
* **Task 4.1.B: MPSC Channel Shuffling**
  * **Target Crates/Files:** `datafusion-physical-plan/src/repartition/mod.rs` (focus on channel routing)
  * **Focus:** Trace the use of Tokio MPSC (Multi-Producer, Single-Consumer) channels to move `RecordBatch`es across async task boundaries.
* **Task 4.1.C: Hash vs Round-Robin Routing**
  * **Target Crates/Files:** `datafusion-physical-plan/src/common/ipc.rs` or routing logic in `repartition`
  * **Focus:** Trace the hash calculation for partitioning. How does it ensure specific rows are routed to the correct downstream partition?
* **Task 4.2.A: Over-The-Wire Encoding (Arrow Flight)**
  * **Target Crates/Files:** `arrow-flight/src/encode.rs`, `arrow-flight/src/decode.rs`
  * **Focus:** How are `RecordBatch` streams converted into gRPC FlightData messages for network transfer?

---

## Phase 5: Memory Tracking & Arbitration (Memory Management)
**Objective:** Map out the strict memory accounting system used to prevent out-of-memory errors.

* **Task 5.1.A: The `MemoryPool` Trait**
  * **Target Crates/Files:** `datafusion-execution/src/memory_pool/mod.rs`
  * **Focus:** Analyze the core `MemoryPool` trait. How are `grow()` and `shrink()` methods defined to track global JVM/process memory?
* **Task 5.1.B: Pool Implementations**
  * **Target Crates/Files:** `datafusion-execution/src/memory_pool/pool.rs`
  * **Focus:** Compare `GreedyMemoryPool` (first-come, first-served) and `FairSpillPool` (ensuring even distribution among tasks).
* **Task 5.2.A: The `MemoryReservation` Lifecycle**
  * **Target Crates/Files:** `datafusion-execution/src/memory_pool/mod.rs` (focus on `MemoryReservation` struct)
  * **Focus:** Trace the RAII pattern here. How does a `MemoryReservation` guarantee that memory is accurately reported and freed when an operator is dropped?
* **Task 5.2.B: Memory Consumers**
  * **Target Crates/Files:** `datafusion-execution/src/memory_pool/mod.rs` (focus on `MemoryConsumer`)
  * **Focus:** How does a specific operator register itself as a consumer? Trace the mechanism an operator uses to request an allocation size.
