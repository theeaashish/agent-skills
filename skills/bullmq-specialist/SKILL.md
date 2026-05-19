---
name: bullmq-specialist
description: >
  Staff/principal-level BullMQ expert for Redis-backed job queues in Node.js/TypeScript.
  Use this skill whenever the user mentions BullMQ, Bull queues, Redis job queues,
  background processing, delayed jobs, repeatable jobs, worker processes, job scheduling,
  async task processing, job flows, job dependencies, queue architecture, multi-tenant queues,
  AI/LLM inference queues, job schema versioning, queue security, Redis persistence for queues,
  worker autoscaling, KEDA, or production queue incidents — even if they don't use "BullMQ" explicitly.
  Also trigger for retry design, DLQ, deduplication, concurrency, distributed tracing of jobs,
  queue topology, priority lanes, or any queue-related incident playbook question.
  This skill targets senior/staff developers building scalable, production SaaS systems.
risk: none
---

# BullMQ Specialist — Production Grade

Senior-focused BullMQ reference for scalable Node.js/TypeScript systems.
Covers patterns most docs skip: DLQ, deduplication, typed queues, multi-replica
rate limiting, stalled job tuning, and DB pool coordination.

## Principles

- **Idempotency is your contract** — jobs may run more than once; design processors accordingly
- **Pass IDs, never payloads** — job data lives in Redis; keep it under 1KB
- **DLQ is not optional** — exhausted jobs must land somewhere observable
- **Multi-replica invalidates per-worker limits** — rate limiters and concurrency must account for N replicas
- **DB pool size ≥ worker concurrency** — mismatching these causes silent queue stalls
- **Dedicated connections per role** — Queue, Worker, and QueueEvents each need their own IORedis instance
- **`maxRetriesPerRequest: null` is mandatory for every BullMQ connection** — Queue, Worker, QueueEvents, FlowProducer. v5 throws if you pass a non-null value. Use a separate ioredis instance for non-BullMQ fail-fast calls.
- **QueueEvents fans out to every replica** — it is NOT a consumer group. Never put side effects (DLQ inserts, billing, audit rows) in QueueEvents handlers without an idempotency key or leader election
- **Exponential backoff with jitter** — prevents thundering herd after outages
- **Graceful shutdown is a deploy requirement** — handle SIGTERM or lose in-flight jobs
- **Stalled job config is tuning, not default** — always set `stalledInterval` and `maxStalledCount` explicitly
- **Redis eviction policy must be `noeviction`** — `volatile-lru` or `allkeys-lru` silently deletes jobs under memory pressure
- **Never store PII or secrets in job payloads** — Redis is not encrypted at rest by default; pass references, not values
- **Queue topology is architecture** — wrong split decisions compound at scale; read `queue-architecture.md` first
- **Version your job schemas** — rolling deploys mean old jobs run on new worker code simultaneously
- **BullMQ durability = Redis persistence config** — no AOF means jobs are lost on Redis restart

---

## Connection Management

**All BullMQ-owned connections (Queue, Worker, QueueEvents, FlowProducer) must use `maxRetriesPerRequest: null`.** BullMQ v5 enforces this — it will throw at construction time if a connection passed into a Worker (and in many code paths, Queue/QueueEvents/FlowProducer) has a non-null value. Use a **separate, non-BullMQ ioredis instance** if you want HTTP handlers to fail fast on Redis down.

```ts
import IORedis from "ioredis";

// BullMQ connection factory — used by Queue, Worker, QueueEvents, FlowProducer.
// maxRetriesPerRequest: null is REQUIRED by BullMQ v5.
export const createBullConnection = () =>
  new IORedis(process.env.REDIS_URL!, {
    maxRetriesPerRequest: null, // required by BullMQ for all roles
    enableReadyCheck: false,
    retryStrategy: (times) => Math.min(times * 200, 5000),
  });

// Optional: a separate, NON-BullMQ ioredis for ad-hoc producer fail-fast use
// (e.g. health checks, custom Lua scripts, dedup probes). Do NOT pass this into
// Queue/Worker/QueueEvents — only use it for your own Redis calls.
export const createFailFastRedis = () =>
  new IORedis(process.env.REDIS_URL!, {
    maxRetriesPerRequest: 3,
    enableReadyCheck: false,
    retryStrategy: (times) => Math.min(times * 200, 3000),
  });

// Three dedicated BullMQ connections — roles have different blocking semantics
export const queueConnection = createBullConnection(); // Queue producers
export const workerConnection = createBullConnection(); // Worker processors
export const eventsConnection = createBullConnection(); // QueueEvents listeners
```

