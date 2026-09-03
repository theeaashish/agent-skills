# MongoDB Performance Reference

Read this before giving MongoDB-specific advice.

## explain()

Three verbosity modes:
- `explain("queryPlanner")` (default) — shows the chosen plan without running anything. Use when you just need to confirm which plan *would* be picked.
- `explain("executionStats")` — actually runs the query and reports real counters. This is the one to ask for when diagnosing a slow query; `queryPlanner` alone isn't evidence of actual performance.
- `explain("allPlansExecution")` — also shows the plans the optimizer considered and rejected, useful when a query is using an unexpected index and you need to see why the "obvious" one lost.

Key fields in the output:
- `winningPlan.stage` — `COLLSCAN` means a full collection scan (the equivalent of Postgres's `Seq Scan` — flag on any collection beyond a few thousand documents with a selective filter); `IXSCAN` means an index was used; `FETCH` means documents were retrieved after an index scan (normal, but worth checking how much filtering happens at the `FETCH` stage versus the `IXSCAN` stage).
- `executionStats.totalDocsExamined` vs. `executionStats.nReturned` — a large ratio here means the index (or scan) is examining far more documents than it returns, i.e. it's not selective enough for this query. This is the MongoDB equivalent of a Postgres plan's estimated-vs-actual row mismatch.
- `executionStats.totalKeysExamined` — how many index entries were scanned; compare against `totalDocsExamined` to see whether the index itself is doing the filtering or just narrowing the field before a heavier document-level scan.

## Index usage statistics and collection stats

- `db.collection.aggregate([{ $indexStats: {} }])` shows how often each index is actually used (`accesses.ops`) — this is how you find indexes that exist but are never used, which are pure write-amplification cost with no read benefit. Don't recommend dropping one without this data backing it up.
- `db.collection.stats()` / `db.stats()` — document count, average document size, total/index size on disk. Useful for sanity-checking whether a collection is actually large enough for a scan to matter, and for catching a mismatch between expected and actual average document size (which can indicate unexpected field bloat).

## Compound index field order — the ESR rule

Order compound index fields as: **Equality** fields first, then **Sort** fields, then **Range** fields. This isn't an arbitrary convention — it follows from how MongoDB walks a B-tree index: equality fields narrow to a single contiguous range immediately, sort fields let the index return results pre-ordered without a separate sort step, and range fields (which themselves span a range in the index) are most efficient last, after the index has already been narrowed as much as possible by the equality and sort fields.

Example: a query that filters `status: "active"`, sorts by `createdAt`, and ranges on `score: { $gt: 50 }` wants an index on `{ status: 1, createdAt: 1, score: 1 }`, not the field order the application code happens to read most naturally.

## Aggregation pipeline optimization

- Put `$match` and `$sort` as early as possible in the pipeline, ideally as the very first stage — MongoDB can use an index for an early `$match`/`$sort`, but loses that ability once the pipeline passes through a stage like `$unwind` or `$group` that reshapes the documents.
- Use `$project` (or `$unset`) early to drop fields that aren't needed downstream — this reduces the amount of data moving through every subsequent stage, including across a `$lookup` or a network hop in a sharded cluster.
- `$group` and `$sort` stages have a default 100MB memory limit per stage; `allowDiskUse: true` lets them spill to disk instead of erroring, but a pipeline that needs this routinely is worth checking — it might mean an earlier `$match` could shrink the working set instead.
- `$lookup` is effectively a join and carries similar costs — for very hot code paths, consider whether the data should be denormalized (embedded) instead of joined at query time, rather than just trying to make the `$lookup` itself faster.

## Pagination

`skip(N)` has the same cost shape as SQL's `OFFSET` — MongoDB still walks and discards the first `N` documents before returning a page, so a deep `skip()` gets linearly more expensive with depth, and this is frequently the actual cause behind "the API gets slower the further a user pages." Prefer range-based (keyset) pagination: carry the last-seen document's sort-field value forward as a query bound (e.g. `{ _id: { $gt: lastSeenId } }` combined with `.sort({ _id: 1 })`, or the equivalent on whatever field defines page order) instead of `skip()`, backed by an index on that field. As with the SQL equivalent, this trades away jumping directly to an arbitrary page number for pages that cost the same regardless of depth.

## Replica sets and failover

A replica set election requires a majority of voting members to agree on a new primary — an even number of data-bearing members (or a network partition that splits votes) can prevent a majority from forming, which is why an odd number of voting members (adding a non-data-bearing arbiter to break ties if needed, rather than a full extra data-bearing replica) is the standard recommendation. Write concern (`w: 1` vs `w: "majority"`) controls the RPO/durability tradeoff directly: `w: 1` acknowledges once the primary alone applies the write (faster, but that write can be lost if the primary fails before replicating it); `w: "majority"` waits for acknowledgment from a majority of the replica set (safer, higher per-write latency). State which one is actually in use, and what it implies about acceptable data loss, before treating replication lag as purely a performance metric.

## Sharding

- The shard key determines both query routing and write distribution — a key that doesn't match the actual query patterns forces broadcast queries to every shard (a "scatter-gather" query), and a key that's monotonically increasing (a default `ObjectId`, a timestamp) concentrates all new writes on a single shard, creating a hot shard even though the cluster has multiple shards available.
- "Jumbo chunks" (chunks too large to migrate, shown in `sh.status()`) are usually a symptom of a shard key with too little cardinality or a skewed value distribution, not a chunk-size configuration issue to tune away.
- Resharding (changing the shard key after the fact) is a major, resource-intensive operation — classify as **High Risk** and make sure the access-pattern evidence actually justifies it before recommending it, since getting the shard key right the first time is far cheaper than correcting it later.

## WiredTiger cache

The default WiredTiger internal cache size is `max(50% of (RAM - 1GB), 256MB)`. When the actual working set (the data and indexes actively being read/written) exceeds this cache, MongoDB has to go to disk far more often, which shows up as elevated latency that doesn't correlate with any single slow query — it's a capacity issue, not a query-shape issue. Check `db.serverStatus().wiredTiger.cache` for eviction and read-from-disk counters before assuming a specific query is the cause of generalized slowness.

## Version assumptions

Default WiredTiger cache sizing, `$indexStats` availability, and aggregation `allowDiskUse` limits assume a reasonably current MongoDB version. Write-concern defaults and sharding behavior have also changed across major versions — confirm the actual server version (`db.version()`) before relying on a specific default rather than assuming it matches an older deployment's behavior.
