# MySQL Performance Reference

Read this before giving MySQL-specific advice. Assume InnoDB unless told otherwise — it's been the default engine since MySQL 5.5, and most of what follows is InnoDB-specific.

## Reading EXPLAIN / EXPLAIN ANALYZE

`EXPLAIN` shows the planner's chosen access path without running the query; `EXPLAIN ANALYZE` (8.0.18+) actually executes it and adds real timing — prefer the latter when available, the same way `EXPLAIN (ANALYZE)` is preferred in Postgres.

The `type` column is the single most important field — it tells you the join/access method, ranked roughly best to worst:

| type | Meaning |
|---|---|
| `system` / `const` | At most one matching row — best case, usually a primary key or unique lookup on a constant. |
| `eq_ref` | Unique index lookup per row of the joined table — excellent for joins. |
| `ref` | Non-unique index lookup. Good, but can return multiple rows. |
| `range` | Index range scan (`BETWEEN`, `>`, `IN`). Reasonable. |
| `index` | Full index scan — better than a full table scan, but still reads every entry. |
| `ALL` | Full table scan. The one to flag first on any table beyond a few thousand rows. |

Other fields worth reading every time: `key` (which index was actually used — `NULL` means none), `key_len` (how much of a composite index was actually used — a short `key_len` relative to the full index width means only a prefix is being used), `rows` (estimated rows examined), `filtered` (estimated percentage of those rows that survive the `WHERE` clause after the index lookup — low `filtered` with high `rows` means the index isn't doing much filtering work).

`Extra` column danger signs:
- `Using filesort` — the result needs an in-memory or on-disk sort that no index satisfies. An index matching the `ORDER BY` can eliminate this.
- `Using temporary` — a temporary table was needed, often from `GROUP BY`/`DISTINCT` without a supporting index. Frequently paired with `Using filesort`.
- `Using index` (no "condition") — best case, the query was satisfied entirely from the index without touching the table (a covering index).
- `Using index condition` — index condition pushdown; decent, but not as good as a fully covering index.

## InnoDB tuning

- `innodb_buffer_pool_size` is the single highest-leverage setting — it's InnoDB's cache for data and indexes. A common starting guideline is 70–80% of available RAM on a dedicated database host, but always verify against what else is running on that host (the OS, other services, connection overhead) before recommending a specific value — copying a percentage without checking actual headroom is exactly the kind of unverified recommendation this skill exists to avoid.
- `innodb_log_file_size` (or `innodb_redo_log_capacity` in 8.0.30+) — too small causes frequent checkpointing, which shows up as write stalls under heavy write load. Size it relative to write volume, not as a fixed default.
- `innodb_flush_log_at_trx_commit` — `1` (default) flushes and syncs on every commit, the durable choice; `2` or `0` trade some durability for throughput. Changing this away from `1` affects what can be lost in a crash — flag as **Moderate Risk** at minimum and make sure the user understands the durability tradeoff explicitly, don't just present it as a performance win.

## The query cache

MySQL's query cache was deprecated in 5.7 and removed entirely in 8.0 — if a user asks about tuning it on a modern version, say so rather than giving tuning advice for a feature that no longer exists. On versions where it does exist (5.6 and earlier), it was known to scale poorly under write-heavy workloads due to invalidation overhead, so its removal generally wasn't a regression.

## Indexing

Same leftmost-prefix logic as most B-tree index implementations: a composite index on `(a, b, c)` can serve lookups on `a`, `(a, b)`, or `(a, b, c)`, but not `b` or `c` alone. Order columns by what the actual queries filter on — equality columns generally before range columns, matching the query shape rather than a generic rule of thumb. A covering index (all columns the query needs are in the index) shows up as `Using index` in `Extra` and avoids a row lookup entirely.

## Replication

Binlog format matters for what gets replicated and how: `ROW` (replicates the actual changed rows, safest for correctness), `STATEMENT` (replicates the SQL statement, smaller but can diverge for non-deterministic statements), `MIXED` (switches automatically). GTID-based replication (`gtid_mode=ON`) simplifies failover and is the modern default recommendation over file-position-based replication.

Check replication lag via `SHOW REPLICA STATUS` (`Seconds_Behind_Source` / the older `Seconds_Behind_Master`) or `performance_schema.replication_applier_status_by_worker` for more granular per-worker detail under multi-threaded replication. As with Postgres, check for a specific large transaction or DDL before assuming the replica is simply under-provisioned.

Automatic failover (via Orchestrator, MySQL Group Replication, or a managed provider) carries the same quorum/split-brain/RPO-RTO considerations as any HA setup — a promoted replica should be verified to have applied all transactions a client might have already read from the old primary (check that GTID sets match) before traffic is routed to it, and the actual data-loss and downtime tolerance should be stated explicitly rather than assumed to be zero just because failover is automated.

## Pagination

`LIMIT ... OFFSET N` has the same cost profile as Postgres's `OFFSET`: MySQL still produces and discards the first `N` rows before returning a page, so cost grows with page depth regardless of how well-indexed the `WHERE`/`ORDER BY` is otherwise. Prefer keyset pagination — carry the last row's sort-key value forward as a `WHERE` bound (`WHERE id > $last_id ORDER BY id LIMIT 20`, or the equivalent on whatever column defines page order) instead of increasing `OFFSET` — backed by an index on that column. This trades away arbitrary page-number jumps for pages that cost the same regardless of depth.

## Partitioning

`RANGE`, `LIST`, `HASH`, and `KEY` partitioning are available; partition pruning works when the query's `WHERE` clause includes the partitioning column. Watch for version-specific restrictions before recommending a scheme — for example, unique keys (including the primary key) on a partitioned table must include all columns used in the partitioning expression, which can force schema changes beyond just adding partitioning.

## Finding what's actually slow

Enable the slow query log (`slow_query_log`, `long_query_time`) to capture queries over a threshold, and use `performance_schema` (or the `sys` schema views built on top of it, e.g. `sys.statements_with_runtime_in_95th_percentile`) to find the highest aggregate-cost queries rather than relying on which one a user happened to notice.

## Version assumptions

Several specifics above are version-gated — `EXPLAIN ANALYZE` (8.0.18+), the query cache's removal (8.0, deprecated in 5.7), `innodb_redo_log_capacity` replacing `innodb_log_file_size` sizing (8.0.30+). Confirm the actual version (`SELECT VERSION();`) before relying on any of these, since MySQL, MariaDB, and managed forks (Aurora, PlanetScale/Vitess) diverge from stock version numbering and feature availability in ways worth checking rather than assuming.