> **Why three instances?** Workers use blocking Redis commands (`BRPOPLPUSH`/`XREAD BLOCK`) that
> monopolise the connection. QueueEvents has its own blocking stream consumer. Sharing connections
> across roles starves event delivery and can deadlock during reconnect.
>
> **Why not a "producer connection with retries=3"?** BullMQ rejects non-null `maxRetriesPerRequest`
> on every role it owns. If you need fail-fast for an HTTP handler, do the Redis probe yourself with
> a separate ioredis instance and only call `queue.add(...)` once the probe succeeds — or accept
> that `queue.add()` may queue commands in the ioredis offline buffer during a brief blip (this is
> almost always what you want).

---

## TypeScript-First Queue Setup

Always type job data and job return value at the Queue and Worker level.

```ts
import { Queue, Worker, QueueEvents } from "bullmq";

// 1. Define job data shapes — keep them narrow and serialisable
interface EmailJobData {
  to: string;
  templateId: string;
  variables: Record<string, string>;
}

interface EmailJobResult {
  messageId: string;
  sentAt: string; // ISO timestamp
}

// 2. Typed Queue — enforces data shape at add() call site
export const emailQueue = new Queue<EmailJobData, EmailJobResult>("emails", {
  connection: queueConnection,
  defaultJobOptions: {
    attempts: 4,
    backoff: { type: "exponential", delay: 2000 }, // 2s → 4s → 8s → 16s
    removeOnComplete: { count: 2000, age: 86400 }, // keep 24h or 2k, whichever first
    removeOnFail: { count: 10_000 }, // keep failed jobs for post-mortems
    // ⚠️ Cleanup is LAZY — jobs are only removed when the NEXT job in the queue
    // completes or fails. A queue with no throughput retains all jobs indefinitely.
  },
});

// 3. Typed Worker — job.data is fully inferred, no casting
export const emailWorker = new Worker<EmailJobData, EmailJobResult>(
  "emails",
  async (job) => {
    // job.data.to, job.data.templateId — fully typed
    await job.updateProgress(10); // numeric 0–100 or arbitrary object

    const result = await sendEmail(job.data);

    await job.updateProgress(100);
    return { messageId: result.id, sentAt: new Date().toISOString() };
  },
  {
    connection: workerConnection,
    concurrency: 10, // tune per downstream service capacity
    stalledInterval: 30_000, // check for stalled jobs every 30s (default: 30s, be explicit)
    maxStalledCount: 2, // move to failed after 2 stall detections (default: 1)
  },
);

// 4. QueueEvents — uses its own dedicated connection
export const emailQueueEvents = new QueueEvents("emails", {
  connection: eventsConnection,
});

// Progress listener — wire to WebSocket/SSE for real-time UI updates
emailQueueEvents.on("progress", ({ jobId, data }) => {
  broadcast(`job:${jobId}:progress`, data);
});
```

---

## Dead Letter Queue (DLQ)

Required in any production system. Never let exhausted jobs disappear silently.

### Critical: QueueEvents is fan-out, not a consumer group

`QueueEvents` is built on Redis Streams **read in pub/sub fan-out mode** — every `QueueEvents` instance in every replica receives **every event independently**. It is NOT a consumer group with at-most-one delivery. If you wire a DLQ-insert in `QueueEvents.on('failed')` and run 5 worker pods, **each failed job produces 5 DLQ inserts**.

Semantics summary:

| Listener                                 | Delivery                                                     | Use for                                                                           |
| ---------------------------------------- | ------------------------------------------------------------ | --------------------------------------------------------------------------------- |
| `worker.on('failed', …)` (process-local) | Fires once on the replica that owned the job                 | Side effects with **exactly-one-per-attempt** intent (DLQ insert, audit, billing) |
| `queueEvents.on('failed', …)`            | Fires on **every** QueueEvents instance in **every** replica | Pure observability (metrics, broadcasts, fan-out to clients) — must be idempotent |

There is no exactly-once delivery on either side: workers can crash _after_ the side effect fires, and QueueEvents can redeliver on reconnect. Design every side effect to be idempotent.

