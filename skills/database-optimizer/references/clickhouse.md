# ClickHouse Performance Reference

Read this before giving ClickHouse-specific advice. ClickHouse performance is dominated by two design decisions made at table-creation time — the `ORDER BY` key and the partitioning scheme — far more than by query-time tuning. Check those first on any investigation.

## Table engine selection (the MergeTree family)

- **MergeTree** — the base engine: an append-mostly, sorted-on-disk store. The right default for raw event/fact data.
- **ReplacingMergeTree** — deduplicates rows with the same sorting key during background merges, keeping the row with the highest value of an optional version column. Deduplication is *eventual* (it happens when merges run, not immediately), so it's a fit for "latest state of an entity" tables where occasional stale reads between merges are acceptable, not a fit for anything that needs true real-time uniqueness guarantees.
- **AggregatingMergeTree** — stores partial aggregation states (via `-State` combinator functions) that get merged together over time. Typically paired with a materialized view that writes into it.
- **SummingMergeTree** — automatically sums numeric columns for rows sharing the same sorting key during merges. A narrower, simpler alternative to `AggregatingMergeTree` when the only aggregation needed is a sum.

Pick based on what the table actually represents — a raw immutable event log, a deduplicated current-state table, or a pre-aggregated rollup — rather than defaulting to plain `MergeTree` for everything and trying to fix the resulting query cost later.

## ORDER BY and partition key design

- The `ORDER BY` key defines ClickHouse's sparse primary index — data is physically sorted on disk by this key, and the index stores one entry per granule (a block of rows, 8192 by default), not per row. This means lookups on the leading column(s) of the `ORDER BY` key are cheap, and lookups on columns *not* in it (or far down the key) require scanning much more data. Column order in `ORDER BY` should match actual filter patterns — most commonly filtered/most selective columns first — not the order they happen to appear in the source schema.
- `PARTITION BY` (typically a date or time truncation) enables two distinct benefits: partition pruning (skipping whole partitions that can't match a query's `WHERE` clause) and cheap data lifecycle management (`ALTER TABLE ... DROP PARTITION` instead of a slow row-by-row delete for retention policies). Too fine a partition granularity backfires — many small partitions mean many small parts, which increases merge overhead and can hurt query performance rather than help it; check `system.parts` for partition/part counts before recommending a finer-grained scheme.

## Materialized views

ClickHouse materialized views are incremental and trigger-based: they run automatically on every insert into the source table and write their result into a target table, rather than being recomputed on read like a view in most other systems. The common pattern is a materialized view paired with an `AggregatingMergeTree` target table, so that dashboard or rollup queries hit small pre-aggregated data instead of scanning raw events at query time. Recommending this is a meaningful schema/pipeline addition — classify as **Moderate Risk** (it changes the write path, adding work to every insert) and confirm the read pattern is frequent/expensive enough relative to the added write cost to justify it.

## Compression

- Default codec is `LZ4` — fast, modest compression ratio. `ZSTD` gives a better ratio at more CPU cost per read/write; worth it for columns read less often or where storage cost dominates.
- `Delta`/`DoubleDelta` codecs suit monotonic or slowly-changing numeric columns (timestamps, auto-incrementing IDs) — they store differences between consecutive values instead of the raw values.
- `Gorilla` suits floating-point time-series data specifically.
- `LowCardinality(String)` wraps a string column with dictionary encoding when the column has a small number of distinct values relative to row count (e.g. a status enum, a country code) — this cuts both storage size and the cost of filtering/grouping on that column, since comparisons happen on the small integer dictionary codes rather than the full strings.

Don't recommend a codec change without checking the actual cardinality/distribution of the column first — a codec mismatched to the data's actual shape (e.g. `LowCardinality` on a column that's actually high-cardinality) can make things worse.

## Query profiling

- `EXPLAIN PLAN` — shows the logical query plan (steps, not execution detail).
- `EXPLAIN PIPELINE` — shows the actual physical execution pipeline, including parallelism — more useful than `EXPLAIN PLAN` for understanding real execution shape.
- `EXPLAIN ESTIMATE` — estimates how many rows/marks/parts a query would need to read, useful for sanity-checking partition pruning and primary key effectiveness before running the query at all.
- `system.query_log` — historical record of executed queries with `query_duration_ms`, `read_rows`, `read_bytes`, and `memory_usage`. This is the equivalent of `pg_stat_statements`/MySQL's slow log for finding the highest-cost queries by actual measured cost rather than by which one someone happened to notice.
- `system.parts` — part and partition counts/sizes per table; a high part count per partition is a sign that inserts are too frequent/small (each insert typically creates a new part) and would benefit from batching inserts, or that merge settings need attention.

## Skip indexes

Secondary "data skipping" indexes (`minmax`, `set`, `bloom_filter`) can speed up filtering on columns outside the primary key, but they're a secondary tool — check whether the `ORDER BY` key itself could be reshaped to cover the access pattern before reaching for a skip index, since a well-chosen primary key benefits every query, while a skip index only helps the specific column it targets.

- `minmax` — good for columns that correlate with the table's physical order (similar use case to Postgres's BRIN).
- `set` — good for low-cardinality columns, similar to a small lookup table per granule.
- `bloom_filter` — good for equality lookups on higher-cardinality columns where `minmax`/`set` wouldn't help.

## Pagination

`LIMIT ... OFFSET N` in ClickHouse still has to produce and discard the first `N` rows of the sorted result before returning a page — the same cost-grows-with-depth shape as any other engine, though it's a less common complaint here since ClickHouse workloads skew toward analytical scans/aggregations rather than paged lookups. Where deep paging over a large result genuinely is the access pattern, the same keyset approach applies: carry the last row's `ORDER BY` key values forward as a `WHERE` bound instead of increasing `OFFSET`, so each page reads only the rows it needs rather than re-reading and discarding everything before it.

## Sharding (Distributed tables)

The `Distributed` table engine fans queries out across shards defined in the cluster configuration. Shard key choice affects query distribution the same way it does in other sharded systems — a key that doesn't spread data and query load evenly across shards undermines the benefit of sharding at all. Changing a cluster's sharding scheme after data already exists is a significant data-movement operation — classify as **High Risk**.

## Version assumptions

Feature availability (specific skip index types, `EXPLAIN ESTIMATE`, certain `MergeTree` settings) has changed across ClickHouse versions at a faster pace than most engines in this reference set. Confirm the actual version (`SELECT version();`) before assuming a specific feature or default is present, especially when comparing self-hosted ClickHouse against ClickHouse Cloud, which can trail or diverge from open-source release behavior.
