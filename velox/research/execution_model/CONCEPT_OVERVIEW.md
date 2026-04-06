# Phase 2: Execution Model — Task Architecture & The Driver Loop (Velox)

Velox's execution model is a cooperative, pull-based pipeline engine built on top of an externally-injected thread pool (`folly::Executor`). Unlike Trino's morsel-driven model with dedicated scheduler threads, or DataFusion's async Rust futures pinned to a `tokio` runtime, Velox uses a self-scheduling design: each `Driver` runs until it blocks, yields, or finishes, then re-enqueues itself. The central coordination point is the `Task`, which owns the plan-to-pipeline decomposition, driver lifecycle, and all inter-pipeline shared state (join bridges, local exchanges). This phase traces the full path from `PlanNode` tree to executing operators on thread pool threads.

## 1. Task Architecture: From Plan Tree to Pipelines

### `Task` — The Top-Level Execution Unit (Trino's `SqlTask`, DataFusion's `ExecutionPlan` + `TaskContext`)

A `Task` owns a `PlanFragment` and is responsible for decomposing it into linear pipelines, deciding concurrency, creating `Driver` instances, and managing their lifecycle. It supports two execution modes: `kParallel` (multi-threaded via executor) and `kSerial` (single-threaded, caller-driven via `Task::next()`).

Key facts from the source:

- **`LocalPlanner::plan()` recursive decomposition.** The plan tree is walked top-down, but pipelines are built bottom-up. Each `PlanNode` is appended to the current pipeline after its sources are processed. A new pipeline (`DriverFactory`) is created whenever `mustStartNewPipeline()` returns true.
- **Pipeline boundaries** occur at: `LocalPartitionNode` (local exchange), `LocalMergeNode`, `MixedUnionNode`, and any non-first source of a multi-source node (e.g., the build side of a hash join always runs in its own pipeline).
- **Concurrency = minimum across all operators.** `detail::maxDrivers()` walks every node in a `DriverFactory` and takes the minimum. A single `requiresSingleThread()` node forces the entire pipeline to one driver. The caller-supplied `maxDrivers` further caps this.
- **DriverFactory is a blueprint; Driver is an instance.** `DriverFactory` holds the plan node sequence and concurrency metadata. `createDriver()` is called N times (once per `numDrivers`), each producing a fresh operator chain. All drivers in a pipeline share the same `PipelinePushdownFilters`.
- **Grouped (bucketed) execution.** Drivers are created on-demand per split group rather than all at once. `concurrentSplitGroups_` limits how many groups run simultaneously. When a group's drivers all finish, the slot is freed and the next queued group starts — amortizing memory across groups.
- **Two execution modes.** Parallel mode uses `Task::start()` → executor; serial mode uses `Task::init()` eagerly with `maxDrivers=1`, and `Task::next()` round-robins across drivers on the caller thread.

### `SplitGroupState` — Inter-Pipeline Shared State

Before creating drivers for a split group, the Task sets up shared coordination structures:
- **Join bridges** (`HashJoinBridge`, `NestedLoopJoinBridge`, `SpatialJoinBridge`) coordinate build and probe pipelines. The last build driver to finish calls `allPeersFinished()`, unblocking probe drivers.
- **Local exchange queues** (`LocalExchangeQueue`) coordinate upstream producer and downstream consumer pipelines for repartitioning.
- **Local merge sources** provide ordered merge input from parallel upstream pipelines.

## 2. Scheduling Engine: Thread Pool and CPU Fairness

### The Executor is External and Opaque

Velox does **not** implement its own thread pool. It delegates entirely to `folly::Executor`, injected via `QueryCtx`. The only API used is `executor->add(func)`. In production (Prestissimo), this is typically a `folly::CPUThreadPoolExecutor` with FIFO semantics and LIFO thread wake-up. Velox has no built-in priority queue, no inter-task fairness at the executor level, and no ability to preempt drivers.

### `Driver::enqueue()` — The Single Scheduling Primitive

```
Driver::enqueue(driver)
  → enqueueInternal()     [set isEnqueued=true, start queue timer]
  → executor->add(lambda)  [capture shared_ptr<Driver>]
```

The driver's `shared_ptr` is captured by the lambda, preventing destruction while queued. This is the only point where drivers are submitted to the thread pool.

### Two-Level Cooperative Time-Slicing

CPU fairness is achieved through two independent yield mechanisms:

**Level 1: Per-driver CPU time slice** (`driverCpuTimeSliceLimitMs`). Each driver checks wall-clock time at every operator iteration via `shouldYield()`. When exceeded, the driver returns `StopReason::kYield` and re-enqueues to the back of the executor queue. Disabled by default (0); must be explicitly configured. Only active in parallel mode.

