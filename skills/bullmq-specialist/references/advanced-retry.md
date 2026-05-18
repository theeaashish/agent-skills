# Advanced Retry Strategies

BullMQ's built-in `exponential` backoff is a starting point.
Production systems need retry classification, jitter, and circuit breakers.

---

## Retry Classification

Not all failures deserve the same retry strategy. Classify first.

| Failure Type      | Example                       | Strategy                                  |
| ----------------- | ----------------------------- | ----------------------------------------- |
| Transient         | Network blip, DB timeout      | Retry with exponential + jitter           |
| Rate limited      | Stripe 429, Sendgrid 429      | Retry after `Retry-After` header delay    |
| Overload          | DB pool exhausted             | Retry with longer delay + circuit breaker |
| Permanent         | Invalid email, missing record | `UnrecoverableError` — don't retry        |
| Dependency outage | Payment provider down         | Retry with long delay + alert             |

```ts
import { UnrecoverableError, DelayedError, Worker } from "bullmq";

async function emailProcessor(
  this: Worker,
  job: Job<EmailJobData>,
  token?: string,
) {
  try {
    return await sendEmail(job.data);
  } catch (err: any) {
    // Permanent failures — skip retry entirely
    if (err.code === "INVALID_EMAIL" || err.status === 422) {
      throw new UnrecoverableError(err.message);
    }

    // Rate limit — use BullMQ's rate-limit primitive instead of moveToDelayed.
    // worker.rateLimit() pauses fetches globally across all replicas; throwing
    // RateLimitError tells BullMQ to put this job back into wait without counting
    // an attempt. This is the v5-idiomatic way to handle 429s.
    if (err.status === 429) {
      const retryAfter = parseInt(err.headers?.["retry-after"] ?? "60", 10);
      await this.rateLimit(retryAfter * 1000);
      throw new Worker.RateLimitError();
    }

    // All other errors — let BullMQ retry with configured backoff
    throw err;
  }
}
```

> **Why not `moveToDelayed` + `throw new Error(...)` for 429s?** That pattern counts the attempt
> against `attempts`, so a sustained rate-limit period exhausts retries and ships the job to DLQ
> spuriously. It also requires the `DelayedError` sentinel (not a plain `Error`) and the worker's
> `token` to release the lock cleanly — getting any of that wrong leaves the job lock dangling
> until `lockDuration` expires, blocking the job for up to a minute per retry.
>
> If you must use `moveToDelayed` (e.g., for adaptive scheduling that is **not** a rate-limit
> response), the correct shape is below — note the `token` argument and the `DelayedError`:
>
> ```ts
> await job.moveToDelayed(Date.now() + delayMs, token);
> throw new DelayedError();
> ```

---

## Full Jitter Implementation

BullMQ's `exponential` backoff is deterministic — all failed jobs retry at the same time
after an outage (thundering herd). Add full jitter to spread retries.

```ts
// Custom backoff strategy — registered on the Worker, named via Queue's backoff option.
import { Worker, Queue, MinimalJob } from "bullmq";

// v5 signature: (attemptsMade, type, err, job) => delayMs
// `type` is the string set on the Queue's backoff option (here: 'full-jitter').
// `err` is the actual error thrown by the processor — useful for per-error tuning.
const fullJitterBackoff = (
  attemptsMade: number,
  type: string,
  err: Error,
  job?: MinimalJob,
): number => {
  const base = 1000; // 1s base
  const cap = 30_000; // 30s max
  const exponential = Math.min(cap, base * Math.pow(2, attemptsMade));
  // Decorrelated jitter — never returns 0 (which would cause immediate hot-loop retries)
  return Math.floor(base + Math.random() * (exponential - base));
};

// Register the strategy under a stable name; the Queue references that name.
const emailWorker = new Worker("email:send", emailProcessor, {
  connection: workerConnection,
  settings: {
    backoffStrategy: fullJitterBackoff,
  },
});

export const emailQueue = new Queue("email:send", {
  connection: queueConnection,
  defaultJobOptions: {
    attempts: 6,
    backoff: { type: "custom" }, // routes to the Worker's backoffStrategy fn
  },
});
```

