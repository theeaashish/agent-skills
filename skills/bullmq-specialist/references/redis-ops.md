# Redis Operations for BullMQ

BullMQ is only as durable as your Redis persistence configuration.
Most queue data loss incidents are Redis config issues, not BullMQ bugs.

---

## Persistence: AOF vs RDB

| Mode             | Durability                          | Performance | BullMQ Verdict                  |
| ---------------- | ----------------------------------- | ----------- | ------------------------------- |
| No persistence   | None — restart = data loss          | Fastest     | ❌ Never in production          |
| RDB only         | Up to last snapshot (5–60 min loss) | Fast        | ⚠️ Only for non-critical queues |
| AOF (`everysec`) | ≤ 1 second loss                     | Moderate    | ✅ Minimum for production       |
| AOF (`always`)   | Near-zero loss                      | Slowest     | ✅ For payment/critical queues  |
| AOF + RDB        | Best recovery options               | Moderate    | ✅ Recommended default          |

```bash
# redis.conf — production minimum
appendonly yes
appendfsync everysec        # fsync every second — good balance
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# For critical queues (payments, auth):
appendfsync always          # every write synced to disk — typically 5–10× slower
                            # on commodity SSD under write-heavy load. Benchmark on
                            # your hardware before committing to this for high-volume queues.
```

---

## Eviction Policy — The Silent Job Killer

```bash
# ❌ NEVER use these with BullMQ — Redis will silently delete jobs under memory pressure
maxmemory-policy volatile-lru      # deletes jobs that have TTLs set
maxmemory-policy allkeys-lru       # deletes ANY key including active jobs
maxmemory-policy volatile-ttl
maxmemory-policy allkeys-random

# ✅ Required for BullMQ
maxmemory-policy noeviction        # OOM error instead of silent deletion
                                   # Lets your app handle it, not Redis
```

**What happens with wrong policy**: Jobs disappear mid-flight. Worker picks up a job → Redis
evicts it → worker crashes with `Missing key error` or job is silently "completed". Extremely
hard to debug because no error appears in BullMQ logs.

```bash
# Verify current policy (run on production Redis)
redis-cli CONFIG GET maxmemory-policy
# → should return: noeviction
```

---

## Memory Economics

```
Approximate Redis memory per job (varies by payload size):

Job state overhead:     ~300–500 bytes base (BullMQ metadata)
Payload:                your data (keep < 1KB)
Waiting set entry:      ~100 bytes (ZSET member)
Completed job retained: same as above — multiplies by removeOnComplete.count

Rule of thumb:
  1,000,000 retained completed jobs × ~1KB avg = ~1GB Redis memory

Monitor with:
  redis-cli INFO memory
  redis-cli MEMORY USAGE bull:emails:completed   # bytes for specific key
```

### ZSET Growth (Delayed Jobs)

```bash
# Delayed jobs live in a sorted set — unbounded growth if jobs never run
redis-cli ZCARD bull:emails:delayed    # count delayed jobs
redis-cli ZCARD bull:emails:wait       # count waiting jobs

# Alert if either exceeds your expected burst ceiling
```

### Retention Tuning

```ts
// Balance: too low = lose debug history; too high = memory cost
defaultJobOptions: {
  removeOnComplete: {
    count: 1000,        // keep last 1000 completed jobs per queue
    age: 24 * 3600,     // AND no older than 24 hours
  },
  removeOnFail: {
    count: 5000,        // keep more failures — you need them for post-mortems
    age: 7 * 24 * 3600, // 7 days for failed jobs
  },
},
```

---

## Replication and Failover