**Level 2: Task-level yield** (`yieldIfDue` / `toYield_`). An external caller (e.g., resource manager) can set `toYield_ = numThreads_`, causing all on-thread drivers of a task to yield at their next `shouldStop()` check. Each yielding driver decrements the counter under the lock.

### Atomic Fast-Path Minimizes Lock Contention

`Task::shouldStop()` is called at every operator iteration in the hot loop. It uses three atomic reads without taking the mutex:

```
if (pauseRequested_) return kPause;
if (terminateRequested_) return kTerminate;
if (toYield_) { lock; return shouldStopLocked(); }
return kNone;
```

Only when a yield is actually pending does the lock get taken. In the common case, this is just three atomic reads.

## 3. Driver Initialization: Two-Phase Lifecycle

### Construction Under Lock, Initialization on Executor Thread

Operators are constructed during `DriverFactory::createDriver()`, which runs under the Task lock. The critical constraint: **operator constructors must NOT allocate memory from the pool**. Memory allocation could trigger arbitration, which also needs the Task lock — causing deadlock.

Instead, memory-allocating initialization is deferred to `Operator::initialize()`, called on first execution by `Driver::initializeOperators()` on the executor thread (outside the Task lock). This is enforced by a zero-usage check: `VELOX_CHECK_EQ(pool()->usedBytes(), 0)`.

### 4-Level Memory Pool Hierarchy

```
QueryCtx pool (root, per-query limit)
  └── task.{taskId}                         (aggregate, TaskReclaimer)
       └── node.{planNodeId}                (aggregate, ParallelMemoryReclaimer)
            └── op.{nodeId}.{pipe}.{drv}.{type}  (leaf, Operator::MemoryReclaimer)
```

- **Node pools are shared** across all drivers for the same plan node, enabling coordinated memory reclaim (critical for hash joins).
- **Join nodes get split-group-scoped pools**, keyed on `{planNodeId}[{splitGroupId}]`, giving each split group its own node pool.
- **Spill config is per-driver**, with directory paths incorporating `{pipelineId}_{driverId}_{operatorId}` to avoid file collisions.

### Filter+Project Fusion

During operator assembly, consecutive `FilterNode` → `ProjectNode` pairs are fused into a single `FilterProject` operator, eliminating intermediate vector materialization. This is done at the factory level by checking the next plan node in sequence.

### Extension Points

- **`DriverAdapter`** allows registered plugins to inspect and modify the operator chain after assembly (first match wins). `isAdaptable_` is locked after adapters run.
- **`PlanNodeTranslator`** provides a pluggable fallback for plan nodes not handled by the built-in `dynamic_pointer_cast` chain.
- **Thread-local `DriverThreadContext`** allows any code on a driver thread to discover the active `DriverCtx` without explicit parameter passing.

## 4. The Cooperative Yield Loop: `Driver::runInternal()`

### Consumer-First Iteration

The inner loop walks operators from the **consumer (sink, highest index) toward the producer (source, index 0)**. When output is successfully passed downstream (`addInput`), the loop restarts from the consumer end via `i += 2; continue`. This prioritizes draining data already in the pipeline before pulling more from the source, preventing unbounded buffering.

### Five Yield/Block Triggers

At every operator iteration, the driver checks:

1. **Task stop signals** — `shouldStop()` checks `pauseRequested_`, `terminateRequested_`, `toYield_` (atomic fast-path).
2. **CPU time-slice** — `shouldYield()` compares wall-clock time against `cpuSliceMs_`.
3. **Memory arbitration** — `checkUnderArbitration()` blocks if the query's pool is under arbitration.
4. **Operator blocking** — `isBlocked()` on both the current (producer) and next (consumer) operator. Both must be checked to prevent data loss or overflow.
5. **Pipeline completion** — sink `isFinished()` triggers `close()` → `Task::removeDriver()`.

### Blocking and Resumption via Futures

When a driver blocks, it creates a `BlockingState` capturing `(Driver, ContinueFuture, Operator*, BlockingReason)` and goes off-thread. The resumption path:

```
BlockingState::setResume(state)
  → future.via(QueuedImmediateExecutor).thenValue(callback)
  → callback: lock Task mutex → clear hasBlockingFuture → if !paused: Driver::enqueue()
```

- **`QueuedImmediateExecutor`** prevents reentrancy: if the future is already fulfilled, the callback is queued rather than executing inline, avoiding two threads trying to enter the same driver simultaneously.
- **Pause-awareness**: if a task pause was requested while the driver was blocked, the driver is NOT re-enqueued. It stays off-thread and will be enqueued when `Task::resume()` iterates over all off-thread drivers.
- **Blocking contract**: every operator that returns a non-`kNotBlocked` reason MUST provide a valid `ContinueFuture`. Without it, the driver hangs forever.

