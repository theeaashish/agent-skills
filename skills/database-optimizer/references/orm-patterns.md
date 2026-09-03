# ORM Query Patterns Reference

Read this before giving ORM-specific advice. The same two problems — N+1 queries and over-fetching — recur across every ORM below; what differs is the exact syntax for detecting and fixing them. Don't guess at an ORM's API from general familiarity with a different one; check this file, and ideally ask to see the actual generated query (most of these ORMs support query logging) before proposing a fix.

## The general N+1 signature

In evidence (a query log, `pg_stat_statements`, a slow query log), N+1 looks like: one query to fetch a parent set, followed by the *same* query shape repeated once per row of that parent set, differing only in a literal parameter. The repeat count roughly equals the parent result set size. This is the single most common ORM-caused performance issue across every framework below, and it's usually invisible in the application code itself — it only shows up once you look at what actually hit the database.

## Streaming and bulk writes

Two patterns beyond N+1 and column over-fetching worth checking when a large-result or bulk-write path is slow:

- **Streaming large results** — materializing an entire large result set in memory before processing it (the common default) doesn't scale past a certain size; server-side cursor/streaming APIs let the application process rows incrementally instead. Prisma has no native streaming cursor beyond `cursor`-based pagination (see below); Drizzle and raw `pg`/`mysql2` clients can use the underlying driver's cursor/stream support directly; Mongoose exposes `.cursor()` on a query for one-document-at-a-time iteration instead of `.find().exec()` materializing the full array; TypeORM's `QueryBuilder` has `.stream()`; Hibernate has `ScrollableResults` for JDBC-level cursor scrolling; EF Core supports `IAsyncEnumerable` via `AsAsyncEnumerable()`.
- **Bulk update/delete** — mutating rows one at a time in an application-level loop (load → mutate → save, per row) is far more expensive than one set-based statement. Prefer `updateMany`/`deleteMany` (Prisma, Mongoose), `QueryBuilder().update()`/`.delete()` (TypeORM), `Model.update()`/`.destroy()` with a `where` clause (Sequelize), bulk HQL/JPQL `UPDATE`/`DELETE` or a `@Modifying` query (Hibernate), and EF Core 7+'s `ExecuteUpdateAsync()`/`ExecuteDeleteAsync()` (which skip change-tracking and issue one set-based SQL statement instead of loading entities first). The tell in evidence is the same shape as N+1: many single-row `UPDATE`/`DELETE` statements where one set-based statement would do.

## Prisma

- `include` vs. `select`: `include` pulls related records, `select` lets you pick exact fields — defaulting to `include` on everything pulls more data than needed.
- Nested `include` across relations: verify what SQL is actually generated (Prisma's query engine batches many nested-include cases into a small number of queries rather than one-per-row, but the exact batching behavior has changed across versions) — don't assume it's automatically N+1-safe; check the logged query count against the result set size.
- `findMany()` with no `select` returns every column — fine for a small table, wasteful for a wide one on a hot path.
- Deep pagination via `skip`/`take` has an `OFFSET`-equivalent cost that grows with the skip value on the underlying database; for deep pagination on a large table, cursor-based pagination (`cursor` + `take`) avoids re-scanning skipped rows.
- `$queryRaw` is the escape hatch when the generated query is provably worse than hand-written SQL for a specific hot path — reasonable to recommend once the generated query has actually been shown (via Prisma's query logging) to be the bottleneck, not as a first resort.

## Drizzle

- The relational query API (`db.query.table.findMany({ with: {...} })`) versus the lower-level query builder (`db.select()...`) — the relational API is more convenient but check the generated SQL (Drizzle's logger option prints it) the same as any other ORM rather than assuming convenience implies efficiency.
- Drizzle generates SQL closer to what you'd hand-write than some other ORMs, which makes it easier to inspect — always ask for or enable the logger output rather than guessing from the query-builder call alone.
- For a hot, repeated query shape, a prepared statement (`.prepare()`) avoids re-planning the query on every call — worth recommending once a specific query has been confirmed (via evidence) to run frequently enough for that to matter.

## TypeORM

- `eager: true` on a relation fetches it on *every* load of the entity, even when the related data isn't used in that particular call — a common source of silent over-fetching that doesn't show up by reading the call site, only by reading the relation's decorator.
- Lazy relations (the default without `eager: true`) cause N+1 if accessed inside a loop without an explicit `relations: [...]` or `leftJoinAndSelect` to fetch them up front.
- `QueryBuilder` gives explicit control over joins and selected columns for complex queries; `repository.find()` is fine for simple cases but can hide what's actually being generated.
- Multiple `leftJoinAndSelect` calls across one-to-many relations on the same query multiply rows (cartesian product) before TypeORM reassembles them into nested objects — this inflates the actual result set size transferred from the database even though the final object graph looks reasonable; check `EXPLAIN`/row counts against what the application code expects.

## Sequelize

- `include` without `separate: true` on a `hasMany` association can produce a cartesian-product result set (duplicated parent rows, one per child) before Sequelize de-duplicates them client-side — check the actual row count returned by the database, not just the final deduplicated object count.
- Lazy access via `instance.getAssociation()` called inside a loop over a parent result set is the classic N+1 pattern — prefer `include` on the original query so the association is fetched up front.

## Mongoose

- `populate()` issues a separate query by default. Calling it inside a `.forEach()`/loop over documents is N+1; calling it once on the original query (`Model.find().populate('field')`) lets Mongoose batch it into a single additional query using `$in`.
- `.lean()` skips Mongoose document hydration (no change-tracking, no instance methods) for read-only paths — meaningfully cheaper for read-heavy endpoints that don't need to call document methods or save changes back.
- `.select()` projection avoids pulling fields that aren't used downstream, the same over-fetching concern as a SQL `SELECT *`.

## Hibernate

- The classic Hibernate N+1: a `@OneToMany`/`@ManyToOne` with `FetchType.LAZY` (the recommended default), accessed inside a loop after the original query/session has already returned — each access triggers its own `SELECT`.
- Fixes: a `JOIN FETCH` in the HQL/JPQL query to eagerly pull the association in the original query, or `@BatchSize` to batch the lazy loads into grouped `IN (...)` queries instead of one per row.
- "Open Session in View" can mask this by deferring the lazy loads to view-rendering time rather than failing fast — which means the N+1 still happens, just later and less visibly in profiling that only covers the controller/service layer. Check evidence at the database level (the actual query log), not just at the request-handler level, or this pattern can hide from the investigation entirely.

## Entity Framework (EF Core)

- `Include()` / `ThenInclude()` for eager loading of related entities in a single query.
- Lazy-loading proxies, if enabled, reproduce the same N+1 risk as Hibernate's lazy associations if a relation is walked inside a loop.
- `AsNoTracking()` for read-only queries avoids EF's change-tracking overhead — a real win on read-heavy paths that don't need to save changes back.
- Multiple `Include()`/`ThenInclude()` calls across one-to-many relations in a single query can produce the same cartesian-product row inflation seen in TypeORM — `AsSplitQuery()` (EF Core 5+) splits a multi-include query into several smaller queries instead of one large joined one, which avoids the row multiplication at the cost of more round trips. Which is better depends on the actual relation shape and row counts — check evidence rather than defaulting to one over the other.
