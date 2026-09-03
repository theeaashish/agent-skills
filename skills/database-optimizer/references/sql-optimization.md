# SQL Query Optimization Reference

Read this before rewriting a query, or before reaching straight for "add an index" when the actual problem is how the query itself is written. Everything below is engine-general — it applies across PostgreSQL and MySQL directly, and with the caveats noted per item to other SQL engines. The ORM-specific translation of these same patterns (what each ORM's query builder actually generates) is in `references/orm-patterns.md`; this file is about the SQL shape itself, independent of what produced it.

## SELECT * and over-fetching

Selecting every column when only a few are used costs more at every layer: more bytes read off disk or index, more bytes sent over the network, more memory held on the client. It also silently defeats a covering index (`Index Only Scan` in Postgres, `Using index` in MySQL's `Extra` column) that would otherwise satisfy the query without touching the table at all — a covering index only covers the columns actually referenced by name in the query, so `SELECT *` forces a heap/row fetch even when a perfectly good covering index exists. Recommend explicit column lists once the evidence shows which columns are actually consumed downstream, not as a blanket style rule applied without checking.

## COUNT(*) on large tables

An exact `COUNT(*)` on a large table requires scanning at least a full index, because MVCC-style visibility rules mean the count can't be read directly off table metadata the way some other systems allow. If the use case tolerates an approximation — "about how many rows," a UI that shows "1,000+ results" rather than an exact figure — a planner estimate is far cheaper than the real count: `reltuples` in Postgres's `pg_class` (or running plain `EXPLAIN`, without `ANALYZE`, on the `COUNT(*)` query to read the estimated row count without executing it), or `SHOW TABLE STATUS` / `information_schema.tables.TABLE_ROWS` in MySQL (an estimate for InnoDB, not exact). If the exact number is genuinely required — e.g. pagination math that must be precise — say so and accept the scan cost rather than quietly substituting an estimate that produces a wrong exact answer.

## IN vs EXISTS vs JOIN

- `WHERE col IN (subquery)` and `WHERE EXISTS (correlated subquery)` are logically similar but not guaranteed to be planned identically. Modern Postgres and MySQL optimizers frequently rewrite both into the same semi-join plan, but this isn't true across every engine or every version — don't assume it without checking the actual plan. `EXISTS` short-circuits on the first matching row per outer row, which can matter specifically when the subquery itself is expensive per row.
- A `JOIN` (with `DISTINCT` if the join can multiply rows) sometimes outperforms either when the planner can drive from the smaller side — but this introduces the cartesian-product row-inflation risk covered for ORMs in `orm-patterns.md`. Check the actual row count returned against expectations after rewriting to a join, not just the plan's estimated cost.
- Don't rewrite from one form to another based on general reputation ("EXISTS is always faster than IN"). This exact question has been answered differently across Postgres versions as the optimizer improved — check the plan for the specific query, engine, and version in front of you rather than applying a rule of thumb as fact.

## Correlated subqueries

A subquery that references a column from the outer query logically re-executes once per outer row — a hidden N+1-equivalent inside a single SQL statement, rather than across multiple round trips like the ORM version of this problem. Look for a `SubPlan`/`InitPlan` node in a Postgres plan, or a `DEPENDENT SUBQUERY` marker in MySQL's `EXPLAIN` `select_type` column, especially paired with a high row or loop count. A common rewrite: replacing a correlated "get the latest related row per parent" subquery with a window function (`ROW_NUMBER() OVER (PARTITION BY parent_id ORDER BY created_at DESC)`, filtered to rank 1) collapses the per-row re-execution into a single pass over the data.

## DISTINCT, GROUP BY, and HAVING cost

`DISTINCT` and `GROUP BY` typically require a sort or hash step to de-duplicate or group rows — visible as `Using temporary; Using filesort` in MySQL's `Extra` column, or an explicit `Sort` / `HashAggregate` node in a Postgres plan. This is expected, not inherently a problem, but it's a real cost worth an index on the grouped/distinct columns if the plan shows it dominating. `HAVING` filters *after* aggregation runs; a condition that could instead be written as `WHERE` (filtering rows before they're aggregated) should be — it reduces how many rows the aggregation itself has to process, rather than aggregating everything and throwing results away afterward.

## UNION vs UNION ALL

`UNION` de-duplicates its combined result, which requires a sort or hash step across the full combined set on top of running both underlying queries. `UNION ALL` skips de-duplication entirely. If the two queries being combined are already known not to overlap — e.g. partitioned by a mutually exclusive `WHERE` condition, or pulling from tables that can't share rows — `UNION ALL` produces an identical result for meaningfully less work. Check whether the de-duplication is actually load-bearing for the specific queries in front of you before defaulting to `UNION` out of habit.