### `CancelGuard` RAII

The `CancelGuard` guarantees `Task::leave()` is called on all exit paths from `runInternal()`. Normal exits call `guard.notThrown()`. If an exception escapes (default `isThrow_ = true`), the destructor marks the driver terminated and fires the `onTerminate_` callback (which closes the driver). This ensures `numThreads_` is always correctly decremented.

## 5. Comparison with Trino and DataFusion

### Execution Model

| Dimension | Velox | Trino | DataFusion |
|-----------|-------|-------|------------|
| Execution unit | `Driver` (linear operator pipeline) | `Driver` (linear operator pipeline) | `ExecutionPlan` partition (async stream) |
| Thread model | External `folly::Executor`, cooperative yield | Dedicated scheduler threads, morsel-driven | `tokio` async runtime, `Stream::poll_next()` |
| Preemption | None — cooperative only | Morsel boundaries = natural yield points | Rust future yield points (`Poll::Pending`) |
| Pipeline decomposition | Recursive `LocalPlanner::plan()` at runtime | Planner-phase `LocalExecutionPlanner` | Optimizer-phase partitioning |
| Concurrency control | `min(maxDrivers, pipeline constraints)` | `maxDriversPerTask` | `target_partitions` config |
| Blocking model | `ContinueFuture`/`ContinuePromise` (folly) | `ListenableFuture` (Guava) | `Poll::Pending` + `Waker` (Rust async) |

### Task and Driver Lifecycle

| Aspect | Velox `Task` | Trino `SqlTaskExecution` | DataFusion `TaskContext` |
|--------|-------------|--------------------------|--------------------------|
| Plan → pipeline | `LocalPlanner::plan()` (recursive tree walk) | `LocalExecutionPlanner` (visitor pattern) | Optimizer produces partitioned `ExecutionPlan` |
| Driver creation | `DriverFactory::createDriver()` under Task lock | `DriverFactory.createDrivers()` | N/A — each partition is an async stream |
| Memory pools | 4-level hierarchy (query → task → node → operator) | 3-level (`MemoryContext` tree) | Flat `MemoryPool` (no hierarchy) |
| Operator init | Two-phase: construct (no alloc) → `initialize()` (deferred) | Single-phase in constructor | Single-phase in constructor |
| Inter-pipeline | Join bridges, local exchange queues (per split group) | Join bridges, local exchange (per pipeline) | No explicit bridges — channels + async streams |
| Grouped execution | On-demand driver creation per split group | Bucketed execution via `BucketedSplitSource` | Not natively supported |

### Scheduling and CPU Fairness

| Aspect | Velox | Trino | DataFusion |
|--------|-------|-------|------------|
| Thread pool | External `folly::CPUThreadPoolExecutor` | Internal `MultilevelSplitQueue` with priority | External `tokio` runtime |
| Scheduling primitive | `executor->add(lambda)` | `SplitRunner` submitted to `TaskExecutor` | `tokio::spawn()` |
| CPU fairness | Cooperative time-slice (`cpuSliceMs_`) + task-level yield | 5-level priority queue with accumulated CPU penalty | Tokio work-stealing; no query-level fairness |
| Yield mechanism | `shouldYield()` wall-clock check per operator iteration | Morsel exhaustion = natural yield | `Poll::Pending` at async boundaries |
| Dynamic scaling | `ScaledScanController` for scan drivers | Not applicable (fixed concurrency) | Not applicable |
| Pause/resume | `pauseRequested_` atomic + `threadFinishPromises_` | Not supported (cancel only) | `tokio::CancellationToken` |

### Blocking and Resumption

| Aspect | Velox | Trino | DataFusion |
|--------|-------|-------|------------|
| Blocking signal | `isBlocked()` returns `BlockingReason` + `ContinueFuture` | `isBlocked()` returns `ListenableFuture` | `Poll::Pending` + `Waker` registration |
| Resume trigger | `ContinuePromise::setValue()` → future callback → `Driver::enqueue()` | `ListenableFuture` completion → `DriverSplitRunner.processFor()` | `Waker::wake()` → tokio re-polls the stream |
| Executor for resume | `folly::QueuedImmediateExecutor` (inline, queued) | Guava `directExecutor()` | tokio waker (task re-scheduled) |
| Pause-awareness | Checked in resume callback; driver stays off-thread if paused | Not applicable | N/A |
| Process-wide tracking | `BlockingState::numBlockedDrivers_` atomic counter | Not exposed | Not tracked |