> **Decorrelated jitter vs full jitter**: AWS's "full jitter" formula (`random(0, exp)`) can
> return 0, retrying immediately and defeating the point of backoff during a sustained outage.
> The decorrelated form (`base + random(0, exp - base)`) keeps a floor while still spreading the
> retry wave. Either is fine; pick one and stick to it across queues so your retry tail is predictable.

---

## Circuit Breaker

Prevent retry storms when a downstream dependency is fully down.

```ts
interface CircuitState {
  failures: number;
  lastFailure: number;
  state: "closed" | "open" | "half-open";
}

class QueueCircuitBreaker {
  private state: CircuitState = {
    failures: 0,
    lastFailure: 0,
    state: "closed",
  };
  private readonly threshold = 10; // open after 10 failures
  private readonly resetTimeout = 60_000; // try again after 60s

  isOpen(): boolean {
    if (this.state.state === "open") {
      // Auto-transition to half-open after reset timeout
      if (Date.now() - this.state.lastFailure > this.resetTimeout) {
        this.state.state = "half-open";
        return false;
      }
      return true;
    }
    return false;
  }

  recordSuccess() {
    this.state = { failures: 0, lastFailure: 0, state: "closed" };
  }

  recordFailure() {
    this.state.failures++;
    this.state.lastFailure = Date.now();
    if (this.state.failures >= this.threshold) {
      this.state.state = "open";
      console.warn("[circuit-breaker] OPEN — downstream appears down");
    }
  }
}

const breaker = new QueueCircuitBreaker();

// ⚠️ Multi-replica caveat: this breaker is in-process. With N worker pods you have N
// independent breakers. That is usually acceptable — each pod independently discovers
// the downstream is down and opens its own breaker within ~10 failures. If you need a
// single shared breaker (e.g., to stop pummelling a paid API with N × threshold failures
// before opening), back this with Redis: INCR a failure counter per queue and read it
// at the start of each processor.

async function emailProcessor(
  this: Worker,
  job: Job<EmailJobData>,
  token?: string,
) {
  if (breaker.isOpen()) {
    // Delay this attempt without burning a retry — pair with DelayedError + token.
    // (moveToDelayed counts as a state transition; combine with attempts logic if you
    // want the circuit-breaker delay to NOT consume an attempt — call job.discard()
    // before moveToDelayed if so.)
    await job.moveToDelayed(Date.now() + 30_000, token);
    throw new DelayedError(); // ← sentinel; lets BullMQ release the lock cleanly
  }

  try {
    const result = await sendEmail(job.data);
    breaker.recordSuccess();
    return result;
  } catch (err) {
    breaker.recordFailure();
    throw err;
  }
}
```

---

## Adaptive Retry Based on Queue Depth

When the queue is deep (backlogged), aggressive retry makes it worse.

```ts
async function adaptiveProcessor(this: Worker, job: Job, token?: string) {
  try {
    return await processJob(job.data);
  } catch (err) {
    // Include 'prioritized' — priority jobs live in the prioritized ZSET, not the wait LIST.
    const counts = await myQueue.getJobCounts(
      "waiting",
      "delayed",
      "prioritized",
    );
    const backlogSize = counts.waiting + counts.delayed + counts.prioritized;

    const adaptiveDelay = Math.min(backlogSize * 100, 60_000); // max 60s
    await job.moveToDelayed(Date.now() + adaptiveDelay, token);
    throw new DelayedError(); // sentinel — releases the lock cleanly
  }
}
```

> **Retry storm prevention**: `moveToDelayed` does **not** count an attempt — it parks the job
> and re-runs it later. That means a buggy adaptive branch can loop forever. Always either
> (a) increment a custom counter in `job.data` and `throw new UnrecoverableError` past a cap, or
> (b) let `attempts` exhaust by `throw`ing a real error after a budget of adaptive deferrals.
