# Phase 4: Communication & Interface Overview

To build or rewrite a Trino worker, one must view it as a hub connecting three distinct planes of communication. Each plane uses different protocols and has different performance characteristics.

## 1. The Control Plane (Coordinator <-> Worker)
This is the **Management Interface**. It is characterized by low-frequency, JSON-based REST communication.

* **Instruction Receipt:** The worker acts as a REST server. The coordinator POSTs a `TaskUpdateRequest` to the worker. The plan and splits arrive **separately**: an initial POST delivers the plan fragment, and subsequent updates add splits as they become available. The worker's `SqlTaskManager` uses `needsPlan` optimization to skip re-sending the plan on follow-up updates.
* **The Status Loop:** Trino does not use a persistent "push" connection for status. Instead, the coordinator uses **Long Polling** with version-based change detection. It asks the worker, "What is the status of Task X since version N?" and the worker holds the request open until either the status version advances or a timeout occurs. The wait includes a randomized component to prevent thundering-herd on restart.
* **Dynamic Filter Coordination:** This is the most sophisticated part of the control plane. The lifecycle has **two tiers**:
  - **Task-local (intra-stage):** When build and probe sides of a join run in the same task, `LocalDynamicFilterConsumer` delivers the domain directly through in-memory futures -- no coordinator round-trip needed.
  - **Coordinator-mediated (inter-stage):** When build and probe run on different workers, the domain travels through `DynamicFiltersCollector` on the build-side worker, is fetched by `DynamicFiltersFetcher` via REST polling (triggered by version bumps in `TaskStatus`), aggregated in `DynamicFilterService` on the coordinator, and pushed to scan-side workers via `TaskUpdateRequest`.

  The aggregation uses **three-tier degradation**: first it tries to collect all distinct values as a `Domain`. If the domain exceeds the size limit, it falls back to a **min/max range**. If even that is too large or the row count exceeds the threshold, it collapses to `Domain.all()` (no filtering). This prevents a single high-cardinality join key from consuming unbounded memory on the coordinator.

## 2. The Data Plane (Worker <-> Worker / Shuffle)
This is the **High-Performance Interface**. It is the "Internal Network" of the cluster where the majority of data movement happens.

* **The Pull Model:** Trino shuffles are **pull-based**. An "Upstream" worker (producing data) stores its results in an `OutputBuffer`. For hash-partitioned shuffles, a `PartitionedOutputBuffer` knows exactly how many downstream partitions exist and maintains a dedicated `ClientBuffer` per partition. The downstream workers are responsible for reaching out via HTTP GET to pull results from their assigned partition.
* **Wire Format:** Data is serialized through `CompressingEncryptingPageSerializer` into a binary format: each serialized page has a **12-byte header** (`[positionCount:i32][uncompressedSize:i32][compressedSize:i32]`), followed by block data. Compression (LZ4 or ZSTD) and optional AES-CBC encryption are applied **per-block**, not per-page, preserving the columnar structure from Phase 3.
* **Token-Based Exchange Protocol:** Each downstream consumer tracks a **sequence token** per upstream source. An HTTP GET to `/v1/task/{taskId}/results/{bufferId}/{token}` returns a `BufferResult` containing the pages starting at that token plus a `nextToken` for the follow-up request. This makes the protocol **idempotent**: replaying a request with the same token returns the same pages. An **eager acknowledgment** request is sent asynchronously after receiving pages, allowing the upstream `ClientBuffer` to free memory immediately rather than waiting for the next poll.
* **Flow Control:** Back-pressure is **client-side capacity gating**, not HTTP header signaling. `DirectExchangeClient` tracks the local `StreamingDirectExchangeBuffer` capacity and only dispatches new HTTP requests when `remainingCapacity > 0`, multiplied by a configurable concurrency factor (default 3x). On the producer side, `OutputBufferMemoryManager` blocks drivers via a `SettableFuture` when the buffer exceeds its memory limit.
* **The Coordinator as Final Consumer:** The root stage's `OutputBuffer` is identical to any intermediate shuffle buffer -- there is no special "final result buffer." The difference is entirely on the consumer side: the coordinator's `Query` object uses the same `DirectExchangeClient` / `HttpPageBufferClient` machinery to pull serialized pages. It then deserializes them and converts Block values to JSON via per-type `TypeEncoder` implementations (e.g., `BigintEncoder`, `VarcharEncoder`, `ArrayEncoder`). The client-facing protocol has two phases: `QueuedStatementResource` (`/v1/statement`) handles submission/queuing, and `ExecutingStatementResource` (`/v1/statement/executing`) streams paginated `QueryResults` JSON with `nextUri`-based pagination.

