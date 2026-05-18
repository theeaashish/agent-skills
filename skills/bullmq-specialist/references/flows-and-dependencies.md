# Flows and Job Dependencies

Use `FlowProducer` when jobs have parent/child relationships — parent only completes
after all children complete. This is BullMQ's native DAG execution primitive.

---

## Basic Flow

```ts
import { FlowProducer, WaitingChildrenError } from "bullmq";

const flowProducer = new FlowProducer({ connection: queueConnection });

// Parent waits for all children — children run in parallel
const flow = await flowProducer.add({
  name: "process-order",
  queueName: "orders",
  data: { orderId: "ord_123" },
  children: [
    {
      name: "validate-inventory",
      queueName: "inventory",
      data: { orderId: "ord_123" },
    },
    {
      name: "charge-payment",
      queueName: "payments",
      data: { orderId: "ord_123" },
    },
    {
      name: "reserve-shipping",
      queueName: "shipping",
      data: { orderId: "ord_123" },
    },
  ],
});

console.log("Flow job ID:", flow.job.id);
```

---

## Accessing Child Results from Parent

`getChildrenValues()` returns a map keyed by the **full Redis key path** of each child job:
`<prefix>:<queueName>:<jobId>`. The default prefix is `bull` (configurable via the
`prefix` option on Queue/Worker), so a child in the `inventory` queue with id `42` appears
under the key `bull:inventory:42` — not `inventory:42`.

```ts
// Child result types
interface InventoryResult {
  reserved: boolean;
  warehouseId: string;
}
interface PaymentResult {
  chargeId: string;
  amount: number;
}

const PREFIX = "bull"; // match Queue/Worker `prefix` option

// Parent processor — runs only after ALL children complete
const orderWorker = new Worker<{ orderId: string }>(
  "orders",
  async (job) => {
    const childrenValues = await job.getChildrenValues<
      InventoryResult | PaymentResult
    >();

    // Extract by full prefixed key — `bull:<queueName>:<jobId>`
    const inventoryEntry = Object.entries(childrenValues).find(([key]) =>
      key.startsWith(`${PREFIX}:inventory:`),
    );
    const paymentEntry = Object.entries(childrenValues).find(([key]) =>
      key.startsWith(`${PREFIX}:payments:`),
    );

    if (!inventoryEntry || !paymentEntry) {
      // Children map incomplete — should not happen if BullMQ moved the parent
      // out of waiting-children, but defensive logging catches misconfiguration.
      throw new Error("Missing child results — verify queue prefix matches");
    }

    const inventory = inventoryEntry[1] as InventoryResult;
    const payment = paymentEntry[1] as PaymentResult;

    if (!inventory.reserved)
      throw new Error(`Inventory not reserved for order ${job.data.orderId}`);
    if (!payment.chargeId)
      throw new Error(`Payment not captured for order ${job.data.orderId}`);

    return {
      orderId: job.data.orderId,
      chargeId: payment.chargeId,
      processed: true,
    };
  },
  { connection: workerConnection },
);
```

> **If you customise `prefix`** (e.g., for Redis Cluster hash tags — `prefix: '{tenant1}'`),
> mirror it in your `startsWith` check or you will silently miss every child result.

---

## Step Jobs — Two Distinct Patterns

BullMQ has two separate primitives for "pause and resume" jobs.
**They are not interchangeable** — using the wrong error class causes silent misbehaviour.

### Pattern A: Delayed Step Jobs (`DelayedError`)

Use when the job itself needs to wait for time to pass or an external event,
with no child jobs involved. Pair with `moveToDelayed`.

```ts
import { Worker, DelayedError } from "bullmq";

interface OrderJobData {
  orderId: string;
  step: "init" | "awaiting-webhook" | "confirmed";
}

// BullMQ passes (job, token) — use the token parameter, not `job.token`.
const stepWorker = new Worker<OrderJobData>(
  "orders",
  async (job, token) => {
    switch (job.data.step) {
      case "init": {
        await initiatePayment(job.data.orderId);
        await job.updateData({ ...job.data, step: "awaiting-webhook" });
        await job.moveToDelayed(Date.now() + 5 * 60_000, token);
        throw new DelayedError(); // ← releases the lock; BullMQ re-queues as delayed
      }

      case "awaiting-webhook": {
        const payment = await getPaymentStatus(job.data.orderId);
        if (!payment.confirmed) {
          await job.moveToDelayed(Date.now() + 10_000, token);
          throw new DelayedError();
        }
        await job.updateData({ ...job.data, step: "confirmed" });
        // Re-park with delay=0 so BullMQ picks the job up again with the new step.
        // Do NOT call worker.processJob() — it is an internal method; calling it
        // re-enters with a held lock and conflicts with the active lock token.
        await job.moveToDelayed(Date.now(), token);
        throw new DelayedError();
      }

      case "confirmed": {
        await fulfillOrder(job.data.orderId);
        return { fulfilled: true };
      }
    }
  },
  { connection: workerConnection },
);
```