### Correct DLQ pattern: worker-local listener + idempotent jobId

```ts
import { Queue, Worker, UnrecoverableError, Job } from "bullmq";

interface DLQData {
  originalQueue: string;
  originalJobName: string;
  originalJobId: string;
  originalData: unknown;
  failureReason: string;
  failedAt: string;
  attemptsMade: number;
}

// Dedicated DLQ — separate queue, separate worker, separate alerting
export const dlqQueue = new Queue<DLQData>("dead-letter", {
  connection: queueConnection,
  defaultJobOptions: {
    attempts: 1, // DLQ jobs don't retry — human review only
    removeOnComplete: false, // keep forever
    removeOnFail: false,
  },
});

// Wire DLQ from the WORKER (process-local). Fires once on the replica that processed
// the job, regardless of how many other replicas exist. Combine with a deterministic
// jobId on the DLQ insert so that even if this same replica retries the listener
// after a crash + recovery, the DLQ row is not duplicated.
export const wireDLQ = (worker: Worker, queueName: string) => {
  worker.on("failed", async (job: Job | undefined, err: Error) => {
    if (!job) return; // job was removed before failed fired
    const attempts = job.opts.attempts ?? 1;
    if (job.attemptsMade < attempts) return; // still has retries remaining

    await dlqQueue.add(
      "exhausted",
      {
        originalQueue: queueName,
        originalJobName: job.name,
        originalJobId: job.id!,
        originalData: job.data,
        failureReason: err.message,
        failedAt: new Date().toISOString(),
        attemptsMade: job.attemptsMade,
      },
      {
        // Idempotency: same source job → same DLQ jobId → BullMQ no-ops duplicate adds
        // Note: Replaced colons with dashes because BullMQ v5 strictly prohibits colons in custom job IDs.
        jobId: `dlq-${queueName}-${job.id}`,
      },
    );
    // alert: PagerDuty / Sentry / Slack — also idempotent on this jobId
  });
};

wireDLQ(emailWorker, "email:send");
```

### Why not QueueEvents?

A `QueueEvents('email:send').on('failed', …)` runs in **every** replica. With 5 worker pods you get 5 DLQ inserts per real failure. The previous version of this guide recommended that pattern; it is wrong.

If you absolutely cannot run the listener inside the worker process (e.g., your DLQ runs in a separate consumer service that never instantiates a Worker), you have two safe options:

1. **Idempotent insert from a single replica via leader election** — use `SET dlq:leader:<queue> <podId> NX PX 30000` with periodic renewal; only the leader's QueueEvents handler writes. All other pods see they're not the leader and drop the event.
2. **Idempotent insert with deterministic key from every replica** — same `jobId: \`dlq-${queueName}-${job.id}\``. BullMQ's `add()`with an existing`jobId` is a no-op, so duplicates collapse to one. Cheaper than leader election but produces N Redis round-trips per failure.

### Worker crashes and stall transitions

If a worker is SIGKILLed mid-job, neither `worker.on('failed')` nor `queueEvents.on('failed')` fires for that attempt — the job moves through the `stalled → waiting` transition managed by another live worker's stall scanner. Eventually the retry either succeeds, exhausts, or stalls past `maxStalledCount` and ends up in the `failed` set. The replica that observes the final transition will fire its `worker.on('failed')` once. The idempotent `jobId` on the DLQ insert covers the residual edge case where two replicas race the stall recovery.

```ts
// Observe stalls separately — pure metric, idempotent broadcast
emailQueueEvents.on("stalled", ({ jobId }) => {
  metrics.increment("queue.stalled", { queue: "email:send" });
});

// UnrecoverableError — bypass retry, go straight to failed (and then DLQ)
async function processPayment(job: Job<PaymentJobData>) {
  const card = await getCard(job.data.cardId);
  if (!card) throw new UnrecoverableError("Card not found — skipping retries");
  return charge(card, job.data.amount);
}
```

---

## Job Deduplication

Prevent the same logical job from queuing twice — critical for webhook-driven systems.

