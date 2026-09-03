# Example Investigation Reports

Four worked examples showing the evidence-first workflow end to end, including the case that matters most: a request with too little evidence to recommend anything yet. Use these as a calibration reference for level of detail and tone — not as templates to fill in with invented specifics for a real investigation.

---

## Example 1 — PostgreSQL: missing index causing a sequential scan

**User's message:** "Our `/orders` endpoint got slow this week. Here's the query: `SELECT * FROM orders WHERE customer_id = $1 ORDER BY created_at DESC LIMIT 20;` — Postgres 15, table has about 4.2M rows."

**What's missing before recommending anything:** an execution plan and confirmation of existing indexes. Request both:

> Before recommending a fix, I'd want to see:
> - `EXPLAIN (ANALYZE, BUFFERS)` for that exact query
> - `\d orders` (or the `CREATE TABLE` statement) so I can see what indexes already exist
>
> Given the row count and the shape of the query, a missing index on `customer_id` is the likely culprit, but I want to confirm against the actual plan rather than assume.

**User provides:**
```
Seq Scan on orders  (cost=0.00..98234.00 rows=21 width=312) (actual time=412.318..890.104 rows=20 loops=1)
  Filter: (customer_id = 48213)
  Rows Removed by Filter: 4199980
  Buffers: shared hit=1102 read=87234
Planning Time: 0.241 ms
Execution Time: 890.391 ms
```
and confirms the only indexes are the primary key on `id` and a unique index on `email`.

**Report:**

1. **Issue Summary** — The `/orders` endpoint query does a full sequential scan of a 4.2M-row table to return 20 rows for a single customer.
2. **Evidence** — `EXPLAIN (ANALYZE, BUFFERS)` shows `Seq Scan on orders`, removing 4,199,980 of 4,200,000 rows by the filter, at 890ms actual execution time. No index exists on `customer_id`; the only indexes present are on `id` and `email`.
3. **Root Cause** — No index supports the `customer_id` equality filter, so the planner has no access path other than scanning every row.
4. **Recommended Fix** —
   ```sql
   CREATE INDEX CONCURRENTLY idx_orders_customer_id_created_at
     ON orders (customer_id, created_at DESC);
   ```
   The composite key matches the query's filter (`customer_id`) and sort (`created_at DESC`) so the index can satisfy both without a separate sort step.
5. **Risk Assessment** — **Safe**. Built with `CONCURRENTLY` (no exclusive lock on the table during build), purely additive, and no existing query depends on this index's absence.
6. **Expected Impact** — This should replace the sequential scan with an index scan limited to the ~20 rows actually needed for this customer, eliminating the need to read and filter 4.2M rows per call. The exact resulting latency should be confirmed by re-running the query rather than predicted, since it depends on cache state and concurrent load at measurement time.
7. **Validation Steps** — Re-run `EXPLAIN (ANALYZE, BUFFERS)` on the same query; confirm the plan shows `Index Scan` (or `Index Only Scan`) instead of `Seq Scan`. If `pg_stat_statements` is enabled, check this query's `mean_exec_time` before and 24 hours after deploy.
8. **Rollback Plan** — `DROP INDEX CONCURRENTLY idx_orders_customer_id_created_at;` — no data loss, the table returns to its prior state. The only cost of rolling back is losing the performance benefit.

---

## Example 2 — MongoDB: N+1 from Mongoose `populate()` plus a missing compound index

**User's message:** "Our order history page calls `Order.find({ userId }).then(orders => orders.forEach(o => o.populate('items')))` and it's slow with more than ~30 orders."

**What's missing:** the actual query log / `explain()` output and the existing indexes on the `items` lookup. Request both, but the code itself already shows a structural problem worth naming as a hypothesis while waiting:

