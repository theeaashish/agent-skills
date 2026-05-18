# Queue Architecture

Queue topology is an architectural decision — getting it wrong compounds at scale.
Read this before designing a new queue system or adding more than 3 queues.

---

## When to Split Queues

Split into separate queues when workloads have **different SLAs, priorities, or resource profiles**.

```
queues/
  critical/       → payment confirmations, auth events     (SLA: < 2s, concurrency: 2)
  default/        → order processing, user actions         (SLA: < 30s, concurrency: 10)
  bulk/           → digest emails, reports, exports        (SLA: < 10min, concurrency: 20)
  webhooks/       → inbound webhook processing             (SLA: < 5s, concurrency: 15)
  ai:inference/   → LLM calls, embeddings                  (SLA: < 60s, concurrency: 3)
  notifications/  → push, email, SMS                       (SLA: < 5min, concurrency: 25)
```

**Split signals**:

- Different retry/backoff requirements
- Different downstream rate limits (Stripe vs Sendgrid)
- Different concurrency needs (CPU-bound vs I/O-bound)
- Different on-call escalation policies

**Do NOT split** just because jobs have different names — queue explosion (50+ queues) makes
Bull Board unusable and multiplies Redis connection overhead (3 connections × N queues × M replicas).

---

## Priority Lane Architecture

BullMQ supports numeric priority (1 = highest). Use sparingly — priority inversion is real.

```ts
// Priority scale: 1 (urgent) → 10 (bulk). Avoid more than 3-4 tiers.
await orderQueue.add("payment-confirmation", data, { priority: 1 }); // always first
await orderQueue.add("order-update", data, { priority: 5 }); // default
await orderQueue.add("weekly-digest", data, { priority: 10 }); // bulk

// ⚠️ Priority starvation: a constant flood of priority=1 jobs blocks priority=10 forever.
// Solution: dedicate separate workers for critical vs bulk workloads.
```

### Reserved Worker Pools (Priority Inversion Prevention)

BullMQ OSS workers pick jobs from a queue in priority order (1 first) — they cannot be
restricted to a priority range on the same queue. "Filter inside the processor" does NOT
give you a reserved pool: the worker has already pulled the job off the queue and burned a
concurrency slot. To truly reserve capacity for high-priority work, use **separate queues**
and a router at the producer side.

```ts
// Two distinct queues — capacity is hard-partitioned
const criticalOrderQueue = new Queue("orders:critical", {
  connection: queueConnection,
});
const bulkOrderQueue = new Queue("orders:bulk", {
  connection: queueConnection,
});

// Producer routes by SLA tier
export const enqueueOrder = (order: OrderInput) => {
  const q = order.tier === "critical" ? criticalOrderQueue : bulkOrderQueue;
  return q.add("process", order);
};

// Dedicated critical worker — its concurrency slots are reserved for critical work
const criticalWorker = new Worker("orders:critical", processor, {
  connection: workerConnection,
  concurrency: 2,
});

// Bulk worker — separate pool, cannot starve critical
const bulkWorker = new Worker("orders:bulk", processor, {
  connection: bulkWorkerConnection,
  concurrency: 20,
});
```

> If you have BullMQ Pro, prefer **groups** on a single queue — they implement true per-group
> fairness in the broker, without the queue-explosion cost of one queue per tier.

---

## Multi-Tenant Queue Design

### Option A: Shared Queue + Tenant Tag (Simple SaaS)

```ts
interface JobData {
  tenantId: string;   // always include for observability
  // ... job-specific fields
}

// All tenants share one queue — simple, but no isolation
await emailQueue.add('send', { tenantId: 'acme', to: 'user@acme.com', ... });
```

**Use when**: All tenants have similar usage patterns, no SLA differentiation needed.  
**Risk**: One noisy tenant can starve others (thundering herd per tenant).

### Option B: Per-Tenant Queues (Isolated SaaS)

```ts
// Queue name includes tenantId — full isolation
const getTenantQueue = (tenantId: string) =>
  new Queue(`tenant:${tenantId}:email:send`, {
    connection: queueConnection,
    ...defaultOpts,
  });

// Create queue lazily on first job, cache in Map
const queueCache = new Map<string, Queue>();

export const getOrCreateQueue = (tenantId: string): Queue => {
  if (!queueCache.has(tenantId)) {
    queueCache.set(tenantId, getTenantQueue(tenantId));
  }
  return queueCache.get(tenantId)!;
};
```

**Use when**: Enterprise SaaS with per-tenant SLAs, billing by usage, or compliance isolation.  
**Risk**: Queue explosion if tenant count > 100 — set a queue count cap and alert on it.

### Option C: BullMQ Pro Groups (Best for SaaS)

```ts
// BullMQ Pro groups: logical per-tenant rate limiting on a shared queue
await emailQueue.add("send", data, {
  group: { id: `tenant:${tenantId}`, limit: { max: 50, duration: 60_000 } },
});
// Each tenant gets independent rate limit — one queue, N isolated groups
```

**Use when**: You have BullMQ Pro license — this is the correct architecture for most SaaS products.

---

## Burst Isolation

Prevent a traffic spike in one domain from affecting another.

```ts
// Separate queue + separate worker pool for burst-prone workloads
const webhookQueue = new Queue("webhooks", { connection: queueConnection });
const webhookWorker = new Worker("webhooks", processWebhook, {
  connection: workerConnection,
  concurrency: 30, // high concurrency — mostly I/O wait
  limiter: { max: 500, duration: 60_000 }, // global inbound webhook cap
});

// Main business queue is completely unaffected by webhook burst
const orderQueue = new Queue("orders", { connection: queueConnection });
const orderWorker = new Worker("orders", processOrder, {
  connection: workerConnection,
  concurrency: 5, // conservative — touches payment APIs
});
```

---

## Queue Explosion Warning

| Queue count | Status                                                   |
| ----------- | -------------------------------------------------------- |
| 1–10        | Ideal — easy to operate                                  |
| 10–30       | Acceptable with naming discipline                        |
| 30–60       | Warning — Redis connection overhead, Bull Board unusable |
| 60+         | Stop — redesign with groups or tenant sharding           |

Rule: Before adding a new queue, ask "can this be a job name or priority on an existing queue?"