```ts
// jobId acts as a deduplication key — BullMQ replaces or ignores duplicates
// Note: Replaced colons with dashes because BullMQ v5 strictly prohibits colons in custom job IDs.
await emailQueue.add(
  "send-welcome",
  { to: user.email, templateId: "welcome", variables: {} },
  {
    jobId: `welcome-${user.id}`, // stable, deterministic ID
    // existing job with same ID is left untouched (add is a no-op)
  },
);

// To force-replace an existing job (re-process):
// queue.remove(jobId) does NOT exist — always fetch the Job instance first, then call job.remove()
const existingJob = await emailQueue.getJob(`welcome-${user.id}`);
if (existingJob) {
  await existingJob.remove(); // job.remove() is the correct API — locked (active) jobs throw
}
await emailQueue.add("send-welcome", data, { jobId: `welcome-${user.id}` });
```

> **Webhook safety**: In webhook handlers always set a deterministic `jobId`
> (e.g., `webhook-${event.id}`) so duplicate deliveries produce one job, not N.

---

## Job Schedulers (Repeatable Jobs — Modern API)

> **⚠️ Deprecation notice**: `queue.add(..., { repeat: { pattern } })` and
> `queue.removeRepeatable()` are deprecated as of BullMQ v5. Use
> `queue.upsertJobScheduler()` and `queue.removeJobScheduler()` instead.
> Do not build new systems on the old `repeat` option.

```ts
// ✅ Modern API — upsertJobScheduler (idempotent, safe to call on every deploy)
await emailQueue.upsertJobScheduler(
  "daily-digest", // scheduler ID — stable, unique per schedule
  { pattern: "0 9 * * *", tz: "America/New_York" }, // cron + required timezone
  {
    name: "send-digest", // job name used for each triggered run
    data: { type: "summary" }, // static job data for every run
    opts: { attempts: 3, backoff: { type: "exponential", delay: 2000 } },
  },
);

// Upsert is idempotent — calling it again with the same ID updates the schedule
// Safe to call on startup/deploy without creating duplicate schedulers

// Remove a scheduler — stops future runs, does not affect in-flight jobs
await emailQueue.removeJobScheduler("daily-digest");

// List all active schedulers on a queue
const schedulers = await emailQueue.getJobSchedulers();
schedulers.forEach((s) => console.log(s.id, s.next)); // next: timestamp of next run
```

> **Timezone is required** — omitting `tz` makes the scheduler use server local time,
> which drifts on DST changes. Always set `tz` explicitly.

> **Scheduler ID uniqueness** — the scheduler ID is the deduplication key.
> Two `upsertJobScheduler` calls with the same ID update the same scheduler,
> not create two. Use a stable, descriptive ID like `'weekly-report-tenant-123'`.

---

## Rate Limiting

BullMQ's worker `limiter` is **globally coordinated via Redis** — it is NOT per-replica.
Per the official docs: "if you have 10 workers for one queue with `limiter: { max: 10, duration: 1000 }`,
still only 10 jobs will be processed per second" across all workers combined.

```ts
// ✅ Worker-level limiter — already global; safe across multiple replicas
// 3 replicas each configured with max:100/min → still only 100/min total, not 300/min
new Worker("email:send", processor, {
  connection: workerConnection,
  concurrency: 10,
  limiter: {
    max: 100, // global across all replicas sharing this queue
    duration: 60_000,
  },
});
```

### Dynamic Rate Limiting (Responding to Upstream 429s)

Use `worker.rateLimit()` + `Worker.RateLimitError` when the upstream tells you to back off. Two things commonly get wrong here:

1. `Worker.RateLimitError` must be invoked with `new` — `throw new Worker.RateLimitError()`.
2. The processor closure cannot reference the `Worker` instance variable directly — at the time the closure is defined (inside the `Worker` constructor call), the variable is still in the temporal dead zone. Use the second `this`-bound `Worker` argument that BullMQ passes, or capture the worker after construction.

```ts
// ✅ Correct — declare the processor as a standalone function so we can pass the
// worker reference in after construction, then bind it via setProcessor-style closure.
const sendgridLimiter = { max: 100, duration: 60_000 };

async function emailProcessor(this: Worker, job: Job<EmailJobData>) {
  const result = await callSendgrid(job.data);

  if (result.status === 429) {
    const retryAfterMs =
      parseInt(result.headers["retry-after"] ?? "60", 10) * 1000;
    // `this` is the Worker instance — bound by BullMQ when invoking the processor
    await this.rateLimit(retryAfterMs);
    throw new Worker.RateLimitError(); // moves job back to wait, applies the rate limit
  }
  return result;
}

export const emailWorker = new Worker<EmailJobData>(
  "email:send",
  emailProcessor,
  { connection: workerConnection, limiter: sendgridLimiter },
);
```

