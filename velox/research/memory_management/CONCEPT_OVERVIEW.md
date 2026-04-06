# Phase 5: Memory Tracking & Arbitration — Memory Management (Velox)

Velox's memory subsystem is a three-component architecture: a **pool tree** for hierarchical accounting, an **allocator** for physical byte-level allocation, and an **arbitrator** for dynamic capacity governance across concurrent queries. The pool tree (Query → Task → Node → Operator) mirrors the execution hierarchy, with all physical allocations happening at leaf pools but reservation accounting propagating up to the root where capacity limits are enforced. The allocator layer provides two production implementations — `MallocAllocator` (delegates to `std::malloc`) and `MmapAllocator` (manages its own address space via `mmap`/`madvise`) — both exposing the same three allocation modes (non-contiguous, contiguous, raw bytes). The arbitrator (`SharedArbitrator`) orchestrates process-wide memory sharing through a multi-phase cascade: self-growth, local self-reclaim, free capacity harvesting, and finally global arbitration via a dedicated background controller thread that coordinates cross-query spill and abort.

As noted in [VELOX_IMPLEMENTATION_CHALLENGES.md](../VELOX_IMPLEMENTATION_CHALLENGES.md), the "Invisible Memory" problem is central to Velox's design as an embeddable C++ library. When Velox runs inside a JVM process (via JNI in Spark/Gluten), its C++ allocations are invisible to the JVM garbage collector. The entire memory management architecture — pools, allocators, arbitration — exists to make every byte of C++ memory explicitly tracked, bounded, and reclaimable, preventing the OS OOM killer from crashing the host process.

## 1. The Pool Tree: Quantized Reservation With Hierarchical Propagation

### Four-Level Hierarchy

```
MemoryManager (process singleton)
  |
  +-- Root Pool (kAggregate) -- one per query, owned by QueryCtx
       |
       +-- Task Pool (kAggregate) -- one per task execution
            |
            +-- Node Pool (kAggregate) -- one per plan node
                 |
                 +-- Operator Pool (kLeaf) -- one per operator instance
```

Only `kLeaf` pools allocate physical memory. `kAggregate` pools exist solely for tree organization and reservation aggregation. Pool names encode the full lineage (e.g., `op.2.0.0.Aggregation`), enabling tree dumps for debugging.

### Quantized Reservation: The Key to Low Contention

The critical optimization: **most allocations never touch parent locks**. Leaf pools reserve memory in quantized increments:

| Reservation Range | Granularity |
|-------------------|-------------|
| 0 - 16 MB | 1 MB |
| 16 MB - 64 MB | 4 MB |
| 64 MB+ | 8 MB |

When a leaf allocates, `reservationSizeLocked()` checks if enough slack exists between `reservationBytes_` and `usedReservationBytes_`. If yes, only the leaf's `std::mutex` is acquired — no parent traversal. The parent chain is updated only when crossing a quantum boundary, which is infrequent.

```
Allocation fast path (most common):
    leaf.mutex_ lock
    -> check reservationSizeLocked(size) == 0
    -> usedReservationBytes_ += size
    leaf.mutex_ unlock
    -> allocator_->allocateBytes(...)

Allocation slow path (infrequent):
    leaf.mutex_ -> determine increment -> leaf.mutex_ unlock
    -> parent.mutex_ -> increment -> parent.mutex_ unlock
    -> root.mutex_ -> check capacity -> root.mutex_ unlock
    -> (if capacity exceeded: arbitrator takes over)
```

### Three Reservation Levels Per Pool

- **`usedReservationBytes_`**: Actual bytes consumed (leaf only)
- **`reservationBytes_`**: Quantized reservation at every level (always >= `usedReservationBytes_` at leaf)
- **`capacity_`**: Root only, set by arbitrator. Hard upper bound for `reservationBytes_`

Invariant: `root.capacity_ >= root.reservationBytes_ >= sum(child.reservationBytes_)`

### Ownership Model

**shared_ptr up, weak_ptr down**: Children hold `shared_ptr<MemoryPool>` to parents (keeping parents alive), parents track children via `weak_ptr` (allowing child destruction). A `folly::SharedMutex` (reader-writer lock) protects the children map, with `visitChildren()` carefully collecting shared_ptrs under read lock then invoking visitors outside the lock to prevent deadlock.

### `tsan_atomic`: Zero-Cost Atomics in Production

Hot-path fields (`reservationBytes_`, `usedReservationBytes_`, `capacity_`) use `tsan_atomic<int64_t>` — plain `int64_t` in production (zero overhead), `std::atomic<int64_t>` only under ThreadSanitizer builds. Safe because all access is under `mutex_` on the thread-safe path.

## 2. Allocation Mechanisms: Three Modes, Two Implementations

### Three Allocation Modes