> The code calls `populate()` inside a `forEach` over the order list — that issues one additional query per order rather than batching, which is consistent with "fine at low volume, slow as the list grows." I'd treat that as a likely contributor, but to confirm and to check whether indexing is also a factor, could you share:
> - the query log (enable Mongoose debug mode or MongoDB's profiler) for one page load
> - `db.items.getIndexes()` and an `explain("executionStats")` for the query `populate('items')` actually runs under the hood

**User provides** a log showing 1 query for `Order.find` followed by 31 separate `Item.find({ orderId: ... })` calls, and an `explain` showing `winningPlan.stage: "COLLSCAN"` on the `items` collection with `totalDocsExamined: 184320` against `nReturned: 6`.

**Report:**

1. **Issue Summary** — Loading an order history page issues one query per order to fetch items (N+1), and the per-item query itself does a full collection scan.
2. **Evidence** — Query log shows 1 `Order.find` query followed by 31 individual `Item.find` calls for a 31-order page. `explain("executionStats")` on the item lookup shows `COLLSCAN` with `totalDocsExamined: 184320` vs. `nReturned: 6`.
3. **Root Cause** — Two compounding issues: (a) `populate()` is called inside a `forEach` instead of on the original `Order.find()` query, producing one query per order instead of one batched query; (b) the `items` collection has no index supporting the `orderId` lookup, so each of those 31 queries also does a full scan.
4. **Recommended Fix** —
   - Batch the populate by calling it once on the original query: `Order.find({ userId }).populate('items')` instead of inside the loop, so Mongoose issues a single `$in`-based query for items across all orders on the page.
   - Add a supporting index: `db.items.createIndex({ orderId: 1 })` (extend to a compound index if items are also commonly filtered/sorted by another field on this path).
5. **Risk Assessment** — Code change: **Safe** (behavior-preserving, just changes where `populate` is called). Index addition: **Safe** (additive; confirm via `$indexStats` after deploy that it's actually being used, and check current write volume on `items` to confirm the new index's write overhead is acceptable — likely negligible at this collection's apparent size, but worth a glance at insert rate before calling it fully cost-free).
6. **Expected Impact** — Reduces the query count for a 31-order page from 32 to 2, and replaces a 184,320-document scan with an indexed lookup. Actual page-load latency improvement should be measured post-deploy rather than estimated.
7. **Validation Steps** — Confirm via query logging that a page load now issues a single batched `items` query. Re-run `explain("executionStats")` on that query and confirm `IXSCAN` replaces `COLLSCAN`, with `totalDocsExamined` close to `nReturned`.
8. **Rollback Plan** — Code change: revert the `populate()` call site (no data risk). Index: `db.items.dropIndex({ orderId: 1 })` — cheap, no data loss, only loses the performance benefit.

---

## Example 3 — Redis: eviction policy mismatched to actual usage

**User's message:** "We're seeing data disappear from Redis that shouldn't — looks like session tokens are getting evicted even though they're supposed to last 24 hours."

**What's missing:** current `maxmemory-policy`, memory headroom, and confirmation of whether the affected keys actually have a TTL set as expected. Request:

> This sounds like an eviction-policy mismatch rather than a TTL bug, but I'd want to confirm with:
> - `CONFIG GET maxmemory` and `CONFIG GET maxmemory-policy`
> - `INFO memory` (specifically `used_memory` vs `maxmemory`)
> - `TTL` on a few of the affected session keys, to confirm whether they actually have the 24h TTL you expect or whether they're being stored without one

**User provides:** `maxmemory-policy: allkeys-lru`, `used_memory` consistently near `maxmemory`, and confirms the session keys do have a correct 24-hour TTL set.

**Report:**

1. **Issue Summary** — Session tokens with a correctly-set 24-hour TTL are being evicted early because the eviction policy treats all keys as evictable, not just expired ones.
2. **Evidence** — `maxmemory-policy` is `allkeys-lru`, which evicts the least-recently-used key regardless of TTL once `maxmemory` is reached. `used_memory` is consistently near `maxmemory`, confirming eviction is actively happening, not just configured. Affected keys have a correct TTL, ruling out an application-side TTL bug.
3. **Root Cause** — `allkeys-lru` is the wrong policy for this usage pattern: it's appropriate when *every* key is disposable cache, but here some keys (sessions, with an intentional TTL) need to survive until their TTL expires, not be evicted early just because they're cold. A `volatile-*` policy only evicts keys that already have a TTL set as a deliberate "ok to drop early under pressure" signal — but that's still not the right fit here, since these sessions are *not* meant to be dropped early either; they're meant to last the full 24 hours.
4. **Recommended Fix** — Two parts, and they should be evaluated together rather than picking one in isolation:
   - Confirm whether anything else sharing this Redis instance genuinely is disposable cache. If so, the cleanest fix is usually separating the workloads: keep cache-shaped, fully-disposable data in one Redis instance/logical database with `allkeys-lru`, and keep session data (which must survive its TTL) in one with `noeviction`, sized with enough `maxmemory` headroom that it never needs to evict.
   - If splitting instances isn't feasible immediately, `noeviction` on the shared instance stops silent data loss immediately, but writes will start erroring once memory fills — which surfaces the actual capacity problem loudly instead of hiding it, but does require the application to handle write errors gracefully and likely means increasing `maxmemory` is also needed.
5. **Risk Assessment** — **High Risk** either way: changing `maxmemory-policy` changes durability behavior for every key on the instance, and switching to `noeviction` without enough memory headroom converts a silent-data-loss problem into a write-failure problem, which still needs to be handled. This needs explicit sign-off, not a quiet config change — flagging that loudly here rather than in a buried risk note.
6. **Expected Impact** — Stops session tokens from being evicted before their TTL. Does not, by itself, solve the underlying memory pressure (`used_memory` near `maxmemory`) — that's a separate sizing question that the workload split or a `maxmemory` increase addresses.
7. **Validation Steps** — After the change, monitor session-key survival directly (e.g. sample a cohort of session keys and confirm none disappear before their TTL) rather than just checking the config value took effect. Watch for write errors if `noeviction` is chosen, and `used_memory` trend if `maxmemory` is also increased.
8. **Rollback Plan** — `CONFIG SET maxmemory-policy allkeys-lru` reverts instantly — but reverting reintroduces the original silent-eviction problem, so frame the rollback path honestly rather than as a free option.

---

## Example 4 — Insufficient evidence: the correct response is to ask, not recommend

**User's message:** "Should we shard our MongoDB? We're growing fast and I want to get ahead of it before it becomes a problem."

**Why this can't be answered yet:** there's no data volume, write rate, query pattern, or current resource utilization in the request — "growing fast" doesn't establish that a real bottleneck exists yet, and sharding is a **High Risk**, hard-to-reverse decision (the shard key is expensive to change later). Recommending a sharding scheme here would be exactly the kind of unsupported, premature-optimization advice this skill is built to avoid.

**Correct response:**

> Sharding is a big, hard-to-reverse decision — the shard key in particular is expensive to change once data and queries depend on it — so I'd want real evidence that you're approaching a limit before designing a scheme, rather than sharding ahead of an actual bottleneck. Could you share:
> - Current `db.stats()` (data size, index size) and roughly how fast that's growing month over month
> - `db.serverStatus()` for current CPU/memory utilization, and whether you're seeing any current slowness or just anticipating future growth
> - Your dominant query patterns — what fields do most reads/writes filter on? This determines whether a workable shard key even exists for your access pattern
> - Whether you're currently on a single replica set, and if so, how close you are to its practical ceiling (WiredTiger cache size vs. working set, disk I/O headroom)
>
> If it turns out you're nowhere near a real ceiling yet, the better near-term investment is usually making sure indexing and query patterns are solid on the single replica set — sharding adds meaningful operational complexity and should solve a bottleneck that's actually been measured, not one that's anticipated. Happy to dig into either direction once I can see where things actually stand.

This is the report format with most sections genuinely empty, and that's the correct shape of the output here — naming what's missing and why it matters is the deliverable, not a placeholder on the way to a recommendation.
