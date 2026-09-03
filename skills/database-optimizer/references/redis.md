# Redis Performance Reference

Read this before giving Redis-specific advice. Redis problems are usually one of three shapes: memory pressure, latency spikes, or a mismatch between the configured persistence/eviction strategy and how the application actually uses Redis (pure cache vs. system of record). Identify which shape you're looking at before recommending a fix.

## Memory analysis

- `INFO memory` — `used_memory` (Redis's own accounting of memory in use) vs. `used_memory_rss` (actual OS-level resident memory) vs. `mem_fragmentation_ratio` (`used_memory_rss` / `used_memory`). A ratio well above 1.0 indicates fragmentation — memory the OS has allocated to Redis that Redis itself isn't using for data — which `MEMORY PURGE` (Redis 4+, jemalloc) can sometimes reclaim without a restart.
- `MEMORY USAGE <key>` — exact memory footprint of a specific key. Use this to confirm a suspected big key rather than guessing from its apparent size in application code.
- `MEMORY STATS` — a fuller breakdown including overhead from the dataset, replication backlog, client buffers, and Lua scripts — useful when `used_memory` is much higher than the data alone would suggest.
- `MEMORY DOCTOR` — Redis's own built-in diagnosis; treat its output as a starting hint, not a substitute for actually looking at the stats above.

## Big keys

`redis-cli --bigkeys` scans the keyspace and reports the largest key of each type. Large hashes, sets, lists, or sorted sets are a common source of two distinct problems: memory imbalance (one key disproportionately large relative to everything else) and latency spikes (operations on these types are O(N) or worse — `HGETALL`, `SMEMBERS`, `LRANGE` on a huge key blocks the single-threaded command loop for everyone else while it runs). If a latency spike correlates with access to a specific large key, the fix is usually restructuring the data model (e.g. splitting one giant hash into multiple smaller keys, or moving to a structure that supports incremental access) rather than a configuration change.

## Latency analysis

- `SLOWLOG GET` — logs commands that exceeded `slowlog-log-slower-than` (microseconds). The first place to look for "what specifically is slow," as opposed to "Redis feels slow."
- `LATENCY HISTORY <event>` / `LATENCY LATEST` — built-in latency monitoring by event type (e.g. `command`, `fork`, `expire-cycle`), useful for separating "a slow command" from "a slow background operation" like an RDB fork or an expiration cycle.
- `redis-cli --latency` / `--latency-history` — basic round-trip latency sampling, good for confirming whether a perceived slowdown is Redis-side at all versus network/client-side.
- Common root causes beyond a single slow command: a large `KEYS *` or unbounded scan run against a big keyspace in production; an expiration storm (many keys with the same TTL expiring at once); `fork()` time for RDB snapshotting or AOF rewrite on a large dataset (the parent process blocks briefly during fork, and copy-on-write memory growth during the fork can itself create memory pressure).

## Eviction policy

`maxmemory-policy` needs to match how the application actually uses Redis — this is one of the most common sources of subtle production bugs, not just a performance knob:

- `noeviction` — returns errors on writes once `maxmemory` is hit instead of evicting anything. The right choice when Redis holds data that must not be silently lost (a system of record), since failing loudly is better than corrupting application assumptions by silently dropping data.
- `allkeys-lru` / `allkeys-lfu` — evicts any key (LRU: least recently used, LFU: least frequently used) regardless of whether it has a TTL. Appropriate for a pure-cache use case where every key is disposable.
- `volatile-lru` / `volatile-lfu` / `volatile-ttl` — only evicts keys that have a TTL set, leaving keys without a TTL untouched. Appropriate for a mixed workload where some keys are cache (and have a TTL) and others are meant to persist (and don't).

Before recommending a policy change, confirm which of these usage patterns actually describes the application — recommending `allkeys-lru` for a Redis instance that's also being used as a system of record for some untyped keys is the kind of "performance fix" that causes silent data loss. Classify a `maxmemory-policy` change as **Moderate Risk** at minimum, and as **High Risk** if any keys without a TTL are meant to be durable.

## Persistence

- **RDB** — periodic point-in-time snapshots via `fork()`. Cheap on CPU during normal operation, but the fork itself briefly pauses the parent process, and large datasets can cause a noticeable latency spike at snapshot time plus a memory spike from copy-on-write pages during the fork.
- **AOF** (append-only file) — logs every write; more durable (configurable fsync policy via `appendfsync`), at the cost of more disk I/O and a separate background rewrite process (also fork-based) to keep the file from growing unbounded.
- **Hybrid** (RDB preamble + AOF, the modern default when both are enabled) — balances durability and recovery speed but still inherits the fork-based latency characteristics of both mechanisms above.

Changing the persistence mode affects what survives a crash or restart — classify as **High Risk** and state the durability tradeoff explicitly rather than presenting it purely as a performance change.

## Key design

- Use a consistent namespacing convention (`entity:id:field`-style) so keys are predictable and scannable without `KEYS *`.
- Avoid storing one large blob as a single key when the application typically only needs to read or update part of it — prefer a structure (hash, sorted set) that supports partial access.
- Set a TTL deliberately on anything that's genuinely cache-shaped, to avoid unbounded keyspace growth; don't rely on eviction alone to bound memory if the access pattern would mean evicting data the application actually expects to still be there.
- In cluster mode, a small number of disproportionately hot keys can overload a single shard even when the cluster overall has capacity — check per-node `CLUSTER` stats or `commandstats` rather than only looking at aggregate cluster memory/CPU.

## Cluster considerations

Resharding (moving hash slots between nodes) is an online operation but consumes real bandwidth and CPU during the move — classify as **Moderate Risk** for a small rebalance and **High Risk** for a large one under load. `MOVED`/`ASK` redirection overhead on the client side is a sign the client's slot cache is stale or the cluster topology changed recently, not necessarily a sign of a sizing problem.

## Caching strategy (Redis as a cache tier in front of another database)

Everything above treats Redis as the system under direct diagnosis. When Redis sits in front of another database as a cache, a different set of failure modes shows up — these are architecture/strategy questions rather than Redis-internals questions, but they come up often enough alongside Redis performance work to cover here:

- **Cache-aside (lazy loading)** — the application checks Redis first, and on a miss, reads from the source database and writes the result into Redis before returning it. The most common pattern; simple, but every miss pays the full source-database latency, and a cold cache (after a restart or a mass eviction) can produce a burst of simultaneous source-database load.
- **Write-through** — writes go to Redis and the source database together (or Redis is updated immediately after a source write), keeping the cache warm for subsequently-read data at the cost of writing to two places on every write. Appropriate when read-after-write consistency for that data matters and write volume doesn't make double-writing a bottleneck.
- **Thundering herd / cache stampede** — many concurrent requests all miss the same (usually just-expired or not-yet-populated) key at once and all hit the source database simultaneously. Mitigate with a short-lived lock/single-flight pattern (the first request to miss populates the cache while others wait briefly or serve stale data), or by adding jitter to TTLs so a batch of keys set at the same time don't all expire in the same instant.
- **Invalidation** — the two standard approaches are TTL-based expiry (simple, but data can be stale for up to the TTL window) and explicit invalidation on write (fresher, but requires finding and evicting every cache key derived from the changed data). A cache key scheme that encodes its own invalidation dependency — e.g. including a version number or the source row's `updated_at` in the key — sidesteps needing to enumerate and delete keys explicitly, at the cost of old keys aging out via TTL rather than being cleaned up immediately.

Which of these is "the fix" depends on the read/write ratio and staleness tolerance of the specific data being cached — ask for both before recommending a pattern, the same evidence-first posture as everything else in this skill.