> **`worker.rateLimit(ms)` is global**: it pauses fetches across _all_ replicas sharing the queue
> (via a Redis-coordinated counter), matching the global semantics of the static `limiter` option.

### Per-Tenant Rate Limiting (BullMQ Pro)

OSS `limiter` is global across all jobs. For per-tenant throttling, use BullMQ Pro groups:

```ts
// BullMQ Pro — each tenant gets an independent rate limit on the same queue
await emailQueue.add("send", data, {
  group: { id: `tenant:${tenantId}`, limit: { max: 50, duration: 60_000 } },
});
// OSS alternative: run separate queues per tenant (see queue-architecture.md)
```

> **Note on `maximumRateLimitDelay`**: Workers will not fetch new jobs while rate-limited.
> Set `maximumRateLimitDelay` on the Worker to cap how long workers sit idle during rate
> limiting — prevents delayed jobs from being starved indefinitely.

---

## Concurrency and DB Pool Alignment

Two separate concerns — get both wrong independently, hit different failure modes:

```ts
// Rule 1 — PER PROCESS: pool_size ≥ local concurrency + small headroom
// Each worker process needs enough connections for its own concurrent jobs
// Example: concurrency=10 → pool=12 (headroom for BullMQ's own Redis + DB overhead)

// Rule 2 — GLOBAL DB CAPACITY: max_connections ≥ sum of all per-process pools
// Example: 3 replicas × pool=12 → DB needs max_connections ≥ 36
// This is a DB server setting, not a per-process setting

// ❌ WRONG — sets per-process pool to replica_count × concurrency
// This over-allocates connections per process AND blows DB global limit
// pool_size ≥ concurrency × replica_count  ← wrong formula

// ✅ CORRECT — Prisma per-process pool
const prisma = new PrismaClient({
  datasources: { db: { url: process.env.DATABASE_URL } },
  // connection_limit = local concurrency + headroom (e.g. 12 for concurrency=10)
  // pool_timeout = seconds to wait for a free connection before throwing
  // ?connection_limit=12&pool_timeout=10 in DATABASE_URL
});

// Verify DB global capacity in postgres:
// SHOW max_connections;
// SELECT count(*) FROM pg_stat_activity;  -- current connections

// Undershoot per-process pool: queries queue inside pg-pool, jobs appear slow/stalled
// Undershoot global max_connections: DB rejects new connections, jobs crash with ECONNREFUSED
```

---

## Graceful Shutdown

```ts
let isShuttingDown = false;

const shutdown = async (signal: string) => {
  if (isShuttingDown) return; // prevent double-shutdown on multiple signals
  isShuttingDown = true;

  console.log(`[shutdown] ${signal} received — draining workers`);

  // 1. Stop accepting new jobs (non-blocking)
  await emailWorker.pause();

  // 2. Wait for active jobs to complete — with a hard timeout
  const timeout = setTimeout(() => {
    console.error("[shutdown] Timeout — forcing exit");
    process.exit(1);
  }, 30_000); // 30s grace period

  await emailWorker.close();
  clearTimeout(timeout);

  // 3. Close all connections in order
  await emailQueue.close();
  await emailQueueEvents.close();
  queueConnection.disconnect();
  workerConnection.disconnect();
  eventsConnection.disconnect();

  console.log("[shutdown] Clean exit");
  process.exit(0);
};

process.on("SIGTERM", () => shutdown("SIGTERM"));
process.on("SIGINT", () => shutdown("SIGINT"));
```

---

## Queue Naming Standards

Pattern: `domain:action` — colon-delimited, lowercase, kebab for multi-word.

```
email:send          invoice:generate       webhook:process
email:send-digest   ai:embed               ai:inference

tenant:{tenantId}:email:send              # multi-tenant
{env}:{domain}:{action}                   # env-scoped shared Redis
```

**Never**: `queue1`, `emails2`, `myQueue` — undebuggable at scale.

---

## Job Schema Versioning

Rolling deploys mean old jobs (schema v1) run on new worker code (expecting v2). Design for this explicitly.

