# Phase 1: Foundation — Data Layout & The Ingestion Bridge

This phase maps Trino's in-memory data representation from the lowest byte-level abstraction up to the Page that flows through the execution engine, and traces the physical path that brings raw S3 bytes into that representation.

## 1. The Data Hierarchy: Slice -> Block -> Page

### Slice — The Byte Substrate
`Slice` (from airlift v2.3) is a bounded view over a heap `byte[]` array with typed, little-endian access via `VarHandle`. It has five fields: `byte[] base`, `int baseOffset`, `int size`, `long retainedSize`, and a lazily-cached `int hash` (XxHash64, benign data race).

Key facts from the source:
- **Heap-only in v2.3.** The historical dual-mode architecture (v0.45 used `Object base + long address` to support both heap and direct/off-heap memory) was removed. Modern Slice wraps only `byte[]`.
- **Unsafe is NOT fully eliminated.** VarHandle replaced Unsafe for single-element typed access (`getInt`, `setLong`, etc.), but `Unsafe` survives in three places: bulk typed-array copies (`copyFromBase`/`copyToBase` via `Unsafe.copyMemory`), `XxHash64` (uses `Unsafe.getLong/getInt/getByte` for high-throughput hashing), and `JvmUtils` (initialization, direct buffer address extraction).
- **Zero-copy slicing:** `slice(offset, length)` returns a new Slice sharing the same `base` array with an adjusted `baseOffset`. The child inherits the parent's `retainedSize`, ensuring accurate memory accounting.
- **Growth strategy:** `Slices.ensureSize()` doubles below 512KB, grows by 1.25x above. Uses `Arrays.copyOfRange` to skip zeroing.

### Block — The Columnar Accessor
`Block` is a **sealed interface** permitting exactly three implementations:
1. **`ValueBlock`** (non-sealed) — concrete physical storage. 11 implementations (`LongArrayBlock`, `VariableWidthBlock`, `IntArrayBlock`, etc.), each a thin wrapper over primitive arrays.
2. **`DictionaryBlock`** — `int[] ids` indexing into a `ValueBlock` dictionary. Lazy projection without data copying.
3. **`RunLengthEncodedBlock`** — a single `ValueBlock` value repeated N times. Constant memory regardless of row count.

The **`arrayOffset` pattern** is the backbone of zero-copy slicing. Both `LongArrayBlock` and `VariableWidthBlock` store an `arrayOffset` field that shifts the logical start within shared backing arrays:
- `LongArrayBlock.getRegion(offset, length)` → new LongArrayBlock with `arrayOffset += offset`, sharing the same `long[] values` and `boolean[] valueIsNull`.
- `VariableWidthBlock.getRegion(offset, length)` → new VariableWidthBlock with `arrayOffset += offset`, sharing the same `Slice`, `int[] offsets`, and `boolean[] valueIsNull`.
- Chained `getRegion()` calls compose additively — all views point to the same backing arrays.

Null representation: `@Nullable boolean[] valueIsNull`. When `null`, the block has zero nulls — `mayHaveNull()` is an O(1) null-pointer check. `build()` drops the array entirely if `hasNullValue` is false. The default `getPositions()` wraps in a `DictionaryBlock` rather than copying data.

### Page — The Passive Envelope
`Page` is a structurally minimal container: `Block[] blocks` + `int positionCount`. Its ~144-byte overhead (10-column page) is negligible. Page stores zero actual data bytes — it is a thin envelope around Block references.

The class provides a vocabulary of zero-copy transformations, all funneling through a package-private `wrapBlocksWithoutCopy` factory that skips the defensive `blocks.clone()` of public constructors:
- **`getColumns(int...)`** — copies Block *references* into a new array. O(columns), never O(rows). The most heavily used transformation across the engine (Aggregator, WindowOperator, TableWriter, PagePartitioner, JoinProbe, etc.).
- **`prependColumn` / `appendColumn`** — creates an N+1 array with `System.arraycopy` for references, attaches the new Block. Used by MarkDistinct, RowNumber, AssignUniqueId.
- **`getRegion(offset, length)`** — delegates to `Block.getRegion()` on each column. View objects with adjusted offsets, no data copy.
- **`compact()`** — the sole mutating operation. Directly modifies `blocks[]` in place. Has special handling for `DictionaryBlock`s sharing the same `DictionaryId`, compacting related dictionaries together.

Size caching: `sizeInBytes` and `retainedSizeInBytes` are `volatile long` fields initialized to `-1`, lazily computed on first access (racy single-check pattern — safe because the computation is deterministic).

## 2. Separation of Data and Compute

Trino strictly separates data from compute. Frameworks like Spark conflate them (a DataFrame holds data *and* provides `.filter()`). In Trino:
- **Data** lives in passive payload objects: Pages and Blocks have no SQL knowledge.
- **Compute** lives in Operators (Phase 3), which consume and produce Pages through the non-blocking Volcano protocol.

This separation means the entire data layer (this phase) can be ported independently of the execution engine.

## 3. From S3 to Memory: The Five-Stage Pipeline

The Parquet/Iceberg read path transforms remote file bytes into Trino Pages through five stages:

```
S3 HTTP Range → Slice → Decompressed Slice → Typed Array → Block → Page
```

| Stage | Representation | Owner | Copy? |
|-------|---------------|-------|-------|
| 1. Remote bytes | HTTP range response `byte[]` | `S3Input` | N/A |
| 2. Raw column chunk | `Slice` (via `Slices.wrappedBuffer`) | `ChunkReader` | Zero-copy wrap |
| 3. Decompressed page | `Slice` (new allocation if compressed) | `PageReader.readPage()` | New alloc |
| 4. Decoded values | `long[]` / `int[]` / `BinaryBuffer` | `ValueDecoder` + `ColumnAdapter` | Bulk decode |
| 5. Trino Page | `Block[]` inside `SourcePage` | `ParquetReader` | Zero-copy wrap |

Key design choices:
- **I/O merge optimization:** `AbstractParquetDataSource.mergeAdjacentDiskRanges()` merges column chunks within 1MB of each other into single range requests (up to 8MB). Multiple logical columns share one physical S3 read via `ReferenceCountedReader` + zero-copy sub-slicing.
- **Lazy column loading:** `ParquetSourcePage.getBlock(channel)` defers column decoding until the block is actually accessed. Columns that are projected but never read (e.g., pruned by dynamic filters) never leave S3.
- **Synchronous, single-threaded I/O per split:** No async I/O or prefetching within a reader. Each driver processes one split at a time with blocking S3 reads. This is the biggest opportunity for a Rust port (async reads + prefetching to hide S3 latency).
- **No row materialization:** The decode path writes directly into typed arrays (`long[]`, `int[]`, `BinaryBuffer`) which are then wrapped zero-copy into Blocks. Rows are never materialized as objects.

## 4. Memory Tracking

Trino relies on the JVM GC for actual deallocation but implements strict **manual accounting**:
- Every `Block` reports two sizes: `getSizeInBytes()` (logical/compacted) and `getRetainedSizeInBytes()` (physical, including over-allocations and object headers).
- `Page.getRetainedSizeInBytes()` sums all blocks' retained sizes plus the Page object overhead.
- As Pages flow through the execution Drivers, their retained size is reported to the worker's `MemoryPool` via the Operator → Driver → Pipeline → Task → Query accounting hierarchy (Phase 5).
- The `retainedSize` for sub-views (via `getRegion()` or `slice()`) always includes the full backing array — preventing under-counting that would break pool accounting.