| Mode | Data Structure | Typical Consumer | Use Case |
|------|---------------|-----------------|----------|
| Non-contiguous | `Allocation` (vector of `PageRun`) | RowContainer, RowPartitions | Variable-size accumulation |
| Contiguous | `ContiguousAllocation` | HashTable, large sort buffers | Single-region random access |
| Raw bytes | `void*` via `allocateBytes()` | `AlignedBuffer` / `Buffer` | Columnar vectors |

### PageRun: 8-Byte Pointer+Size Encoding

On x86-64, virtual addresses use only the lower 48 bits. Velox packs the page count into the upper 16 bits:

```cpp
class PageRun {
  uint64_t data_;  // lower 48 bits = pointer, upper 16 bits = page count
  // Max run: 65535 pages (~256MB)
};
```

Zero metadata overhead per run — a C++-specific trick unavailable to JVM-based engines.

### MmapAllocator: Virtual Pre-Mapping With Physical Page Balancing

MmapAllocator pre-maps the entire capacity for all 9 size classes (powers of 2 from 4KB to 1MB) at construction. For a 64GB capacity, this maps 576GB of virtual address space, but physical memory is committed only on first touch.

Two-bitmap design per size class:
- **`pageAllocated_`**: 1 = allocated
- **`pageMapped_`**: 1 = backed by physical memory

This enables **mapped-free preference** — the allocator prefers pages that are both free and backed by physical memory (avoiding TLB misses), using SIMD-accelerated bitmap scanning. Physical memory is fungible across allocation types via `madvise(MADV_DONTNEED)` — contiguous hash table allocations can steal physical backing from free size-class pages.

### Three-Tier Byte Allocation

MmapAllocator routes by size:
1. **< 3KB**: `malloc()` — avoids bitmap overhead for tiny allocations
2. **3KB - 1MB**: Size class allocation — bitmap-managed, mapped-free preference
3. **> 1MB**: Contiguous `mmap` — dedicated address range with huge page support

### Contiguous Over-Reservation

`allocateContiguous(numPages, maxPages)` maps a large address range upfront but only counts `numPages` against capacity. `growContiguous()` advances the committed size without remapping — critical for hash table growth without data copying.

### AlignedBuffer: Inline Placement-New

`AlignedBuffer` header (exactly 64 bytes) and data payload live in a single contiguous allocation via placement `new`. Data starts at `this + 64`, giving natural cache-line alignment with zero indirection to reach data.

### MallocAllocator: Sharded Counters

`ConcurrentCounter` shards the global allocation counter by thread ID with cache-line-aligned per-shard mutexes. Each shard refills from the global counter in 1MB bulk operations. Small allocations rarely touch the global counter.

## 3. Memory Arbitration: Synchronous, Deterministic, Multi-Phase

### Trigger Path

```
Operator allocates -> leaf pool reserve -> parent chain propagation
  -> root pool: reservationBytes_ + size > capacity_
    -> arbitrator_->growCapacity(pool, size)  [BLOCKS requesting thread]
```

The requesting thread is **synchronously blocked** until memory is available (configurable timeout, default 5 minutes). This contrasts with Trino's asynchronous revocation model.

### Six-Phase Growth Strategy

```
growCapacity(op)
  Phase 1: maybeGrowFromSelf()           -- Pool already has enough (concurrent growth)
  Phase 2: ensureCapacity()              -- Local self-reclaim (spill own data)
  Phase 3: growWithFreeCapacity()        -- Take from arbitrator's free pool
  Phase 4: reclaimUnusedCapacity()       -- Harvest free capacity from ALL other pools
  Phase 5: (if no global arb) local self-reclaim
  Phase 6: startAndWaitGlobalArbitration -- Cross-query spill/abort via controller thread
```

### Global Arbitration Controller

A dedicated background thread coordinates cross-query reclaim. It wakes when any operation enters the global waiter queue, then runs in rounds:

1. Calculate target: `max(sum of waiters' maxGrowBytes, capacity * 10%)`
2. Decide: spill or abort? (abort after 50% of maxArbitrationTime elapsed, default 2.5 min)
3. Select victims via two-dimensional sort: capacity bucket (largest first) × priority (lowest first)
4. Dispatch spill to `memoryReclaimExecutor_` (thread pool, sized at `hardware_concurrency * 0.5`)
5. Collect freed capacity, resume waiters (oldest queries first via ordered map)

### The Reclaim Path

```
Arbitrator -> ArbitrationParticipant (holds reclaimMutex_)
  -> MemoryPool::reclaim()
    -> Task::MemoryReclaimer::reclaimTask()
      1. requestPause() -- wait for all drivers to exit execution loop
      2. MemoryReclaimer::reclaim() on child pools
        -> Operator::MemoryReclaimer::reclaim()
          -> op->reclaim() (e.g., HashBuild spills hash table partitions to disk)
      3. Resume task on exit (RAII guard)
```

### Driver Suspension: Preventing Self-Deadlock

