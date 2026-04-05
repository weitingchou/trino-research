# Phase 5: Memory Tracking & Arbitration Overview

In a distributed SQL engine, memory is the most constrained resource. While Trino runs on the JVM with garbage collection, the engine cannot rely on the GC to prevent Out-Of-Memory crashes. Instead, Trino implements a strict, manual accounting system where every byte must be explicitly tracked before allocation.

## 1. The Single Pool

Trino 480 uses a **single unified `MemoryPool`** per worker node (the old dual-pool "general + reserved" design was removed). Pool size = JVM max heap - 30% headroom, created by `LocalMemoryManager` at startup. The pool tracks reservations in two `ConcurrentHashMap`s: `queryMemoryReservations` (total per query) and `taggedMemoryAllocations` (per-operator breakdown within each query, keyed by allocation tag like `"HashBuilderOperator"`).

When the pool is exhausted, `reserve()` still succeeds (bytes go negative) but returns a pending `NonCancellableMemoryFuture`. When `free()` brings the pool back above zero, all waiting futures are completed, unblocking parked drivers.

## 2. The Tracking Tree

Memory tracking mirrors the execution hierarchy from Phase 2. Each level is a `ChildAggregatedMemoryContext` node that delegates to its parent before recording locally:

```
MemoryPool (single per node)
  └─ QueryContext (per query, enforces maxUserMemory)
       └─ TaskContext (per task)
            └─ PipelineContext (per pipeline)
                 └─ DriverContext (per driver)
                      └─ OperatorContext (per operator — the leaf)
                           ├─ userMemory (LocalMemoryContext)
                           └─ revocableMemory (LocalMemoryContext)
```

When an operator calls `setBytes()`, the delta propagates bottom-up: `SimpleLocalMemoryContext` → `ChildAggregatedMemoryContext` chain → `RootAggregatedMemoryContext` → `QueryContext.updateUserMemory()` → `MemoryPool.reserve()`. Each node synchronizes only on `this` (fine-grained locking). Three mechanisms prevent lock contention:
- **`CoarseGrainLocalMemoryContext`**: batches allocations to 64KB granularity, reducing pool interactions ~1000x
- **CAS-based future management**: `OperatorContext.updateMemoryFuture()` uses `AtomicReference.compareAndSet` — no lock needed for blocking state propagation
- **Listener notification outside the pool lock**: `MemoryPoolListener.onMemoryReserved()` fires after the lock is released

## 3. User Memory vs. Revocable Memory

At the `OperatorContext` level, memory is tracked in two separate context trees (both rooted in `MemoryTrackingContext`):

* **User Memory:** "Hard" memory the operator needs to make progress. Cannot be taken away. When the pool is exhausted, the operator blocks via a `ListenableFuture` — the Driver detects this through `isWaitingForMemory()` and yields the CPU quantum.
* **Revocable Memory:** "Soft" memory for performance (e.g., in-memory hash tables). The operator agrees this memory can be reclaimed. `HashBuilderOperator` and `HashAggregationOperator` allocate their hash table memory as revocable via `localRevocableMemoryContext.setBytes()`.

When the per-query limit (`query.max-memory-per-node`) is exceeded, `QueryContext.enforceUserMemoryLimit()` throws `ExceededMemoryLimitException` immediately — the query dies. When only the pool is exhausted (other queries sharing the node), a pending future is returned and the operator suspends until memory frees up.

## 4. Revocation: The Pressure Relief Valve

When the pool fills beyond a configurable threshold (default 90%), `MemoryRevokingScheduler` forces operators to spill revocable memory to disk:

1. **Dual triggers**: a 1-second periodic timer AND an event-driven `MemoryPoolListener` callback on every `reserve()` call (coalesced via `AtomicBoolean checkPending`)
2. **Tree traversal**: `VoidTraversingQueryContextVisitor` walks Task → Pipeline → Driver → Operator, calling `requestMemoryRevoking()` on operators with revocable bytes, oldest tasks first
3. **Two-phase protocol in the Driver**: `startMemoryRevoke()` initiates async spill (returns a `ListenableFuture`), `finishMemoryRevoke()` completes cleanup after the spill future resolves
4. **Operator strategies**: `HashBuilderOperator` tries compaction first (if <20% savings, spills to disk instead); `SpillableHashAggregationBuilder` spills current groups and immediately rebuilds an empty builder for continued input

## 5. Cluster Arbitration: The OOM Killer

When a worker's pool is completely full and no revocable memory remains, the worker enters a "blocked" state (`freeBytes + reservedRevocableBytes <= 0`). The coordinator detects this:

1. **Aggregation**: `RemoteNodeMemory` polls each worker's `GET /v1/memory` endpoint asynchronously (minimum 1s interval). `ClusterMemoryPool.update()` sums all node snapshots.
2. **Detection**: `ClusterMemoryManager.process()` runs every 1s, checks `pool.getBlockedNodes() > 0`.
3. **Kill chain**: Task killers are tried first (minimize user impact), then query killers. Three implementations:
   - `TotalReservationOnBlockedNodesTaskLowMemoryKiller` — biggest task on blocked nodes (speculative tasks first)
   - `LeastWastedEffortTaskLowMemoryKiller` — highest memory/runtime ratio (with 30s floor)
   - `TotalReservationOnBlockedNodesQueryLowMemoryKiller` — biggest query across blocked nodes
4. **Signal propagation**: The coordinator never directly manipulates worker memory. It issues state-transition commands through the existing task infrastructure:
   - Whole-query kill: `QueryExecution.fail()` → stage abort → `DELETE /v1/task/{taskId}?abort=true` on each worker
   - Task-level kill: `QueryExecution.failTask()` → `POST /v1/task/{taskId}/fail` with `FailTaskRequest` on the target worker
5. **Debounce**: `lastKillTarget` prevents cascading kills while the previous victim's memory release propagates