```ts
// Always embed a version field in job data
interface EmailJobData {
  _v: 1 | 2; // explicit schema version
  to: string;
  templateId: string;
  // v2 added:
  replyTo?: string;
}

// Worker handles both versions gracefully during rollout window
const emailWorker = new Worker<EmailJobData>("email:send", async (job) => {
  const { _v, to, templateId, replyTo } = job.data;

  // Backfill defaults for older schema versions
  const resolvedReplyTo = _v >= 2 ? replyTo : undefined;

  await sendEmail({ to, templateId, replyTo: resolvedReplyTo });
});

// After full rollout + all v1 jobs drained: remove the compat branch
```

> **Deploy rule**: Never remove a field from job data in the same deploy that removes the reader.
> Two-phase: (1) deploy reader that handles both, (2) after queue drains, deploy cleanup.

---

## Advanced Patterns

For complex patterns, read the reference files as needed:

**Core patterns**

- **`references/flows-and-dependencies.md`** — FlowProducer, parent/child jobs, DAG processing, step jobs
- **`references/sandboxed-processors.md`** — Child process isolation for CPU/memory-intensive jobs
- **`references/monitoring.md`** — Bull Board, QueueEvents observability, Prometheus, Sentry

**Architecture & Scale**

- **`references/queue-architecture.md`** — Queue topology, priority lanes, multi-tenant design, fairness, burst isolation
- **`references/scaling.md`** — KEDA autoscaling, worker lifecycle, rolling deploys, Kubernetes, health probes
- **`references/advanced-retry.md`** — Full jitter, circuit breakers, retry classification, adaptive strategies

**Operations & Security**

- **`references/redis-ops.md`** — AOF/RDB persistence, eviction policy, memory economics, disaster recovery
- **`references/security.md`** — PII handling, payload encryption, audit logging, GDPR, tenant isolation
- **`references/incident-playbooks.md`** — DLQ floods, retry storms, Redis outage, stuck workers, runbooks

**Modern Patterns**

- **`references/ai-queue-patterns.md`** — LLM inference queues, embedding pipelines, GPU routing, token budgeting
- **`references/otel-tracing.md`** — Distributed tracing, correlation IDs, request-to-job spans, microservice trace propagation

---

## Validation Checks

### `maxRetriesPerRequest` missing

**Severity**: ERROR  
BullMQ requires `maxRetriesPerRequest: null`. Omitting it causes workers to halt permanently on any Redis blip.

### Shared IORedis connection across Queue + Worker + QueueEvents

**Severity**: ERROR  
Each role must have its own connection. Sharing causes blocked event loops and missed job events.

### No DLQ wiring on any worker

**Severity**: ERROR  
Exhausted jobs with no DLQ destination are silently discarded. Wire DLQ via `worker.on('failed')` (process-local; fires once on the replica that processed the job) with a deterministic `jobId` (`dlq-<queue>-<originalJobId>`) on the DLQ insert for idempotency. Do **not** wire DLQ via `QueueEvents.on('failed')` — it fans out to every replica and produces N duplicate DLQ inserts per failure.

### Worker `limiter` misunderstood as per-replica

**Severity**: INFO  
BullMQ's worker `limiter` is globally coordinated via Redis — it does NOT multiply per replica.
"10 workers with `max: 10` → still only 10 jobs/sec total" (official docs). The concern is not
multi-replica multiplication; it's that `limiter` is global per-queue, not per-tenant. For
per-tenant throttling use BullMQ Pro groups or separate queues.

### No `stalledInterval` / `maxStalledCount` set

**Severity**: WARNING  
Default stall detection (30s, maxStalledCount=1) is aggressive for slow jobs. Tune explicitly.

### `concurrency` set without DB pool comment

**Severity**: WARNING  
High concurrency without matching pool size causes silent DB queue stalls.

### Job data contains non-ID fields over ~500 bytes

**Severity**: WARNING  
Pass IDs, not full objects. Large payloads bloat Redis memory and slow serialisation.

### Retry without `backoff` strategy

**Severity**: WARNING  
Immediate retries hammer a failing downstream. Always use `exponential` backoff.

### No `jobId` in webhook-triggered `queue.add`

**Severity**: WARNING  
Webhook duplicates cause duplicate job execution. Set a deterministic `jobId`.

### Repeatable job without `tz`