> **Rule**: `moveToDelayed` → always throw `DelayedError` immediately after.
> BullMQ uses the thrown error to cleanly release the job lock before re-queuing.
> Not throwing it leaves the lock dangling until it expires.

### Pattern B: Waiting for Child Jobs (`WaitingChildrenError`)

Use only inside a FlowProducer parent job when the parent needs to pause
until dynamically added children complete. This is not a general "pause" signal.

```ts
import { Worker, WaitingChildrenError } from "bullmq";

const parentWorker = new Worker(
  "orders",
  async (job, token) => {
    // Stage 1 — add children dynamically after inspecting job data
    const shouldRunFraud = job.data.amount > 1000;

    if (shouldRunFraud) {
      const queue = new Queue("fraud-check", { connection: queueConnection });
      await queue.add(
        "check",
        { orderId: job.data.orderId },
        {
          parent: { id: job.id!, queue: "orders" }, // link child to this parent
        },
      );
    }

    // Pause parent until all children finish
    // moveToWaitingChildren returns true if there are children pending
    const hasPending = await job.moveToWaitingChildren(token!);
    if (hasPending) {
      throw new WaitingChildrenError(); // ← correct sentinel — not for delayed jobs
    }

    // No pending children — proceed to final stage
    const childrenValues = await job.getChildrenValues();
    await finaliseOrder(job.data.orderId, childrenValues);
    return { done: true };
  },
  { connection: workerConnection },
);
```

> **Rule**: `moveToWaitingChildren` → throw `WaitingChildrenError` only if it returns `true`.
> `WaitingChildrenError` must never be used with `moveToDelayed` — it signals a different
> internal BullMQ state transition and will cause incorrect behaviour if misused.

---

## Typed FlowProducer

```ts
interface OrderJobData {
  orderId: string;
  step?: string;
}
interface InventoryJobData {
  orderId: string;
}
interface PaymentJobData {
  orderId: string;
  amount: number;
}

// FlowProducer doesn't support generic params directly — type your add() call instead
await (flowProducer as FlowProducer).add({
  name: "process-order",
  queueName: "orders",
  data: { orderId: "ord_123" } satisfies OrderJobData,
  children: [
    {
      name: "validate-inventory",
      queueName: "inventory",
      data: { orderId: "ord_123" } satisfies InventoryJobData,
    },
  ],
});
```

---

## Redis Cluster Caveats

FlowProducer relies on **multi-key Lua scripts** that touch the parent, children, and meta-keys
atomically. In Redis Cluster, multi-key operations require all keys to live on the same hash slot.

- Children added with `parent: { id, queue }` MUST hash to the same slot as the parent — use
  matching `prefix` hash tags on both queues (e.g., `prefix: '{flow-tenant1}'`).
- Children in **different queues** are only possible on Cluster if every queue shares a hash tag.
  This collapses every flow-participating queue onto one shard, defeating Cluster's horizontal
  scale benefits.
- The BullMQ team has historically advised against FlowProducer on Cluster for non-trivial DAGs.
  If you need DAGs at multi-shard scale, use Sentinel HA with vertical Redis scale instead, or
  decompose the DAG into independent queues coordinated by an external orchestrator (Temporal).

> **Recursive flows**: Adding children from within a parent processor that itself uses
> `WaitingChildrenError` re-enters the same code path on each resume. Cap recursion depth
> in `job.data` and treat the parent as a state machine — unbounded fanout will eventually
> exhaust Redis memory and Worker concurrency simultaneously.

---

## Flow Observability

```ts
// Get the full job tree by parent job ID
const flow = await flowProducer.getFlow({
  id: parentJobId,
  queueName: "orders",
  // depth controls how many child levels to fetch
  depth: 3,
  maxChildren: 100,
});

// flow.children is an array of JobNode with nested .children
const childStates = flow.children?.map((c) => ({
  name: c.job.name,
  state: c.job.finishedOn,
}));
```
