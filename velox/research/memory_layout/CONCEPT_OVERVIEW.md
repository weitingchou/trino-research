# Phase 1: Foundation — Memory Layout & The Ingestion Bridge (Velox)

Velox is a C++ vectorized execution engine designed by Meta as an embeddable library. Its memory model sits between Trino's JVM-managed heap and DataFusion's Rust RAII approach — Velox uses explicit `MemoryPool`-tracked allocations with intrusive reference counting and copy-on-write semantics. This phase maps the data hierarchy from the lowest byte-level `Buffer` up through the `RowVector` that flows through the operator pipeline.

## 1. The Data Hierarchy: Buffer → Vector → RowVector

### `Buffer` — The Byte Substrate (Trino's `Slice`, DataFusion's `Buffer`)

`Buffer` (from `velox/buffer/Buffer.h`) is a 64-byte, cache-line-aligned memory wrapper that serves as the backing store for all vector payloads — value arrays, null bitmaps, dictionary indices, and string data.

Key facts from the source:

- **Intrusive reference counting.** Uses `boost::intrusive_ptr<Buffer>` (`BufferPtr`) with an embedded `std::atomic_int32_t referenceCount_`. Eliminates the separate control block that `std::shared_ptr` requires, saving 16+ bytes per buffer and one pointer indirection per refcount operation.
- **Placement-new layout.** `AlignedBuffer` is constructed in-place: `new (memory) AlignedBuffer{pool, capacity}` with `data_ = (uint8_t*)this + 64`. Header and payload form a single contiguous allocation — one pool call to allocate, one to free.
- **Copy-on-write via refcount.** `isMutable() = !isView() && unique()`. When an operator wants to write, it checks this. If shared (refcount > 1) or a view, a new buffer is allocated and data copied. The refcount *is* the mutability gate — no explicit freeze/thaw protocol.
- **Zero-copy slicing.** `Buffer::slice<T>()` creates a `BufferView<BufferReleaser>` that points into the parent's memory. The `BufferReleaser` holds a `BufferPtr` to the parent, keeping it alive. Special case: `slice<bool>` must physically copy when bit offsets are not byte-aligned.
- **POD vs non-POD dispatch at compile time.** `is_pod_like_v<T>` routes to `AlignedBuffer` (memcpy) or `NonPODAlignedBuffer<T>` (placement new/destructor calls). `StringView` is classified as POD-like despite its non-trivial appearance.
- **SIMD-safe padding.** `kPaddedSize` guarantees addressable bytes past `capacity()` for full-width SIMD stores. Debug builds write a sentinel (`0xbadaddbadadddeadUL`) at `data_ + capacity_` to catch overruns.
- **Pool-tracked accounting.** Every allocation flows through `MemoryPool::allocate()` → `reserve()` → `allocateBytes()`, propagating up the 4-level pool hierarchy (query → task → node → operator).

| Aspect | Velox `Buffer` | Trino `Slice` | DataFusion `Buffer` |
|--------|---------------|---------------|---------------------|
| Ownership | Intrusive refcount (`boost::intrusive_ptr`) | GC-managed heap | `Arc<Bytes>` (atomic refcount) |
| Alignment | 64 bytes (cache-line, static_assert enforced) | JVM-dependent | 64/128 bytes (architecture-dependent) |
| Mutability | COW via `unique()` check | Immutable after creation | Immutable by design |
| Deallocation | Deterministic (last ref drop → pool free) | Non-deterministic (GC) | Deterministic (Arc drop) |
| SIMD padding | `simd::kPadding` bytes beyond capacity | None | 64-byte rounding only |

### `BaseVector` / `FlatVector` — The Columnar Accessor (Trino's `Block`, DataFusion's `Array`)

`BaseVector` (from `velox/vector/BaseVector.h`) is the abstract root for all columnar data. `FlatVector<T>` provides direct contiguous storage; `DictionaryVector<T>` provides zero-copy reordering via an indices buffer.

Key facts:

- **Inverted null polarity.** Bit-set (1) = NOT NULL, bit-clear (0) = NULL. This means `countNonNulls()` maps directly to hardware `popcount`, and a freshly allocated nulls buffer (`memset 0xFF`) represents "all not-null" — the common case.
- **Lazy null allocation.** `nulls_` starts as `nullptr` (no nulls). First `setNull(idx, true)` triggers allocation.
- **FlatVector is `final`.** Enables compiler devirtualization in tight loops — `valueAtFast()`, `isNullAt()`, `set()` can all be inlined when the static type is known.
- **DictionaryVector triple-cache.** `setInternalState()` caches `rawIndices_`, `scalarDictionaryValues_`, and `rawDictionaryValues_`. When the base is a non-bool `FlatVector`, `valueAtFast()` resolves to `rawDictionaryValues_[rawIndices_[idx]]` — a single array dereference with zero virtual dispatch.
- **Two-level null model.** Unlike Trino's `DictionaryBlock` (nulls only at its own level), `DictionaryVector::isNullAt()` checks both dictionary-level nulls AND base-level nulls.
- **`wrapInDictionary` optimizations.** Short-circuits constant vectors. Flattens redundant dictionaries when the base is 8x larger than the wrapper (prevents holding large unreferenced bases alive).
- **StringView inlining.** 16-byte non-owning view: strings ≤12 bytes are fully inlined (no pointer chase). Larger strings store a 4-byte prefix + pointer to `stringBuffers_`.

### `RowVector` — The Envelope (Trino's `Page`, DataFusion's `RecordBatch`)

`RowVector` (from `velox/vector/ComplexVector.h`) groups N child `BaseVector` columns under a single row-level nulls bitmap. It is the unit of data flowing between operators via `addInput()`/`getOutput()`.

Key facts:

- **Thin envelope, not a data container.** Holds `std::vector<VectorPtr>` (shared pointers to children) plus an optional nulls bitmap. Structural overhead: ~128 bytes + 16 bytes per column.
- **Mutable (unlike Trino's `Page` and DataFusion's `RecordBatch`).** Children can be swapped, lazy vectors loaded in place, and buffers mutated when singly referenced.
- **Three levels of zero-copy projection** in `Operator::fillOutput()`:
  1. Full identity: `return std::move(input_)` — zero allocation.
  2. Column subset without filter: `shared_ptr` copy (16 bytes per column).
  3. With filter: each column wrapped in `DictionaryVector` (4 bytes/row index, no data copy).
- **Row-level nulls bitmap.** A unique feature vs. Trino (per-Block only) and DataFusion (per-Array only). Enables efficient representation of SQL NULL rows without scanning all columns.
- **Three-tier lazy loading.** Children can be: (a) `nullptr` (unprojected — never materialized), (b) `LazyVector` (projected but deferred until accessed), (c) fully materialized. `containsLazyNotLoaded_` tracks lazy state.
- **`SelectivityVector`** provides word-aligned bitmap row selection. When `isAllSelected()`, `applyToSelected()` compiles to a vectorizable loop. When sparse, uses `__builtin_ctzll` for efficient bit scanning. More space-efficient than DataFusion's `BooleanArray` (1 bit vs 1 byte per row).
- **`DecodedVector`** collapses multi-level dictionary encodings into flat base + one level of indices. Allocated from the system allocator (not pool) for reuse via `LocalDecodedVector`.

| Aspect | Velox `RowVector` | Trino `Page` | DataFusion `RecordBatch` |
|--------|-------------------|--------------|--------------------------|
| Container overhead | ~128 bytes + N×16 bytes | ~40 bytes + N×8 bytes | ~64 bytes + N×16 bytes |
| Null tracking | Row-level bitmap + per-column | Per-Block only | Per-Array only |
| Immutability | Mutable (COW via refcount) | Immutable after construction | Immutable (Arrow model) |
| Column projection | shared_ptr copy | new Page + Block[] copy | Arc clone |
| Row selection | DictionaryVector wrap (4 bytes/row) | DictionaryBlock wrap (4 bytes/row) | New array allocation (full copy) |
| Lazy columns | Native (`LazyVector` child) | Not supported | Not natively supported |

## 2. From Storage to Memory: The DWIO Ingestion Pipeline

Velox's DWIO (Data Warehouse I/O) framework provides a pluggable, format-agnostic pipeline for reading columnar file formats (Parquet, ORC/DWRF, Nimble) into `RowVector`s. All memory is pool-tracked.

### The Four-Phase Pipeline

```
ReadFile → BufferedInput → PageReader → RowVector
(storage)   (I/O batching)  (decode)     (in-memory)
```