**Severity**: WARNING  
Cron jobs without timezone drift on DST boundaries. Always set `tz` explicitly.

### `UnrecoverableError` not used for business-logic failures

**Severity**: INFO  
Retrying unresolvable failures (missing record, bad data) wastes attempts and delays DLQ visibility. Use `UnrecoverableError`.

### No progress reporting on long-running jobs

**Severity**: INFO  
`job.updateProgress()` enables live UI feedback — use for any job over a few seconds.

### Redis eviction policy not `noeviction`

**Severity**: ERROR  
Any LRU eviction policy silently deletes jobs under memory pressure. Set `maxmemory-policy noeviction` in Redis config.

### PII or secret-looking fields in job data

**Severity**: ERROR  
Fields like `password`, `ssn`, `token`, `apiKey` in job data land in Redis unencrypted. Pass a reference ID and fetch from vault at execution time.

### No `_v` version field in job data interface

**Severity**: WARNING  
Schema evolution without versioning causes silent failures during rolling deploys. Embed `_v` from day one.

### Queue names not following `domain:action` convention

**Severity**: WARNING  
Inconsistent naming causes collision in shared Redis and unreadable Bull Board. Enforce `domain:action` pattern.

### `repeat` option used instead of `upsertJobScheduler`

**Severity**: WARNING  
`queue.add(..., { repeat })` is deprecated in BullMQ v5. Migrate to `queue.upsertJobScheduler()` — it is idempotent, explicitly named, and safely updatable on every deploy without creating duplicates.

### No AOF persistence confirmed for Redis

**Severity**: WARNING  
Without `appendonly yes` in Redis config, a restart loses all waiting/delayed jobs. Verify persistence tier before production.

---

## Anti-Patterns

| ❌ Anti-Pattern                                                                               | ✅ Fix                                                                                                                                          |
| --------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `void queue.add(...)` for critical jobs (payments, orders, webhooks)                          | `await queue.add(...)` — if Redis is down, you must know immediately; silently losing a payment job is worse than a failed HTTP response        |
| `await queue.add(...)` for non-critical analytics/telemetry in hot path                       | `void queue.add(...)` — acceptable for best-effort events where loss is tolerable                                                               |
| `try/catch` swallowing all errors in processor                                                | Let errors bubble — BullMQ manages retry/fail state                                                                                             |
| `queue.remove(jobId)` — method does not exist                                                 | `const job = await queue.getJob(jobId); await job?.remove()`                                                                                    |
| `new Queue(...)` created per request                                                          | Singleton Queue at startup; reuse everywhere                                                                                                    |
| `new IORedis()` inside the processor function                                                 | Create connections once at module level                                                                                                         |
| Non-null `maxRetriesPerRequest` on a connection passed into Queue/Worker/QueueEvents          | Always `null` for BullMQ-owned connections (v5 throws otherwise). Use a separate ioredis instance for non-BullMQ fail-fast calls.               |
| DLQ insert / audit log / billing increment wired to `QueueEvents.on('failed' \| 'completed')` | Wire to `worker.on(...)` (process-local) **and** use a deterministic idempotency key on the side effect — QueueEvents fans out to every replica |
| Storing full objects in job data                                                              | Store only IDs; fetch full data inside the processor                                                                                            |
| `removeOnComplete: true` in production                                                        | Use `{ count, age }` — you need history for debugging                                                                                           |
| Empty `catch` in shutdown handler                                                             | Always `clearTimeout` + `process.exit(1)` on drain failure                                                                                      |

---

## Scope Boundaries

| Concern                                         | Delegate To                                                             |
| ----------------------------------------------- | ----------------------------------------------------------------------- |
| Redis cluster / memory tuning                   | `redis-specialist`                                                      |
| Serverless queues (no Redis)                    | Upstash QStash                                                          |
| Complex saga / compensation / durable workflows | Temporal (`references/incident-playbooks.md` has the handoff heuristic) |
| Event sourcing / CQRS / Kafka integration       | `event-architect`                                                       |
| Worker Kubernetes scaling / KEDA setup          | `devops` + `references/scaling.md`                                      |
| Metrics / alerting dashboards                   | `performance-hunter` + `references/monitoring.md`                       |
| AI/LLM inference routing                        | `references/ai-queue-patterns.md`                                       |
| Security audit / compliance                     | `references/security.md`                                                |
