# KG-ICE-3: Rust Iceberg/Parquet Ecosystem Assessment

**Date:** 2026-04-09
**Status:** Complete
**Purpose:** Evaluate the Rust Iceberg and Parquet ecosystem for building a native Rust Iceberg connector in the Trino-compatible worker (Project Crucible/Forge). Assess readiness, gaps, and risks against Trino's production Java Iceberg connector.

---

## 1. iceberg-rust (Apache)

### 1.1 Project Overview

The `apache/iceberg-rust` project is the official Rust implementation of the Apache Iceberg table format. It is an Apache Software Foundation project with active governance and community participation.

| Metric | Value |
|--------|-------|
| GitHub Stars | ~1,300 |
| Forks | ~446 |
| Language | 98.7% Rust |
| Current Version | **0.9.0** (released 2025-03-18) |
| Previous Versions | 0.8.0 (2025-01-16), 0.7.0 (2024-10-10), 0.6.0 (2024-07-30), 0.5.1 (2024-06-05) |
| Release Cadence | ~Quarterly (consistent since mid-2024) |
| 0.9.0 Contributors | 28 contributors, 109 PRs merged |
| 0.8.0 Contributors | 37 contributors, 144 PRs merged |
| Commits (main) | ~1,165 |

The crate ecosystem is structured as a workspace of multiple crates:
- `iceberg` -- core library (table, scan, metadata, schema, spec types)
- `iceberg-catalog-rest` -- REST catalog implementation
- `iceberg-catalog-hms` -- Hive Metastore catalog
- `iceberg-catalog-glue` -- AWS Glue catalog
- `iceberg-s3tables-catalog` -- AWS S3 Tables catalog
- `iceberg-datafusion` -- DataFusion `TableProvider` integration
- `iceberg-storage-opendal` -- Storage abstraction via Apache OpenDAL

### 1.2 Catalog Support

**Supported catalogs (all shipped as separate crates):**

| Catalog | Crate | Status |
|---------|-------|--------|
| REST Catalog | `iceberg-catalog-rest` | Production-ready. Improved auth in 0.8.0 with public types and documentation. Recommended for new deployments. |
| Hive Metastore (HMS) | `iceberg-catalog-hms` | Supported. Thrift-based integration. |
| AWS Glue | `iceberg-catalog-glue` | Supported since 0.7.0 via catalog loader. |
| AWS S3 Tables | `iceberg-s3tables-catalog` | Supported since 0.7.0. Uses `aws-sdk-s3tables`. |
| In-Memory | `iceberg-catalog-memory` | Available for testing/development. |

**Not supported (present in Trino):**
- JDBC catalog
- Nessie catalog
- Snowflake catalog

**Assessment:** For our target Lakehouse architecture over S3, the REST and Glue catalogs are the critical ones, and both are available. However, **for the Rust worker, catalog interaction is a coordinator responsibility** -- the worker receives `IcebergSplit` definitions from the coordinator. The worker only needs to parse split metadata and read data files. Catalog support in iceberg-rust is most relevant if we use the library for scan planning validation or testing.

### 1.3 Scan Planning & Pruning

iceberg-rust implements a multi-layered scan planning pipeline:

**Manifest pruning:**
- `ManifestEvaluator` filters manifests using `field_summary` bounds (upper/lower bound of partition values per manifest).
- Skips entire manifests whose partition ranges do not intersect the query predicate.

**Data file pruning:**
- `ExpressionEvaluator` evaluates predicates against per-data-file partition values.
- Filters data files within manifests based on partition struct evaluation.

**Predicate pushdown to Parquet reader:**
- Predicates are converted from Iceberg expressions to Arrow filter expressions.
- Pushed down into the Parquet reader for row-group and page-level pruning.

**Row selection filtering:**
- Row-level filtering applied during record batch construction.

**LIMIT pushdown (new in 0.9.0):**
- `IcebergTableScan` tracks a row limit field.
- Limit applied at stream level by filtering/slicing record batches, stopping I/O once the limit is reached.