## 3. The Storage Plane (Worker <-> Connector)
This is the **External Interface**. It defines how Trino interacts with the outside world (Iceberg, Delta, Postgres, S3).

* **The SPI Boundary:** The worker does not talk to S3 directly; it talks to a **Connector**. The interface is defined by the **Service Provider Interface (SPI)**.
* **Splits to Pages (Read Path):** The Connector is given a `ConnectorSplit` (an opaque handle representing a file or a range of rows). The Connector's job is to provide a `ConnectorPageSource` that turns that split into a stream of `SourcePage` objects.
* **Three Predicate Pushdown Channels:** The worker pushes predicates into the storage plane through three distinct mechanisms:
  1. **ConnectorTableHandle:** Static predicates baked into the table handle during planning. The connector can use these to skip partitions, files, or row groups at the physical level (e.g., Parquet metadata pruning).
  2. **PageProcessor:** The engine's compiled filter/projection expressions applied by `ScanFilterAndProjectOperator` after pages are read. This catches predicates that the connector cannot evaluate.
  3. **DynamicFilter:** Runtime predicates that narrow over query lifetime. Passed as a live object to `ConnectorPageSourceProvider.createPageSource()`, allowing the connector to re-check `getCurrentPredicate()` before each page read and skip newly-prunable data.
* **Pages to Storage (Write Path):** For `INSERT` and `CREATE TABLE AS` operations, a `TableWriterOperator` pushes pages into a `ConnectorPageSink` via `appendPage()`. When the upstream pipeline is exhausted, `finish()` returns opaque **fragment** descriptors (serialized as `Slice`). These fragments flow upward through an Exchange to the coordinator, where a `TableFinishOperator` collects all fragments and calls `ConnectorMetadata.finishCreateTable()` / `finishInsert()` to **atomically commit** the entire write. This two-phase commit design ensures no metadata changes are visible until the coordinator's final commit succeeds -- providing all-or-nothing semantics. The `abort()` path deletes partially written data on failure.

## 4. Worker Discovery & Cluster Membership

Before any task communication begins, the worker must join the cluster:

* **ANNOUNCE Protocol (default):** Workers POST their URI to the coordinator's `/v1/announce` endpoint every 5 seconds. The coordinator maintains a 30-second TTL cache — if a worker misses ~6 consecutive announcements, it drops out. Three pluggable backends: `ANNOUNCE` (direct POST), `DNS` (SRV records), and `AIRLIFT_DISCOVERY` (legacy service discovery).
* **Health Validation:** The coordinator independently polls each worker's `GET /v1/info` every 5 seconds, verifying the environment name, Trino version, and node lifecycle state (`ACTIVE`, `DRAINING`, `DRAINED`, `SHUTTING_DOWN`). Mismatched workers are marked `INVALID` and excluded from scheduling.
* **Workers Are Topology-Blind:** Workers have zero knowledge of cluster topology. `WorkerInternalNodeManager` is a stub — workers only know their own identity and the coordinator URI.

## 5. Wire Protocol: Exact Serialization Formats

Building a compatible worker requires bit-level fidelity across four serialization domains.

### 5.1. Plan Fragment & Expression JSON

The coordinator sends a `TaskUpdateRequest` JSON payload containing the physical plan:

* **PlanFragment (12 fields):** The top-level envelope carries `id`, `root` (a recursive plan node tree), `symbols` (with custom length-prefixed key encoding: `"7|bigint|col_a"`), `partitioning` (`PartitioningHandle`), `outputPartitioningScheme`, `partitionedSources`, and `activeCatalogs`.
* **47 PlanNode types** are polymorphically tagged via `@JsonTypeInfo` with an `"@type"` discriminator field (e.g., `"tableScan"`, `"join"`, `"filter"`, `"aggregation"`, `"exchange"`). Nodes reference children directly as nested JSON objects.
* **Expressions** are a JSON-serialized AST (not bytecode) — a sealed hierarchy of **18 `Expression` subtypes** (`Reference`, `Constant`, `Call`, `Comparison`, `Cast`, `Logical`, `Case`, `Switch`, `Between`, `In`, `IsNull`, `Coalesce`, `NullIf`, `Array`, `Row`, `Lambda`, `Bind`, `FieldReference`). All are Java records; component names become JSON field names.
* **Constants** are serialized as **base64-encoded Trino Block binary** (not JSON primitives). A BIGINT literal `10` becomes a base64-encoded `LongArrayBlock` with one element.
* **Function references** are fully resolved by the coordinator: each `Call` node carries a `ResolvedFunction` with `FunctionId`, `BoundSignature` (name + concrete types), `CatalogHandle`, and behavioral flags (`deterministic`, `neverFails`). Workers never perform function lookup.
* **Types** serialize as **flat SQL-parseable strings** (`"bigint"`, `"varchar(256)"`, `"decimal(38,18)"`, `"row(\"x\" bigint,\"y\" integer)"`), not structured JSON objects.
* **Connector handles** (table handles, column handles, split handles) use a separate `AbstractTypedJacksonModule` with `@type` values in `"classLoaderId:fullyQualifiedClassName"` format (e.g., `"hive:io.trino.plugin.hive.HiveTableHandle"`).

### 5.2. TaskUpdateRequest & TaskStatus JSON

* **TaskUpdateRequest (10 fields):** A single idempotent message for both create and update. `fragment` (PlanFragment) is sent only on the first update; the coordinator uses `needsPlan` feedback to stop re-sending. `session` (29 fields including `queryId`, `transactionId`, `user`, `catalog`, `schema`, `systemProperties`, `catalogProperties`) travels with every request. `sources` (split assignments) are incremental, with `sequenceId` for idempotent delivery.
* **TaskStatus (22 fields):** Lightweight heartbeat record polled at ~5s cadence. Version-monotonic — `Long.MAX_VALUE` on terminal states forces immediate coordinator notification.
* **TaskInfo:** Full response (superset of TaskStatus) with nested `TaskStats` (44 fields) → `PipelineStats` (39 fields) → `OperatorStats` (40 fields) → `DriverStats` (27 fields). Hierarchical `summarize()` strips detail levels for bandwidth control.
* **Duration** serializes as airlift's custom format: `"15.50ms"`, `"2.00s"` (not ISO-8601). **DataSize** serializes as human-readable: `"3.50MB"`, `"12B"`.
* **OperatorInfo** is polymorphic (8 `@JsonSubTypes` via `@type` discriminator). **Metrics** use `@JsonTypeInfo(use = Id.CLASS)` — full Java class names as discriminators.

### 5.3. Page Binary Wire Format

Data flows between workers as a four-layer binary format:

```
HTTP Response Envelope (16 bytes)
  magic: 0xfea4f001 (4B) + xxHash64 checksum (8B) + pageCount (4B)
  |
  +-- Page (12-byte header per page)
       positionCount (4B) + uncompressedSize (4B) + compressedSize (4B)
       |
       +-- channelCount (4B)
       +-- Block × channelCount (self-describing)
            encoding name: length-prefixed UTF-8 (e.g., "LONG_ARRAY")
            block payload: type-specific binary data
```

* **13 block encoding types:** `LONG_ARRAY`, `INT_ARRAY`, `SHORT_ARRAY`, `BYTE_ARRAY`, `INT128_ARRAY`, `FIXED12`, `VARIABLE_WIDTH`, `DICTIONARY`, `RUN_LENGTH`, `ROW`, `ARRAY`, `MAP`, `VARIANT`.
* **Null bitmaps** are bit-packed, **MSB-first** (position 0 → bit 7 of byte 0). This differs from Apache Arrow's LSB-first convention.
* **Null compaction:** Primitive blocks write only non-null values, preceded by a `nonNullCount` int32. This is the primary wire-format size optimization.
* **All multi-byte integers are little-endian.**
* **Compression** is applied to **64 KB chunks** of the raw page payload (not per-block). Each chunk has a 4-byte marker (MSB = isCompressed, lower 31 bits = size). No codec identifier in the wire format — both sides must agree via session properties.
* **Encryption** (optional): per-chunk AES-256-CBC with random 16-byte IV prepended.