When an operator triggers arbitration for its own pool, the driver enters **suspended state** via `enterSuspended()`, decrementing the task's `numThreads_`. This allows `requestPause()` to complete immediately for the requesting task — the arbitrator can then reclaim from sibling operators in the same task without deadlock.

### NonReclaimableSection Guards

Operators mark critical sections (e.g., mid-hash-table-probe) as non-reclaimable via RAII guards. The arbitrator skips these operators rather than blocking, tracking `numNonReclaimableAttempts` for monitoring.

### Abort Escalation

After `globalArbitrationAbortTimeRatio * maxArbitrationTime` (default 2.5 min), the arbitrator transitions from spill to **query abort**. Victim selection favors the youngest, lowest-priority, largest-capacity query. The abort propagates recursively through the pool tree, causing all future allocations to throw immediately.

### Capacity Growth Strategy

- **Fast phase**: If `2 * currentCapacity <= 512MB`, double the capacity
- **Slow phase**: Grow by 25% of current capacity
- **Initial allocation**: 256MB per new pool from the arbitrator

## 4. The JNI Memory Boundary (Cross-Language Integration)

As described in [VELOX_IMPLEMENTATION_CHALLENGES.md](../VELOX_IMPLEMENTATION_CHALLENGES.md), when Velox runs embedded inside a JVM process (Spark/Gluten via JNI), C++ allocations are invisible to the JVM garbage collector. The `MemoryPool` and `MemoryAllocator` architecture directly addresses this with cross-language hooks:

1. The `reservationCB` callback on every allocation propagates accounting up the pool tree
2. The arbitrator can be configured to call back across JNI to the Java memory manager
3. The Java manager decides whether to grant memory or force Velox to spill

In the pure-C++ Prestissimo deployment, the arbitrator manages its own `capacity_` budget. The same architecture serves both deployment models because the abstraction boundary (pool → arbitrator → allocator) is the same — only the arbitrator's capacity source differs.

## 5. Comparison with Trino and DataFusion

### Memory Pool Architecture

| Dimension | Velox | Trino | DataFusion |
|-----------|-------|-------|------------|
| Pool hierarchy | 4-level tree (Query → Task → Node → Operator) | 3-level tree (Query → Task → Operator) via `MemoryTrackingContext` | Flat `MemoryPool` with optional children |
| Reservation model | Quantized (1/4/8 MB tiers) with slack | Immediate propagation on every allocation | Immediate tracking via `AtomicUsize` |
| Capacity source | Arbitrator-assigned, dynamic | Fixed per-query limit from coordinator | Fixed limit, no dynamic adjustment |
| Thread safety | Per-pool `std::mutex` + `tsan_atomic` | `AtomicLong` counters, no pool-level locks | `AtomicUsize` counters |
| Ownership | `shared_ptr` up / `weak_ptr` down | GC-managed (Java references) | `Arc<dyn MemoryPool>` (Rust ownership) |

### Allocation Mechanisms

| Dimension | Velox | Trino | DataFusion |
|-----------|-------|-------|------------|
| Allocator | `MmapAllocator` (pre-mapped size classes) or `MallocAllocator` | JVM heap (`byte[]`, `long[]`) | jemalloc or system allocator via Arrow |
| Fragmentation strategy | Virtual pre-mapping eliminates fragmentation | JVM GC compaction | Relies on malloc implementation |
| Physical page control | `madvise(MADV_DONTNEED)` for fine-grained return | None (JVM manages heap) | None |
| Buffer layout | Inline placement-new (header + data contiguous) | `Slice` wrapping `byte[]` (one indirection) | Arrow `Buffer` with separate data pointer |
| Hash table growth | `growContiguous()` within pre-reserved address range | Full `long[]` copy | `realloc` |
| Cache integration | Allocator-level `Cache::makeSpace()` | Separate cache management | No built-in cache integration |

### Memory Arbitration

| Dimension | Velox | Trino | DataFusion |
|-----------|-------|-------|------------|
| Trigger model | Synchronous on requesting thread | Asynchronous periodic scheduling (`MemoryRevokingScheduler`) | Self-managed per-operator threshold |
| Arbitration scope | Process-wide cross-query | Per-query revocable memory | Per-operator independent |
| Victim selection | Capacity bucket × priority × age | Most revocable memory | N/A (no cross-query coordination) |
| Spill coordination | Task pause + parallel reclaim via executor | Cooperative revoke flag + operator yield | Operator self-spill |
| Deadlock prevention | Driver suspension (`enterSuspended`/`leaveSuspended`) | Decoupled (scheduler thread triggers, operator thread spills) | N/A |
| Abort escalation | Automatic after timeout (youngest, lowest-priority) | Admin/OOM kill, no automatic escalation | No abort mechanism |
| Fairness | Age-based (oldest queries served first via ordered map) | No explicit fairness | N/A |
| JNI integration | Cross-language callback hooks in `MemoryAllocator` | N/A (pure JVM) | N/A (pure Rust) |