**Manifest caching:**
- Object Cache for parsed `Manifest` and `ManifestList` objects, reducing redundant metadata file reads.

**What is present:** The full Iceberg scan planning pipeline -- manifest pruning, partition pruning, predicate pushdown to Parquet, row filtering, and limit pushdown -- is implemented.

**What is missing/uncertain:**
- Incremental scan between two snapshots (open issue #1469; Java SDK supports this for streaming reads).
- The depth of Parquet statistics-based pruning within iceberg-rust's integration (vs. delegating to arrow-rs) needs source-level verification.

### 1.4 Delete File Support

Delete file support has been a major development focus through the 0.5.x-0.8.x releases:

**Positional delete files:** Supported. The reader applies positional deletes by matching delete file entries to data files based on file path and row positions.

**Equality delete files:** Supported. The reader parses equality delete files, builds predicates from them, and applies the predicates during data file reading. Case-sensitive matching is supported.

**Mixed delete types:** Supported since 0.8.0. A bug was fixed where `FileScanTask` objects containing both positional and equality delete files were not handled correctly. The condition in `try_start_eq_del_load` was inverted, causing equality delete files to be skipped when not cached. This is now fixed and tested with `test_load_deletes_with_mixed_types`.

**Delete file caching:** Shared delete file caching is implemented to avoid redundant reads when multiple data files reference the same delete file.

**Binary type support:** Added in 0.8.0 for equality delete processing.

**Assessment:** Delete file *reading* (MOR read path) is functional for both V2 delete types. This is the critical capability for the worker, since the worker receives splits that may reference delete files and must apply them during reads.

**What is missing:**
- **Writing delete files** (producing positional or equality delete files as output of UPDATE/DELETE operations) -- this is a coordinator/commit concern but may be needed for write path.
- **V3 deletion vectors** (DVs) -- iceberg-rust has V3 metadata support (added in 0.6.0) but DV reading/writing support status is unclear and likely incomplete.

### 1.5 Schema Evolution

**Read-side schema evolution:**
- iceberg-rust uses Iceberg's field-ID-based column tracking, which is the foundation for schema evolution.
- Position-based column projection for migrated Parquet files was added in 0.8.0, handling the case where Parquet files written by older writers (e.g., Hive) use positional column mapping rather than field-ID-based mapping.
- The reader can project data files written under older schemas to the current query schema by matching on field IDs.

**Write-side schema evolution (Transaction API):**
- iceberg-rust supports a `Transaction` API with `update_schema()` and `commit()` methods.
- Schema changes are metadata-only operations (no data file rewrite required).
- Supported operations: add column, drop column, rename column, type widening.

**Assessment:** Schema evolution is functional for the read path, which is the primary worker concern. The worker must read Parquet files potentially written under different historical schemas and project them to the coordinator's expected output schema. The field-ID-based approach and the position-based fallback for migrated files cover the main scenarios.

### 1.6 DataFusion Integration

The `iceberg-datafusion` crate provides deep DataFusion integration:

**TableProvider implementations:**

| Type | Description | Use Case |
|------|-------------|----------|
| `IcebergTableProvider` | Catalog-backed, automatic metadata refresh | Write operations, latest-state queries |
| `IcebergStaticTableProvider` | Static snapshot-based, read-only | Analytical queries, time travel |

**Supported operations:**
- **Read/scan:** Both providers implement DataFusion's `TableProvider` trait with `scan()`.
- **Predicate pushdown:** `supports_filters_pushdown()` is implemented. Predicates from DataFusion's optimizer are converted to Iceberg expressions and pushed into `TableScanBuilder` via `expr_to_predicate.rs`. Boolean column predicates are handled (bare boolean -> `column = true`, NOT boolean -> `column = false`).
- **LIMIT pushdown:** Integrated into `IcebergTableScan` (0.9.0).
- **INSERT INTO:** `insert_into()` implemented for `IcebergTableProvider` (0.8.0), including partitioned tables.
- **DROP TABLE:** Supported in DataFusion context (0.9.0).
- **Metadata tables:** `metadata_table` module for querying Iceberg metadata (snapshots, manifests, history, refs, partitions, etc.).
- **SchemaProvider and CatalogProvider:** Traits are implemented for catalog-level browsing.

**What this means for the Rust worker:**
The DataFusion integration is *reference architecture* for our connector, not a direct dependency. Our worker does not run DataFusion's query planner -- it executes Trino physical plan fragments. However:
1. The `IcebergStaticTableProvider` demonstrates how to build a scan from a known snapshot + predicate + projection -- exactly what our connector must do from an `IcebergSplit`.
2. The predicate conversion logic (`expr_to_predicate.rs`) is directly reusable or adaptable for converting Trino predicates to Iceberg expressions.
3. The delete file application pipeline in the DataFusion integration is the closest Rust reference for our MOR implementation.

### 1.7 Maturity Assessment

**Strengths:**
- Quarterly releases with growing contributor base (28-37 contributors per release).
- Active Apache project with proper governance.
- Core read path is production-quality: scan planning, predicate pushdown, delete files, schema evolution.
- Multiple production adopters (RisingWave uses it for Iceberg compaction; Databend integrates with scan planning).
- Spec V1/V2 support is comprehensive; V3 metadata support added.

**Weaknesses:**
- Still pre-1.0 (currently 0.9.0). API stability is not guaranteed between minor versions.
- Write path is less mature than read path (FastAppend is primary; MergeAppend, overwrite, compaction are limited).
- Table maintenance operations (expire snapshots, rewrite manifests, compact data files) are not implemented -- these are Java-SDK or Spark-procedure features.
- V3 deletion vectors and Puffin file support are incomplete.
- No JDBC, Nessie, or Snowflake catalog support.
- Incremental (streaming) scans between two snapshots not yet supported.

**Risk level for our use case: MODERATE.** The read path -- which is the worker's primary concern -- is mature and actively tested. The pre-1.0 status means we should pin versions carefully and be prepared for API changes. The write path gaps are less critical since the worker's write responsibilities are limited (write Parquet files, return fragment metadata to coordinator for commit).

---

## 2. arrow-rs/parquet Crate

The `parquet` crate from the `arrow-rs` project is the official Rust implementation of Apache Parquet. It is mature, heavily used (DataFusion, DuckDB-rs, Polars, Delta Lake, iceberg-rust all depend on it), and under very active development.

### 2.1 Async Reading

**`ParquetRecordBatchStreamBuilder`** is the primary async reading API:

- Reads Parquet file metadata (footer) asynchronously.
- Returns a `ParquetRecordBatchStream` that yields `RecordBatch` objects.
- Supports configurable batch sizes (`with_batch_size`).
- Supports column projection (`with_projection` via `ProjectionMask`).
- Supports row group selection (`with_row_groups`).
- Supports row-level filtering (`with_row_filter` via `RowFilter`).
- Supports offset-based range reads for cloud storage optimization.

**Integration with `object_store` crate:**
The async reader works with any `AsyncRead + AsyncSeek` implementation, which includes the `object_store` crate's S3/GCS/Azure readers. This enables direct reading from object storage without intermediate buffering.

**Performance optimizations (recent):**
- **Predicate evaluation cache** for async reader (PR #7850, Aug 2025) -- caches predicate evaluation results to avoid redundant computation.
- **Adaptive predicate pushdown** (PR #8733, Nov 2025) -- introduces mask-backed execution for short selectors with heuristics to choose between masks and selectors.

### 2.2 Predicate Pushdown

The `parquet` crate implements a multi-level predicate pushdown pipeline:

**Level 1 -- Row Group Pruning (Statistics):**
- Each row group has footer statistics: min/max values, null counts per column.
- Query predicates are evaluated against these statistics to skip entire row groups.
- Supported for all primitive types.

**Level 2 -- Page Index Pruning:**
- Optional Parquet page index stores per-page min/max/null-count statistics.
- When present, enables pruning at the page level (much finer granularity than row groups).
- The decoder only processes pages whose row ranges survive the filter.

**Level 3 -- Bloom Filter Pruning:**
- Parquet Split Block Bloom Filter (SBBF) support added in arrow-rs 28.0.0.
- Both reading and writing bloom filters are supported.
- Effective for high-cardinality equality predicates (e.g., `id = X`) where min/max statistics are useless.

**Level 4 -- Row-Level Filtering (`RowFilter`):**
- `RowFilter` applies predicates in order during decoding.
- Decodes only the columns required for predicate evaluation first.
- As predicates eliminate rows, fewer rows from subsequent columns need decoding.
- Any `RowSelection` from page index pruning is applied before the first predicate.
- This is the deepest pushdown level, operating within the Parquet decoder itself.

**`RowSelection` API:**
- Decouples the Parquet decoder from the query engine.
- Represents a set of row ranges to include/exclude.
- Page index pruning produces a `RowSelection` that is fed into the decoder.

### 2.3 Column Projection

- `ProjectionMask` specifies exactly which columns to read.
- Supports both leaf-column indices and field-ID-based (schema-level) projection.
- Unselected columns are never decoded or even read from storage (I/O is skipped entirely for object storage backends that support range requests).
- Nested/struct column projection is supported.

**Schema handling:**
- The reader returns a projected `SchemaRef` matching the requested projection.
- Schema metadata is available via `ParquetRecordBatchStreamBuilder::schema()`.
- For schema evolution scenarios, the caller (iceberg-rust) is responsible for mapping between the file's physical schema and the query's logical schema using field IDs, then constructing the appropriate `ProjectionMask`.

### 2.4 Performance

**Architecture advantages:**
- Zero-copy where possible: Arrow buffers are directly populated from decoded Parquet data.
- SIMD-accelerated decoding via Rust/LLVM auto-vectorization for bit-unpacking, dictionary decoding, and delta encoding.
- Async I/O with Tokio integration prevents thread blocking on storage reads.
- Memory-efficient: only projected columns and surviving rows are materialized.

**Recent performance work:**
- Adaptive predicate pushdown (Nov 2025) optimizes the selection strategy based on selectivity.
- Predicate evaluation caching (Aug 2025) eliminates redundant computation in the async path.
- The DataFusion Comet team contributed reader performance improvements that landed in iceberg-rust 0.9.0, benefiting the arrow-rs Parquet reader as well.

**Benchmark context:**
- The C++ Arrow Parquet reader achieves ~4 GB/s throughput for full-table scans.
- The Rust implementation is expected to be in the same ballpark due to similar LLVM code generation and memory layout.
- No direct published benchmarks of arrow-rs/parquet vs. Java parquet-mr, but the architectural advantages (no GC, no boxing, zero-copy Arrow buffers, SIMD) strongly favor Rust for CPU-intensive decode workloads.
- DataFusion (which uses this reader) regularly benchmarks competitively against Trino and other JVM engines on TPC-H/TPC-DS.

**Assessment:** The arrow-rs/parquet crate is production-grade and is the most mature component in the Rust data ecosystem. It is the standard Parquet reader for every major Rust data project. For our Rust worker, this is a zero-risk dependency.

---

## 3. Gap Analysis vs. Trino Java Connector

This section compares capabilities of `iceberg-rust` + `arrow-rs/parquet` against the Trino 480 Java Iceberg connector, focusing on what the **Rust worker** needs (primarily the read path, with limited write support).

### Reference: Trino Java Iceberg Connector Feature Set

For context, the Trino Java Iceberg connector provides:
- **Catalogs:** REST, HMS, Glue, JDBC, Nessie, Snowflake
- **Spec versions:** V1, V2, V3 (experimental)
- **File formats:** Parquet, ORC, Avro
- **Read path:** Full scan planning with manifest/partition/data-file pruning, predicate pushdown, column projection, schema evolution, time travel (snapshot ID / timestamp), branch/tag references, metadata tables
- **Delete files:** Positional deletes (V2), equality deletes (V2), deletion vectors (V3 experimental)
- **Write path:** INSERT, INSERT OVERWRITE, UPDATE, DELETE, MERGE, sorted writing, file rolling
- **Table operations:** CREATE, ALTER (schema evolution, partition evolution, properties), DROP, COMMENT
- **Maintenance:** ANALYZE (column statistics), OPTIMIZE (compaction), expire_snapshots, remove_orphan_files
- **Advanced:** Materialized views (incremental/full refresh), dynamic filtering, file system caching, metadata caching

### 3.1 Ready to Use

These capabilities are available in `iceberg-rust` + `arrow-rs/parquet` today and can be used with minimal integration work:

| Capability | Rust Ecosystem Status | Notes |
|------------|----------------------|-------|
| **Parquet async reading** | arrow-rs: Production-grade | `ParquetRecordBatchStreamBuilder` with full async support |
| **Column projection** | arrow-rs: Production-grade | `ProjectionMask` with nested column support |
| **Row group pruning (statistics)** | arrow-rs: Production-grade | Min/max/null-count statistics evaluation |
| **Page index pruning** | arrow-rs: Production-grade | Per-page statistics when available in Parquet files |
| **Bloom filter pruning** | arrow-rs: Production-grade | SBBF read/write since arrow-rs 28.0 |
| **Row-level filtering** | arrow-rs: Production-grade | `RowFilter` + `RowSelection` API |
| **Manifest pruning** | iceberg-rust: Implemented | `ManifestEvaluator` with field summary bounds |
| **Partition pruning** | iceberg-rust: Implemented | `ExpressionEvaluator` on partition struct |
| **Predicate pushdown to Parquet** | iceberg-rust: Implemented | Iceberg expressions converted to Arrow filters |
| **Positional delete files** | iceberg-rust: Implemented | File-path + row-position matching |
| **Equality delete files** | iceberg-rust: Implemented | Predicate construction from delete files, case-sensitive |
| **Mixed delete file types** | iceberg-rust: Fixed in 0.8.0 | Both position + equality on same `FileScanTask` |
| **Delete file caching** | iceberg-rust: Implemented | Shared cache across data files |
| **Schema evolution (read)** | iceberg-rust: Implemented | Field-ID-based projection + positional fallback for migrated files |
| **S3 object storage** | OpenDAL + object_store: Production-grade | Full S3/GCS/Azure support |
| **REST catalog** | iceberg-rust: Production-ready | Auth improvements in 0.8.0 |
| **Glue catalog** | iceberg-rust: Supported | Catalog loader since 0.7.0 |
| **LIMIT pushdown** | iceberg-rust: Implemented (0.9.0) | Stream-level limit application |
| **Manifest caching** | iceberg-rust: Implemented | Object cache for parsed manifests/manifest lists |
| **Iceberg V1/V2 spec** | iceberg-rust: Supported | Core spec compliance |
| **Metadata tables** | iceberg-rust: Supported | SNAPSHOTS, MANIFESTS, FILES, HISTORY, REFS, PARTITIONS, etc. |

### 3.2 Partially Implemented

These capabilities exist but have gaps, limitations, or need additional integration work:

| Capability | Status | Gap Description |
|------------|--------|-----------------|
| **Schema evolution (write/DDL)** | Partial | Transaction API supports `update_schema()` + `commit()` for add/drop/rename column. However, the worker does not manage schema DDL -- this is coordinator-driven. The gap is mainly in testing maturity. |
| **Time travel / snapshot selection** | Partial | iceberg-rust can load specific snapshots, and `IcebergStaticTableProvider` supports snapshot-based access. However, incremental scan between two snapshots (for streaming) is not supported (issue #1469). For our worker, the coordinator selects the snapshot and encodes it in the split, so basic snapshot loading suffices. |
| **Write path (Parquet file writing)** | Partial | FastAppend is supported. DataFusion `insert_into` for partitioned tables works (0.8.0). Clustered and fanout writers added (0.8.0). However, MergeAppend and overwrite operations are less mature. For the worker, the write path is: produce Parquet files -> return fragment metadata -> coordinator commits. This simpler path is covered. |
| **V3 metadata format** | Partial | V3 metadata parsing support added in 0.6.0. However, V3-specific features (deletion vectors, Puffin files) are not fully implemented. |
| **Hidden partitioning transforms** | Partial | Iceberg partition transforms (year, month, day, hour, truncate, bucket) need to be correctly applied during scan planning and write partitioning. iceberg-rust supports the partition spec model, but coverage of all transform types in predicate evaluation needs source-level verification. |
| **Predicate conversion (Trino -> Iceberg)** | Partial | iceberg-rust has `expr_to_predicate.rs` for DataFusion expressions. We need an equivalent converter for Trino `TupleDomain`/`Domain` predicates that arrive in the plan fragment JSON. This is custom work but the iceberg-rust predicate model is the target. |

### 3.3 Missing / Custom Implementation Needed

These capabilities are either absent from the Rust ecosystem or require significant custom implementation:

| Capability | Trino Java Status | Rust Ecosystem Status | Custom Work Required |
|------------|-------------------|----------------------|---------------------|
| **ORC format support** | Full read/write | Not in arrow-rs (Parquet only) | **Not needed** if we target Parquet-only Lakehouse. If ORC is required, `orc-rust` crate exists but is less mature. |
| **Avro format support** | Full read/write | `apache-avro` crate exists | **Not needed** for data files (Parquet-only target). Avro is used by Iceberg for manifests, which iceberg-rust handles internally. |
| **Trino `IcebergSplit` parsing** | Native Java class | Does not exist | **Must build.** Parse the JSON split definition from the coordinator's `TaskUpdateRequest`. Extract: data file path, file format, partition values, delete file list, file I/O properties, predicate (as `TupleDomain`). |
| **Trino `TupleDomain` -> Iceberg predicate** | Internal Java conversion | Does not exist | **Must build.** Convert Trino's `TupleDomain<IcebergColumnHandle>` representation (from plan fragment JSON) into iceberg-rust's expression model for predicate pushdown. |
| **Trino page output format** | `IcebergPageSource` produces Trino `Page` | No Trino Page support | **Must build.** Convert Arrow `RecordBatch` to Trino's `Page`/`Block` wire format (12-byte header binary protocol). This is in `trw-protocol` crate scope, not Iceberg-specific. |
| **Dynamic filter integration** | Coordinator sends dynamic filters during execution | Does not exist | **Must build.** Receive dynamic filter updates from coordinator, convert to additional predicates, and apply mid-scan. This is a worker-level feature, not Iceberg-specific. |
| **Materialized views** | Full support (incremental + full refresh) | Not applicable | **Not needed** for worker. Materialized view refresh is coordinator-orchestrated. |
| **ANALYZE (column statistics)** | Collects NDV, min/max, histogram | Not implemented in iceberg-rust | **Not needed** for initial worker. ANALYZE is a coordinator-initiated command. |
| **OPTIMIZE (compaction)** | Rewrites small files into larger ones | Not implemented in iceberg-rust | **Not needed** for worker. Compaction is a maintenance operation. |
| **Expire snapshots / orphan cleanup** | Table maintenance procedures | Not implemented in iceberg-rust | **Not needed** for worker. Maintenance is coordinator/external tooling responsibility. |
| **JDBC / Nessie / Snowflake catalogs** | Supported | Not available | **Not needed** for worker (catalog is coordinator's domain). Only relevant if worker needs to validate catalog operations in tests. |
| **Sorted writing with staging** | Configurable sort order during writes | Clustered/fanout writers exist | **Partial custom work.** If the coordinator requests sorted output, the worker needs to sort before writing. DataFusion's `SortExec` (integrated in 0.9.0) or custom sort operator can handle this. |
| **V3 deletion vectors (DVs)** | Experimental support | Not implemented | **Deferred.** V3 DV support is experimental in Trino too. Not needed for initial MVP. |
| **File system caching** | Caches file handles/metadata | Not implemented as unified layer | **May need custom work.** Can leverage Rust's async caching patterns or OpenDAL's caching layer. Lower priority for initial implementation. |

---

## 4. Recommendations

### 4.1 Architecture Decision

**Use `iceberg-rust` as a library dependency, not as a framework.**

The worker should depend on the `iceberg` core crate for:
- Iceberg metadata types (schema, partition spec, manifest, snapshot)
- Expression model and evaluators (for predicate conversion)
- Delete file reading and application
- Schema evolution (field-ID-based projection)

The worker should depend on `arrow-rs/parquet` directly for:
- Async Parquet reading with `ParquetRecordBatchStreamBuilder`
- Predicate pushdown via `RowFilter` and `RowSelection`
- Column projection via `ProjectionMask`
- Row group and page index pruning

The worker should **not** depend on `iceberg-datafusion` directly, since we are not using DataFusion's query planner. However, it serves as reference architecture for:
- How to wire predicate pushdown from a query engine into iceberg-rust's scan API
- How to apply delete files during scan execution
- How to handle schema evolution in the read path

### 4.2 Critical Custom Work (Priority Order)

1. **`IcebergSplit` parser** -- Parse coordinator's split JSON into a struct containing file path, format, partition data, delete file references, and serialized predicate. This is the entry point for all connector operations.

2. **Trino predicate converter** -- Convert `TupleDomain` from the plan fragment JSON into iceberg-rust's `BoundPredicate` / `Expression` types. Reference: `expr_to_predicate.rs` in `iceberg-datafusion`.

3. **Scan executor** -- Given a parsed `IcebergSplit`, construct a `ParquetRecordBatchStreamBuilder` with the correct projection mask, row filter (from converted predicates), and row group selection. Apply delete files from iceberg-rust's delete file pipeline.

4. **Arrow-to-Trino Page serializer** -- Convert output `RecordBatch` to Trino's wire format. This is not Iceberg-specific and belongs in `trw-protocol`.

### 4.3 Version Pinning Strategy

- Pin `iceberg` to 0.9.x. Monitor the 0.10.0 release for API changes.
- Pin `parquet` (arrow-rs) to the latest stable release. arrow-rs follows a faster release cadence (~monthly) with strong backward compatibility.
- Monitor `iceberg-rust` issues #1469 (incremental scan) and any V3 deletion vector work for future relevance.

### 4.4 Risk Mitigation

| Risk | Mitigation |
|------|------------|
| iceberg-rust pre-1.0 API instability | Wrap all iceberg-rust calls behind our own `ConnectorSpi` trait boundary. If APIs change, only the adapter layer needs updating. |
| Delete file handling edge cases | Invest in comprehensive integration tests using Iceberg's `TestTables` or generating test data with the Java SDK. The 0.8.0 mixed-delete-type bug shows this area needs careful testing. |
| Performance of Rust MOR vs. Java MOR | Benchmark early. The Rust path should be faster (no GC, zero-copy Arrow), but delete file application adds overhead that needs profiling. |
| Missing V3 DV support | V3 is experimental in Trino too. Acceptable to defer. Track iceberg-rust's V3 roadmap. |
| Schema evolution edge cases | Test with files written by different engines (Spark, Hive, Trino) under various schema versions. The positional-mapping fallback (0.8.0) for Hive-migrated files is a good sign. |

### 4.5 Overall Verdict

**The Rust Iceberg/Parquet ecosystem is ready for building a production-quality read path.** The combination of `iceberg-rust` 0.9.0 and `arrow-rs/parquet` covers all critical read-side features: scan planning, predicate pushdown (multi-level), column projection, delete file handling (positional + equality), and schema evolution. The primary custom work is at the integration boundary: parsing Trino's split format, converting Trino's predicate representation, and serializing output to Trino's wire format. These are well-scoped, connector-layer tasks that do not require modifying the upstream libraries.