### 5.4. Exchange HTTP Protocol

Three endpoints, seven custom headers:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/task/{taskId}/results/{bufferId}/{token}` | GET | Fetch pages starting at sequence token |
| `/v1/task/{taskId}/results/{bufferId}/{token}/acknowledge` | GET | Fire-and-forget ACK for memory release |
| `/v1/task/{taskId}/results/{bufferId}` | DELETE | Destroy buffer |

| Header | Direction | Value |
|--------|-----------|-------|
| `X-Trino-Max-Size` | Request | Max response body size in bytes |
| `X-Trino-Internal-Bearer` | Request | JWT authentication (HKDF-SHA256 derived) |
| `X-Trino-Task-Instance-Id` | Response | UUID identifying the task instance |
| `X-Trino-Page-Sequence-Id` | Response | Starting sequence token of returned pages |
| `X-Trino-Page-End-Sequence-Id` | Response | Token for next fetch request |
| `X-Trino-Buffer-Complete` | Response | Boolean: all pages have been produced |
| `X-Trino-Task-Failed` | Response | Boolean: task has failed |

Empty buffer returns HTTP 204 (no content); non-empty returns 200 with binary body.

### 5.5. Partition Hash Function

Shuffle partitioning uses a four-stage pipeline:

1. **Per-type hash:** Fixed-width types use a custom single-round xxHash64 mix: `rotateLeft(value * 0xC2B2AE3D27D4EB4F, 31) * 0x9E3779B185EBCA87`. Variable-width types (VARCHAR, VARBINARY) use the full xxHash64 algorithm. `NULL_HASH_CODE = 0`.
2. **Multi-column combining:** `hash = 31 * previousHash + columnHash` (polynomial, initial seed = 0).
3. **Hash-to-partition:** Lemire "fastrange" multiply-shift: `(Integer.toUnsignedLong(Long.hashCode(rawHash)) * partitionCount) >>> 32`. Avoids modulo bias without division.
4. **Bucket-to-partition indirection:** A `bucketToPartition` array decouples hash buckets from output partitions.

### 5.6. Split JSON Formats

Splits are wrapped in a four-level nesting: `SplitAssignment` (plan node ID + splits + `noMoreSplits` flag) → `ScheduledSplit` (`sequenceId` for idempotent delivery) → `Split` (`CatalogHandle` string + polymorphic `ConnectorSplit`) → concrete split.

* **HiveSplit (17 fields):** file path, start offset, length, file size, partition keys, bucket number, storage format, S3 region. Addresses excluded via `@JsonIgnore`.
* **IcebergSplit (15 fields):** file path, offset, length, file format (PARQUET/ORC), delete files, partition data as pre-serialized JSON, `fileStatisticsDomain` as `TupleDomain`.
* **Polymorphic `@type`:** `"hive:io.trino.plugin.hive.HiveSplit"` (classloader ID + FQCN).

## Summary: The Hub Architecture

1.  **Worker** registers with the **coordinator** via **ANNOUNCE** (POST URI every 5s). Coordinator validates health via `/v1/info` polling.
2.  **Coordinator** sends a **TaskUpdateRequest** (Plan fragment as JSON AST with 47 PlanNode types + 18 Expression types, then Splits incrementally) via **REST/JSON**.
3.  **Worker** starts **Drivers** to process **Splits** via the **Connector SPI**.
4.  **Connector** streams raw data as **SourcePages** into the Worker, with predicates pushed through three channels.
5.  **Worker** transforms data, serializes into a **four-layer binary format** (16-byte HTTP envelope → 12-byte page header → channel count → self-describing blocks with MSB-first null bitmaps and 64KB chunk compression), and places pages in an **Output Buffer**.
6.  **Downstream Workers** (or Coordinator) fetch data via **token-based HTTP pull** with 7 custom headers, client-side capacity gating, and fire-and-forget ACKs.
7.  Shuffle partitioning uses a **custom xxHash64 mix** with **Lemire fastrange** reduction for bias-free partition assignment.
8.  **Worker** reports **TaskStatus** (22 fields) with version-based long-polling; **Dynamic Filters** flow through a two-tier coordination path.
9.  For writes, **fragment descriptors** flow upward to the Coordinator for **atomic two-phase commit** via the Connector SPI.
