# Phase 5: Memory Tracking & Arbitration Overview (DataFusion)

DataFusion's memory management is architecturally simpler than Trino's but achieves comparable safety guarantees through Rust's ownership model. Where Trino builds a hierarchical tracking tree with explicit locking and GC-based size estimation, DataFusion uses a flat pool with RAII-based reservations that make memory leaks structurally impossible.

## 1. The Single Pool

Like Trino, DataFusion uses a **single `MemoryPool`** per process. The pool is held in `RuntimeEnv` (via `Arc<dyn MemoryPool>`) and shared across all concurrently executing queries. Unlike Trino's `ConcurrentHashMap`-based tracking, DataFusion's pool implementations use either `AtomicUsize` (lock-free) or a single `Mutex` (for fairness logic).

Four implementations ship with DataFusion:

| Pool | Tracking | Limit Enforcement | Use Case |
|---|---|---|---|
| **`UnboundedMemoryPool`** | `AtomicUsize` counter | None | Development, testing |
| **`GreedyMemoryPool`** | `AtomicUsize` counter | Hard cap, first-come-first-served | Simple production use |
| **`FairSpillPool`** | `Mutex<FairSpillPoolState>` | Fair share: `(pool - unspillable) / num_spill` per consumer | Production with spilling |
| **`TrackConsumersPool<I>`** | Wraps any pool `I` | Delegates to inner pool | Debugging — adds per-consumer usage tracking for error messages |

**Critical difference from Trino:** When a DataFusion pool is exhausted, `try_grow()` returns `Err(ResourcesExhausted)` immediately — it does **not** return a pending future. The operator must either spill to disk and retry, or propagate the error to abort the query. Trino's approach (returning a `ListenableFuture` that blocks the driver until memory frees up) is more graceful but requires the custom cooperative scheduler to work.

## 2. The Flat Tracking Model (No Hierarchy)

Trino builds a deep tracking tree mirroring the execution hierarchy:

```
MemoryPool → QueryContext → TaskContext → PipelineContext → DriverContext → OperatorContext
```

DataFusion's model is flat:

```
MemoryPool ← MemoryReservation (per operator)
```

There is no per-query, per-task, or per-pipeline aggregation. Each operator creates a `MemoryConsumer`, registers it with the pool to obtain a `MemoryReservation`, and interacts with the pool directly. The pool sees individual consumers, not a hierarchy.

**Consequence:** DataFusion has no per-query memory limit equivalent to Trino's `query.max-memory-per-node`. All consumers compete for the same global pool. The `FairSpillPool` provides consumer-level fairness but not query-level isolation.

## 3. The RAII Reservation Pattern

This is DataFusion's key innovation over Trino's memory model. The `MemoryReservation` struct leverages Rust's ownership system:

* **Creation:** `MemoryConsumer::new("SortExec").register(&pool)` returns a `MemoryReservation` with 0 bytes.
* **Growth:** `reservation.try_grow(bytes)` asks the pool for more memory. Pool check first, then local `AtomicUsize` counter update. If pool rejects, size never increments.
* **Shrinkage:** `reservation.shrink(bytes)` uses `fetch_update` with `checked_sub` to prevent underflow, then calls `pool.shrink`. `reservation.free()` uses `swap(0)` for idempotent atomic zeroing — calling `free()` twice is safe (second call is a no-op). This eliminates double-accounting bugs where manual cleanup and `Drop` could conflict.
* **Drop:** `Drop` calls `self.free()`. Combined with `swap(0)` idempotency, this is a watertight compiler-enforced guarantee — no double-free, no panics.

**Why this matters for a Rust engine:** In Trino (JVM), an operator can allocate heap memory without telling the `MemoryPool`, leading to under-counting. The GC might also delay collection, causing over-counting. DataFusion's model makes these bugs structurally impossible — the `MemoryReservation` is the only way to interact with the pool, and Rust's compiler enforces that it is always dropped.

### Split Reservations

A single operator can hold **multiple reservations** sharing the same `MemoryConsumer` (via `Arc<SharedRegistration>`). For example, `SortExec` creates:
1. A **spillable** main reservation for accumulating input batches.
2. A **non-spillable** merge reservation for the final merge phase.

`reservation.split(bytes)` transfers bytes from one reservation to a new child. The `SharedRegistration` ensures `pool.unregister()` is only called when the *last* reservation is dropped.

## 4. The `can_spill` Contract

Each `MemoryConsumer` declares `can_spill: bool`. This is not just metadata — it's a contract:

* **`can_spill = true`:** The operator promises to handle `ResourcesExhausted` by spilling to disk and retrying with less memory. The `FairSpillPool` uses this to calculate fair shares: each spillable consumer gets at most `(pool_size - unspillable_bytes) / num_spillable_consumers`.
* **`can_spill = false`:** The operator cannot free memory on demand. The `FairSpillPool` gives these consumers priority — their bytes are subtracted from the pool before the fair share is calculated.

**Note:** `GreedyMemoryPool` ignores `can_spill` entirely. The flag only affects `FairSpillPool` and `TrackConsumersPool` (for error reporting). This means the pool type determines whether the contract has any practical effect.