```
Redis Sentinel (standard HA):
  Primary → Replica (async replication, ~ms lag)
  Sentinel monitors → promotes replica on primary failure
  BullMQ reconnects automatically via ioredis Sentinel support

Redis Cluster:
  Jobs are distributed across shards by key hash.
  BullMQ requires ALL keys for a given queue to live on the same slot — set the
  Queue `prefix` option to a hash-tagged string:
    new Queue('emails', { prefix: '{emails}', connection })
  This forces every key (bull:{emails}:wait, bull:{emails}:active, bull:{emails}:delayed,
  the events stream, the meta-key, etc.) to hash to the same slot.

  Multi-queue concerns:
  - A DLQ for `emails` must share the tag: prefix: '{emails}' on dead-letter too,
    or live on the same slot some other way.
  - Job schedulers and rate-limit keys are derived from the queue prefix — same tag covers them.
  - FlowProducer parent + children must share the hash tag (parent prefix). In practice this
    means every queue participating in a flow shares ONE tag, collapsing them to one shard.
    For non-trivial DAGs, FlowProducer on Cluster is effectively unsupported.

Replication lag risk:
  Primary writes job → replica not yet synced → primary crashes
  Replica promoted → job is gone (was in primary's AOF buffer)
  Mitigation: appendfsync always on primary reduces this window
```

---

## Events Stream Trimming

BullMQ writes job lifecycle events to a per-queue Redis Stream (`bull:<queue>:events`).
Without trimming, this stream grows unboundedly and becomes one of the dominant memory
consumers in a busy queue. BullMQ trims to a default `maxLen` per write — confirm and
tune via Queue construction:

```ts
new Queue("emails", {
  connection,
  streams: { events: { maxLen: 10_000 } }, // keep ~10k recent events
});
```

Verify in production:

```bash
redis-cli XLEN bull:emails:events            # current length
redis-cli MEMORY USAGE bull:emails:events    # bytes
```

A stream of 1M entries at ~200 bytes each is 200MB — a common cause of "Redis grew
overnight" pages when retention is left at defaults on a high-throughput queue.

---

## Prioritized and Delayed Job Structures

| Job state                 | Redis structure | Key                                  | Length command |
| ------------------------- | --------------- | ------------------------------------ | -------------- |
| `waiting` (no priority)   | LIST            | `bull:<queue>:wait`                  | `LLEN`         |
| `waiting` (with priority) | ZSET            | `bull:<queue>:prioritized`           | `ZCARD`        |
| `delayed`                 | ZSET            | `bull:<queue>:delayed`               | `ZCARD`        |
| `active`                  | LIST            | `bull:<queue>:active`                | `LLEN`         |
| `waiting-children`        | ZSET            | `bull:<queue>:waiting-children`      | `ZCARD`        |
| `completed` / `failed`    | ZSET            | `bull:<queue>:completed` / `:failed` | `ZCARD`        |
| `stalled-check`           | SET             | `bull:<queue>:stalled`               | `SCARD`        |
| events                    | STREAM          | `bull:<queue>:events`                | `XLEN`         |

**Critical implication for autoscaling**: monitoring only `LLEN bull:<queue>:wait` misses
prioritized jobs, delayed jobs about to fire, and waiting-children. See `scaling.md` for the
correct aggregate queue-depth signal.

---

## Disaster Recovery Checklist

```
□ maxmemory-policy noeviction confirmed
□ appendonly yes with appendfsync everysec minimum
□ Daily RDB backup to S3/GCS (redis-cli BGSAVE + copy dump.rdb)
□ Replica in separate AZ/region
□ Alert on Redis memory > 70% of maxmemory
□ Alert on Redis ZSET keys growing unboundedly
□ Tested failover: promote replica, verify workers reconnect
□ DLQ jobs exported to cold storage (S3) weekly for audit
□ removeOnComplete/removeOnFail counts match memory budget
```

---

## Managed Redis Platform Notes

| Platform      | AOF Default            | Notes                                            |
| ------------- | ---------------------- | ------------------------------------------------ |
| Upstash       | ✅ AOF enabled         | Serverless — eviction policy configurable        |
| Redis Cloud   | ✅ AOF enabled         | Choose persistence tier on plan                  |
| ElastiCache   | ❌ RDB only by default | Must enable AOF explicitly per replication group |
| Railway Redis | ⚠️ RDB only            | Not recommended for critical queues              |
| Fly.io Redis  | ⚠️ Check config        | Review persistence settings per deploy           |
