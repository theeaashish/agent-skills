# PostgreSQL Performance Reference

Read this before giving PostgreSQL-specific advice. It covers how to read the evidence Postgres gives you and what each piece of evidence actually implies — use it to ground recommendations, not as a substitute for asking the user for their actual output.

## Reading EXPLAIN (ANALYZE, BUFFERS)

Always ask for `EXPLAIN (ANALYZE, BUFFERS)`, not bare `EXPLAIN`. Bare `EXPLAIN` only shows the planner's *estimate* — without `ANALYZE` it never actually runs the query, so it can't tell you what really happened. `BUFFERS` adds shared/local block hit and read counts, which is how you tell a plan is cache-friendly versus hitting disk.

Key fields per node:
- `cost=startup..total` — planner's internal cost units, useful only for comparing plans against each other, not as a time estimate.
- `rows=N` — planner's *estimated* row count for that node.
- `actual time=X..Y` — wall-clock milliseconds for first row and all rows, per loop iteration.
- `actual rows=N` — the real row count. Compare against the estimate above; a large gap means stale statistics.
- `loops=N` — how many times this node executed (relevant under nested loops — actual time is *per loop*, so multiply for the real total contribution).

Danger signs, in roughly descending order of "ask about this first":
- **`Seq Scan`** on a table with more than a few thousand rows, where the filter is selective (returns a small fraction of rows). Not inherently bad — Postgres correctly prefers a seq scan when it expects to return most of the table — but worth checking whether an index could change that.
- **`Nested Loop`** with a high `loops` count multiplied by non-trivial per-loop time — often the result of a misestimated row count one level up, causing the planner to choose a nested loop where a hash or merge join would have been cheaper at the actual scale.
- **Sort with `Sort Method: external merge  Disk`** — the sort spilled to disk because `work_mem` was too small for the data being sorted. `Sort Method: quicksort  Memory` is the in-memory, healthy version.
- **`Hash Join` with `Batches: N` where N > 1`** — the hash table spilled to disk, again usually a `work_mem` sizing issue.
- **Large gap between estimated and actual rows** anywhere in the plan — points at stale statistics (run `ANALYZE` on the table) rather than a structural problem; don't recommend an index fix on top of a stats problem without ruling this out first.

### Other plan nodes worth recognizing

These show up often enough in real plans to name explicitly rather than treating them as a mystery node the first time they appear — none are inherently a danger sign the way the ones above are:

- **`Bitmap Index Scan` → `Bitmap Heap Scan`** — Postgres's answer to moderately selective predicates, and how it combines *multiple* indexes for an `AND`/`OR` across columns that don't share one composite index: it builds an in-memory bitmap of matching row locations, then fetches heap pages in physical order. Check `Heap Blocks: exact` vs `lossy` in the `Bitmap Heap Scan` line — `lossy` means `work_mem` was too small to hold the exact bitmap and it fell back to page-level matching, which does more heap-level rechecking than necessary.
- **`Merge Join`** — joins two inputs that are already sorted (or explicitly sorted as part of the plan) by walking both in order together. Good when both sides are large and already sorted, e.g. by an index; worse than a hash join when one side needs an extra sort step added just to enable the merge.
- **`Index Only Scan`** — the payoff of a covering index (see `INCLUDE` above): satisfied entirely from the index, no heap fetch. Watch the `Heap Fetches` counter in `ANALYZE` output — nonzero means visibility-map bits weren't set for those pages (usually because they were recently modified and not yet vacuumed), so the "index only" scan still had to check the heap for some rows anyway.
- **`Gather` / `Gather Merge`** (parallel query) — a plan run by multiple worker processes and gathered back into one stream (`Gather Merge` when the workers' output has to stay ordered). Controlled by `max_parallel_workers_per_gather` and a cost threshold below which Postgres won't bother parallelizing. If a large-table plan looks like it should parallelize but doesn't, check that threshold and `max_worker_processes` availability before assuming the query shape itself is the blocker.

## pg_stat_statements

This is how you find out *which* query is actually worth optimizing, instead of optimizing whichever one the user happened to notice. Requires `pg_stat_statements` in `shared_preload_libraries` (a server restart to enable if not already on).

Useful columns: `calls`, `total_exec_time`, `mean_exec_time`, `rows`, `shared_blks_hit` / `shared_blks_read` (cache hit ratio for that specific query). Sort by `total_exec_time` to find what's costing the most aggregate time — a query that's individually fast but called 10,000×/minute can dominate total load more than a single slow report query.

Since PG16, `pg_stat_io` complements this with I/O broken out by backend type and context (e.g. how much of a table's I/O is normal reads versus autovacuum versus bulk writes) — useful when `pg_stat_statements` shows a query is expensive but doesn't explain whether that cost is cache misses or genuine CPU/row-processing work.

## Index selection

- **B-tree** (default) — equality and range predicates, sorting. The right default for almost everything unless a specific reason points elsewhere.
- **BRIN** — large tables where the indexed column correlates with physical row order (e.g. an append-only `created_at` on a time-series table). Much smaller and cheaper to maintain than a B-tree, at the cost of being a coarser (block-range) filter rather than a precise lookup. Good fit when the table is too big for a B-tree to be cheap and the data is naturally ordered; bad fit if the column's values are scattered randomly through the table.
- **GIN** — full-text search (`tsvector`), `jsonb` containment (`@>`), array containment. The standard choice when the query uses `@>`, `?`, `?|`, `?&`, or full-text `@@`.
- **GiST** — geometric data, range types, exclusion constraints, and an alternative for full-text/trigram search when GIN's build/update cost is a concern (GiST builds faster, GIN looks up faster — pick based on whether the workload is read- or write-heavy on that column).
- **Partial indexes** (`WHERE` clause on the index itself) — when queries consistently filter to a subset of rows (e.g. `WHERE status = 'active'`), a partial index on just that subset is smaller and faster to maintain than indexing the whole table.
- **Covering indexes** (`INCLUDE`) — add columns that are only ever read, never filtered on, so the planner can satisfy the query straight from the index (`Index Only Scan`) without a heap fetch.
- **Composite index column order** — put equality-filtered columns before range-filtered/sort columns, matching the actual query's `WHERE`/`ORDER BY` shape; a composite index's leftmost columns are the ones that must appear in equality predicates for the index to be usable for that prefix.
- **GIN operator classes** — the default `jsonb_ops` indexes every key and value, supporting `@>`, `?`, `?|`, and `?&`; `jsonb_path_ops` only supports `@>` but produces a smaller, faster index for that one operator. Worth the switch when containment (`@>`) is the only operator actually used against a given `jsonb` column — check the query patterns before assuming the broader default is needed.
- **Expression indexes** — index the result of an expression (e.g. `CREATE INDEX ON users (lower(email))`) when queries filter on a computed form of a column rather than the raw column. The query has to use the identical expression for the planner to match it to the index — `WHERE lower(email) = $1` matches, `WHERE email ILIKE $1` does not.

## VACUUM, autovacuum, and bloat

Postgres's MVCC model means `UPDATE`/`DELETE` don't immediately reclaim space — they leave dead tuples behind for `VACUUM` to clean up. Left unmanaged, this is "bloat": tables and indexes larger on disk than their live data would require, with knock-on effects on cache efficiency and scan cost.

- Check `pg_stat_user_tables.n_dead_tup` relative to `n_live_tup` — a high ratio signals vacuum isn't keeping up.
- `autovacuum_vacuum_scale_factor` (default 0.2 — vacuum after 20% of the table changes) and `autovacuum_vacuum_threshold` control trigger frequency; high-churn tables often need a lower scale factor set per-table (`ALTER TABLE ... SET (autovacuum_vacuum_scale_factor = 0.05)`).
- A long-running transaction blocks vacuum from cleaning up rows newer than that transaction's snapshot — check `pg_stat_activity` for old transactions before assuming a tuning problem.
- `VACUUM FULL` reclaims disk space by rewriting the table, but takes an `ACCESS EXCLUSIVE` lock for the duration — classify as **High Risk** on any table that's actively queried, and prefer an online alternative (`pg_repack`) if the table can't tolerate a maintenance window.

## Partitioning

Range, list, or hash partitioning lets the planner skip whole partitions that can't match a query (`partition pruning`) and makes dropping old data cheap (`DROP TABLE` on an old partition instead of a slow bulk `DELETE`). Good fit for very large, naturally-segmented tables — time-series data is the classic case.

Retrofitting partitioning onto an existing large live table is invasive (effectively a full data rewrite) — classify this as **High Risk** and make sure the user actually has the access pattern (e.g. regularly querying or dropping by the partition key) that justifies the migration cost, not just "partitioning sounds like it would help."

## Pagination

`OFFSET N` doesn't skip rows for free — Postgres still has to produce and discard the first `N` rows before returning a page, so cost grows roughly linearly with how deep into the result set the page is. This is one of the most common real-world scaling complaints ("page 1 is fast, page 500 times out") and it's a query-shape issue, not something an index on the filter alone fixes.

Prefer keyset (cursor) pagination: instead of `ORDER BY created_at OFFSET 10000 LIMIT 20`, carry the last-seen row's sort-key value forward and filter with it instead — `WHERE created_at < $last_seen_created_at ORDER BY created_at DESC LIMIT 20` — backed by an index on `created_at` (or the composite sort key, if paginating within a filtered subset). This makes every page roughly constant-cost regardless of depth, at the cost of losing the ability to jump straight to an arbitrary page number rather than paging forward/backward from a cursor.

If the evidence is `EXPLAIN` output showing a large `OFFSET` with rows scanned far exceeding rows returned, that gap is the signature to point at directly — the same way a `Seq Scan` cost gap points at a missing index.

## Concurrency and locking

- **Isolation levels** — `Read Committed` is the Postgres default: each statement within a transaction sees a fresh snapshot, so two statements in the same transaction can see different committed data. `Repeatable Read` freezes the snapshot at transaction start. `Serializable` adds runtime conflict detection on top of that and can abort a transaction with a serialization failure the application must be prepared to retry. Don't recommend raising the isolation level as a performance fix — it's a correctness/anomaly tradeoff, not a speed one — and treat lowering it with the same explicit tradeoff framing as any other durability-adjacent change.
- **Deadlocks** — Postgres detects and breaks deadlocks automatically by aborting one of the involved transactions, but recurring deadlocks are a modeling problem, not a config problem. Check `pg_locks` joined against `pg_stat_activity` for what was actually held and waited on, and enable `log_lock_waits` (with `deadlock_timeout` tuned appropriately) to capture recurring cases instead of relying on the application-level error alone. The usual fix is consistent lock-acquisition order across transactions, not a database setting.
- **Advisory locks** (`pg_advisory_lock` / `pg_try_advisory_lock`) — application-level locks Postgres tracks without tying them to any table row; useful for coordinating across application instances (e.g. "only one worker should run this job at a time") without modeling a row to lock. They're tied to the session, not the transaction, so they don't survive connection pooling in transaction mode cleanly (see Connection pooling below).
- **`SELECT ... FOR UPDATE` / `FOR SHARE`** — row-level locking within a transaction. `FOR UPDATE` takes an exclusive lock (blocks other `FOR UPDATE`s and writes on those rows); `FOR SHARE` takes a shared lock (blocks writes, allows other readers). `FOR UPDATE SKIP LOCKED` is the standard pattern for a job-queue-style "grab the next available row" query without blocking on rows another worker already claimed — reach for this before hand-rolling a polling/retry loop around lock contention.

## Prepared statements and plan caching

Server-side `PREPARE`/`EXECUTE` (or whatever a driver/ORM does under the hood for prepared statements) skips re-parsing and re-planning a query shape on every call. Postgres's own behavior here has a specific gotcha worth knowing: for the first 5 executions of a prepared statement, it plans against the actual parameter values each time (a "custom plan"); from the 6th execution on, it compares the average cost of those custom plans against one "generic plan" (planned without knowing the parameter values) and switches to the generic plan if it looks cheaper on average. If a column's data is skewed — some parameter values match far more rows than others — the generic plan chosen based on an average can be badly wrong for a specific unlucky value. This shows up as "the query is fast normally but occasionally very slow with no plan change visible," because the plan itself didn't change, just which parameter value it's now running against. `plan_cache_mode` (`auto` / `force_custom_plan` / `force_generic_plan`) lets you force custom planning per-session once `pg_stat_statements` confirms high variance for one normalized query across different parameter values.

## Bulk loading

Row-by-row `INSERT` in a loop is the slowest way to load data — each statement round-trips and commits (or participates in a transaction) independently. In order of typical leverage: wrap multiple inserts in one explicit transaction to avoid a commit-per-row; use multi-row `INSERT ... VALUES (...), (...), (...)` to cut round trips; use `COPY` (or a driver's binary-copy protocol, e.g. `pg-copy-streams` in Node, `psycopg`'s `copy_expert`) for genuinely bulk loads — it's substantially faster than any `INSERT` variant since it bypasses per-row planning and parses a stream format instead. If foreign keys or triggers are slowing a bulk load, disabling them for the load and re-validating afterward is an option worth naming — but classify it as **Moderate Risk**, since it changes what gets enforced during the load window, not as a free performance win.

## CTEs and window functions

Before Postgres 12, a CTE (`WITH ... AS (...)`) was always materialized — computed once into a temporary result before the outer query ran, acting as an optimization fence the planner couldn't push outer filters through. Since Postgres 12, the planner can inline a CTE like a subquery unless it's referenced more than once or explicitly marked `MATERIALIZED`. If a query with CTEs behaves unexpectedly on an older Postgres version, or has `MATERIALIZED` forcing the old behavior on a newer one, that inlining difference — not a missing index — is often the actual cause; check the Postgres version before diagnosing a CTE-heavy query as an indexing problem.

Window functions (`ROW_NUMBER()`, `RANK()`, `LAG`/`LEAD`, etc.) require the input sorted per their `PARTITION BY`/`ORDER BY` clause, so an index matching that clause can let the planner avoid an explicit sort step — check for a `WindowAgg` node paired with an unindexed sort in the plan before assuming the window function itself is the cost.

## Replication

Streaming replication lag is visible in `pg_stat_replication` on the primary. Common causes, roughly in order of likelihood: a single large transaction or DDL statement generating a burst of WAL the replica can't apply fast enough; network bandwidth between primary and replica; the replica being under-provisioned relative to the primary's write rate; or too many replicas competing for the primary's WAL-sending capacity. Don't default to "add more replica resources" without first checking whether the lag correlates with a specific large operation.

Automatic failover (via a tool like Patroni, or a managed provider's built-in mechanism) adds its own failure modes worth naming when relevant: a quorum-based setup needs a majority of nodes to agree before promoting a new primary (protecting against split-brain, where two nodes both believe they're primary), and any failover has a nonzero RPO (data since the last replicated transaction, lost if the old primary doesn't come back) and RTO (time to detect failure and complete promotion) — both should be sized against the business's actual tolerance for data loss and downtime, not assumed to be zero just because failover is automated.

## Configuration

Before recommending a value for `shared_buffers`, `effective_cache_size`, `work_mem`, `maintenance_work_mem`, or `max_connections`, get the actual available RAM and current connection count — these settings are sized relative to hardware, and a value copied from a blog post tuned for a different instance size can do more harm than the default. `work_mem` in particular is allocated *per sort/hash operation, per connection* — a value that looks reasonable can multiply unexpectedly under high concurrency.

## Connection pooling

High-connection-count applications (e.g. many short-lived serverless/lambda connections) often benefit more from a pooler like PgBouncer than from raising `max_connections`, since each Postgres connection carries real backend process overhead. Transaction-mode pooling gives the highest connection multiplexing but breaks session-level features (e.g. session-level `SET`, prepared statements in some configurations, advisory locks held across statements) — check whether the application relies on any of those before recommending transaction mode over session mode.

As a starting-point sizing heuristic, not a substitute for measuring: pool size ≈ `(core_count × 2) + effective_spindle_count` for the database-side connection count, per PgBouncer's own sizing guidance — most workloads saturate throughput well before every core has its own connection, since a connection spends much of its time waiting on I/O rather than using CPU. Pool exhaustion shows up as requests queuing at the pooler — visible in PgBouncer's `SHOW POOLS`, specifically a nonzero `cl_waiting` — rather than as a slow individual query in `pg_stat_statements`. If the evidence is generalized request slowness with no single query standing out, check the pool's waiting-client count before assuming a query problem.

## Version assumptions

This file's claims about defaults and availability (e.g. `pg_stat_io` at PG16+, CTE inlining behavior at PG12+, `MERGE` at PG15+) are current as of major versions through PG16/17. Confirm the actual server version before relying on version-gated behavior — `SELECT version();` — since recommending a feature on a server that predates it is exactly the kind of unverified claim this skill exists to avoid.