| Phase | Component | I/O? | Description |
|-------|-----------|------|-------------|
| 1. Metadata | `ReaderBase::loadFileMetaData()` | Yes | Speculative footer read, Thrift deserialize, schema tree |
| 2. Row Group Scheduling | `ReaderBase::scheduleRowGroups()` | Yes | Stats-based pruning, prefetch N+1 groups, region sort-merge |
| 3. Page Decode | `PageReader::readWithVisitor()` | No | Decompression, rep/def levels, encoding-specific decoders |
| 4. Assembly | `getValues()` | No | Assemble `RowVector` with flat/lazy/constant children |

### Key Design Choices

- **Lazy stream resolution.** `BufferedInput::enqueue()` returns a `SeekableArrayInputStream` wrapping a lambda callback. Actual I/O is deferred until `load()`, which first sorts and merges all enqueued regions for optimal sequential reads.
- **Region sort-merge I/O.** All column chunk byte ranges are sorted by file offset and merged within `kMaxMergeDistance` (1.25 MB). This converts N small reads into fewer large sequential reads — critical for network-attached storage.
- **Row group lifetime management.** `ReaderBase::inputs_` maps row group indices to `shared_ptr<BufferedInput>`. Previous groups are evicted when new ones are scheduled, releasing I/O buffers immediately.
- **Speculative footer read.** Small files (below `filePreloadThreshold`) are loaded entirely in one read. Large files speculatively read `footerSpeculativeIoSize` bytes from the end, avoiding a two-round-trip pattern.
- **AllocationPool bump-pointer.** I/O buffers use `AllocationPool`, which does bump-pointer allocation from `Allocation` runs (16+ pages = 64+ KB). After 256 KB usage, switches to `ContiguousAllocation` with huge page support for better TLB performance.
- **Page-level visitor pattern.** `PageReader::readWithVisitor<Visitor>()` is compile-time specialized on `hasFilter`/`filterOnly`/`hasHook` booleans. Dispatches to encoding-specific decoders (dictionary, direct, delta, string, boolean) without virtual dispatch in the inner loop.
- **Decompression buffer reuse.** `PageReader` reuses `decompressedData_` across pages via `ensureCapacity()`. Since pages within a column chunk have similar sizes, this avoids repeated allocation/deallocation.
- **Format-agnostic framework.** `FormatData`/`FormatParams` polymorphism allows the same `SelectiveColumnReader` base framework to work across Parquet, ORC, and Nimble.

### Comparison with Trino and DataFusion Ingestion

| Dimension | Velox DWIO | Trino `ConnectorPageSource` | DataFusion `ParquetExec` |
|-----------|-----------|---------------------------|--------------------------|
| I/O model | Synchronous with prefetch | Synchronous per-split | Async (`tokio`) with `ParquetPushDecoder` |
| Format framework | Format-agnostic DWIO | Connector SPI per format | Arrow `parquet` crate + `object_store` |
| Byte-range optimization | Sort-merge in `BufferedInput` | Format-reader dependent | `ObjectStore::get_ranges()` batching |
| Decoding | Visitor pattern, compile-time dispatch | Format-specific decoders | Push-based `NeedsData`/`Data`/`Finished` |
| Row group pruning | Stats + filter in `filterRowGroups()` | `ConnectorPageSource` + predicate pushdown | 4-level cascade (file → row group → page → row) |
| Memory tracking | All through `MemoryPool` hierarchy | `MemoryContext` + `getRetainedSizeInBytes()` | `MemoryReservation` RAII |
| Lazy column loading | `LazyVector` children in `RowVector` | Not supported | Not natively supported |

## 3. Memory Tracking Overview

Velox uses explicit C++ memory accounting with a hierarchical pool tree:

```
MemoryManager (process-wide)
  └── Root Pool (per-query, managed capacity)
       └── Task Pool (per-task)
            └── Node Pool (per-plan-node)
                 └── Operator Pool (leaf, allocates)
```

Every `Buffer` allocation flows through `MemoryPool::allocate()` → `reserve()` → `allocateBytes()`. The `reserve()` call propagates up the tree with quantized sizes (1 MB → 4 MB → 8 MB granularity). This provides per-query memory limits and enables the arbitration framework (Phase 5) to find victim queries for spilling.

**Key difference from DataFusion:** DataFusion has a flat pool (no hierarchy, no per-query limits). Velox mirrors Trino's hierarchical tree but in C++ with deterministic deallocation instead of GC.

**Key difference from Trino:** Trino uses `getRetainedSizeInBytes()` to query object sizes after the fact. Velox tracks every allocation at the point of `pool->allocate()` — no separate estimation step.