## 5. Pull-Based Spilling vs. Push-Based Revocation

This is where the architectures radically diverge.

* **Trino (Push):** A background thread (`MemoryRevokingScheduler`) notices the pool is full, walks the tracking tree, and *pushes* a revoke signal to the operators, forcing them to spill via `startMemoryRevoke()`/`finishMemoryRevoke()`.
* **DataFusion (Pull):** There is no background revoking thread. If `try_grow()` fails, the operator catches the error inline. The operator then *proactively* chooses to spill its own data to disk using the `SpillManager`/`DiskManager`, frees its `MemoryReservation`, and tries to grow again. The operator is in complete control of its own lifecycle. *(See Phase 3, Task 3.5 for full spilling trace.)*

This means DataFusion's spilling is **synchronous and local** — the decision to spill is made within the same `poll_next()` call that triggered the memory pressure. No cross-task coordination, no background scheduler, no revocation protocol.

### Operator Spill Patterns

| Operator | Registration | Spill Behavior |
|---|---|---|
| **ExternalSorter** | Dual reservation: spillable (main) + non-spillable (merge headroom) | `try_grow(2 * batch_size)` fails → sort-and-spill entire accumulated state, shrink, retry. Merge headroom is non-spillable to prevent starvation during final merge. |
| **GroupedHashAggregateStream** | Single reservation, 3 OOM modes | `try_resize()` after each batch. Spill mode: emit all groups as intermediate state, sort, write to disk. After entering merge phase, switches to `ReportError` to prevent recursive spilling. Uses 2x memory reservation as headroom. |
| **HashJoinExec** | Non-spillable (`can_spill=false`) | Memory exhaustion is a hard error — no spill path. |
| **RepartitionExec** | Per-batch spill decisions | Finest granularity: each batch independently goes to memory or disk based on instantaneous `try_grow()` result. Can mix in-memory and spilled batches within a single partition channel. |

## 6. Task Context and Global Limits

* **`RuntimeEnv`:** The `MemoryPool` is instantiated inside `RuntimeEnv` (via `RuntimeEnvBuilder`). The default pool is `UnboundedMemoryPool` — **no memory limit unless explicitly configured**. Production deployments should configure `FairSpillPool` or `GreedyMemoryPool`.
* **`TaskContext`:** Holds the `RuntimeEnv` (via `Arc`) and passes it to all operators. Each operator accesses the pool through `context.runtime_env().memory_pool`.
* **No per-query memory limit:** Unlike Trino's `query.max-memory-per-node`, DataFusion has no built-in per-query limit. All consumers across all concurrent queries compete for the same global pool. The `FairSpillPool` provides consumer-level fairness but not query-level isolation.
* **Key config options:** `sort_spill_reservation_bytes` (10MB), `max_spill_file_size_bytes` (128MB), `max_temp_directory_size` (100GB) — these are execution-level configs, not pool-level limits.

## 7. No Cluster Arbitration

DataFusion is a single-process, embeddable query engine. It has no equivalent of Trino's:
* **`ClusterMemoryManager`** — no coordinator aggregating worker memory
* **`MemoryRevokingScheduler`** — no periodic revocation sweeps
* **`LowMemoryKiller`** — no OOM killer selecting victim queries

When the pool is exhausted, the operator that triggered the limit gets an immediate error. If it can spill, it does. If it can't, the query fails. There is no mechanism to preemptively revoke memory from other queries to make room.

This is a deliberate design choice: DataFusion targets embedded analytics and single-node execution. Distributed memory coordination is left to higher-level frameworks (like Ballista or custom deployments) that build on DataFusion.

## 8. Summary: DataFusion vs. Trino Memory Model

| Dimension | Trino | DataFusion |
|---|---|---|
| **Pool structure** | Single `MemoryPool` per node, `ConcurrentHashMap`-based | Single `MemoryPool` per process, `AtomicUsize` or `Mutex`-based |
| **Tracking hierarchy** | Deep tree: Pool → Query → Task → Pipeline → Driver → Operator | Flat: Pool ← Reservation (per operator) |
| **Allocation protocol** | `setBytes()` propagates delta bottom-up through context tree | `try_grow()` talks directly to the pool |
| **Blocking behavior** | Returns `ListenableFuture` — driver yields and resumes later | Returns `Err` immediately — operator must spill or fail |
| **Deallocation** | Manual `setBytes(0)` + GC handles actual heap cleanup | RAII `Drop` — compiler-guaranteed, zero-delay |
| **Per-query limits** | `query.max-memory-per-node` enforced at `QueryContext` | None — all consumers share one flat pool |
| **Revocation** | `MemoryRevokingScheduler` + operator `startMemoryRevoke()`/`finishMemoryRevoke()` | None — operators spill on `try_grow()` failure |
| **Cluster OOM** | `ClusterMemoryManager` + `LowMemoryKiller` kills victim queries | N/A — single process |
| **Lock contention mitigation** | `CoarseGrainLocalMemoryContext` (64KB batching), CAS-based futures | `AtomicUsize` (lock-free) or single `Mutex` with minimal critical section |
