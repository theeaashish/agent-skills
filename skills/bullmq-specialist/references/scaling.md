# Scaling & Deployment

---

## KEDA — Queue-Depth Autoscaling

KEDA scales worker pods based on BullMQ queue depth. CPU-based HPA is the wrong signal for queue workers (a busy worker often blocks on I/O, showing low CPU).

### The wrong signal: `LLEN bull:<queue>:wait`

KEDA's built-in `redis` scaler reads a single LIST length. This is **insufficient** for BullMQ:

- `wait` is a LIST only for jobs **without** a `priority` option. Jobs with priority go into the `prioritized` ZSET — `LLEN wait` reports 0 while real work is queued.
- Delayed jobs live in the `delayed` ZSET — never visible to `LLEN wait`.
- Waiting-children jobs (FlowProducer parents) live in the `waiting-children` ZSET.

A queue full of high-priority jobs with `LLEN`-based scaling will **never scale up**, missing SLAs silently. The KEDA `redis` scaler also has no `zsetName`/`zsetLength` fields — those were a fabricated example in the previous version of this guide.

### Correct signal: aggregate queue depth via Prometheus

Expose an aggregate metric from any pod (or a sidecar) that sums every state KEDA needs to see, and scale on that via the `prometheus` trigger.

```ts
// Exporter — runs on one pod (or all; the metric is read by KEDA from Prometheus)
import client from "prom-client";

const queueDepth = new client.Gauge({
  name: "bullmq_queue_depth",
  help: "Sum of waiting + prioritized + delayed + waiting-children",
  labelNames: ["queue"],
});

setInterval(async () => {
  const counts = await emailQueue.getJobCounts(
    "waiting",
    "prioritized",
    "delayed",
    "waiting-children",
  );
  const total =
    (counts.waiting ?? 0) +
    (counts.prioritized ?? 0) +
    (counts.delayed ?? 0) +
    (counts["waiting-children"] ?? 0);
  queueDepth.set({ queue: "email:send" }, total);
}, 5_000);
```

```yaml
# keda-scaledobject.yaml — scale on the aggregate metric
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: email-worker-scaler
spec:
  scaleTargetRef:
    name: email-worker-deployment
  minReplicaCount: 1 # always keep 1 warm — cold start adds latency
  maxReplicaCount: 20
  cooldownPeriod: 60
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.monitoring.svc:9090
        metricName: bullmq_queue_depth
        threshold: "50" # scale up when aggregate depth exceeds 50
        query: |
          sum(bullmq_queue_depth{queue="email:send"})
```

> **If you cannot run Prometheus**, the `redis` scaler can be used as a partial signal — but
> document the blind spots (priority, delayed, waiting-children) and only use it for queues
> that do not use priority or delayed jobs.
>
> **Counting delayed jobs**: only delayed jobs whose `delayUntil` is past should drive scale-up.
> The simple `ZCARD delayed` over-counts (long-future jobs are not actionable). For accuracy,
> use `ZCOUNT bull:<queue>:delayed -inf <now>` in the exporter.

### Scale-to-Zero Caveat

```yaml
minReplicaCount:
  0 # ⚠️ Risk: cold start adds ~10–30s to first job
  # Only use for batch/non-SLA workloads (reports, exports)
  # For real-time queues: keep minReplicaCount: 1
```

---

## Worker Lifecycle Management

### Warmup Phase

```ts
// Allow worker to "warm up" before accepting high concurrency
// Useful when processor initialises heavy resources (ML model, DB pool)
const worker = new Worker("ai:inference", processor, {
  connection: workerConnection,
  concurrency: 1, // start conservative
});

// After 30s warmup window, increase concurrency
setTimeout(async () => {
  await worker.setConcurrency(10);
  console.log("[worker] Warmed up — concurrency: 10");
}, 30_000);
```

### Rolling Deploy — Zero Job Loss

```
Deploy sequence for worker pods:

1. New pod starts → worker begins with worker.pause() (not accepting jobs)
2. New pod passes readiness probe (health check returns 200)
3. K8s sends SIGTERM to old pod
4. Old pod: worker.pause() → worker.close() → drain in-flight jobs
5. Old pod exits cleanly after grace period
6. New pod: worker.resume() — begins processing

K8s terminationGracePeriodSeconds must be > your longest job duration.
```

```yaml
# deployment.yaml
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 120 # must exceed max job duration
      containers:
        - name: email-worker
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"] # allow SIGTERM handler to run
```

### SIGKILL and Orphaned Jobs

```ts
// SIGKILL cannot be caught — BullMQ handles this via stall detection
// A job moves from 'active' → 'stalled' when worker misses heartbeat
// After maxStalledCount stalls → moved to 'failed'

// Set lockDuration to match your longest expected job + buffer.
// BullMQ v5 auto-renews the lock at lockDuration / 2 — there is no `lockRenewTime` option
// (it was a Bull v3 carry-over and is silently ignored in v5).
const worker = new Worker("orders", processor, {
  connection: workerConnection,
  lockDuration: 60_000, // job lock; auto-renewed every 30s by BullMQ
  stalledInterval: 30_000, // how often a worker scans for stalled jobs
  maxStalledCount: 2, // move to failed after N stall detections
});
```

---

## Health Checks

```ts
import express from "express";

const app = express();

// Readiness: worker is connected and ready to process
app.get("/health/ready", async (req, res) => {
  try {
    await workerConnection.ping(); // Redis is reachable
    const isPaused = await emailQueue.isPaused();

    if (isPaused) return res.status(503).json({ status: "paused" });
    res.json({ status: "ready" });
  } catch {
    res.status(503).json({ status: "redis-unreachable" });
  }
});

// Liveness: worker process itself is alive (not stuck)
app.get("/health/live", (req, res) => {
  // If we can respond, the process is alive
  res.json({ status: "alive", uptime: process.uptime() });
});

app.listen(8080);
```

---

## Docker Production Setup

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM node:20-alpine AS worker
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules

# Run as non-root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Health check for container orchestration
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD wget -qO- http://localhost:8080/health/live || exit 1

CMD ["node", "dist/worker.js"]
```

---

## Testing: Integration Patterns

```ts
import { Queue, Worker, QueueEvents } from "bullmq";
import { GenericContainer } from "testcontainers";

describe("EmailWorker", () => {
  let redisContainer: any;
  let testQueue: Queue<EmailJobData>;
  let testWorker: Worker<EmailJobData>;
  let testEvents: QueueEvents;

  beforeAll(async () => {
    // Spin up real Redis for integration tests — no mocking
    redisContainer = await new GenericContainer("redis:7-alpine")
      .withExposedPorts(6379)
      .start();

    const redisUrl = `redis://localhost:${redisContainer.getMappedPort(6379)}`;
    // Each role gets its own ioredis instance — even in tests. Sharing a single
    // connection across Queue + Worker + QueueEvents can deadlock on blocking commands.
    const mk = () => new IORedis(redisUrl, { maxRetriesPerRequest: null });

    testQueue = new Queue("test:email", { connection: mk() });
    testWorker = new Worker("test:email", emailProcessor, { connection: mk() });
    testEvents = new QueueEvents("test:email", { connection: mk() });
    await testEvents.waitUntilReady();
  });

  afterAll(async () => {
    await testWorker.close();
    await testEvents.close();
    await testQueue.close();
    await redisContainer.stop();
  });

  it("processes email job and returns messageId", async () => {
    const job = await testQueue.add("send", {
      to: "test@example.com",
      templateId: "welcome",
      variables: {},
    });

    const result = await job.waitUntilFinished(testEvents, 5000);
    expect(result.messageId).toBeDefined();
  });

  it("exhausts retries and lands the job in failed", async () => {
    const failingQueue = new Queue("test:fail", {
      connection: new IORedis(redisUrl, { maxRetriesPerRequest: null }),
    });
    const failingWorker = new Worker(
      "test:fail",
      async () => {
        throw new Error("Simulated failure");
      },
      { connection: new IORedis(redisUrl, { maxRetriesPerRequest: null }) },
    );
    const failingEvents = new QueueEvents("test:fail", {
      connection: new IORedis(redisUrl, { maxRetriesPerRequest: null }),
    });
    await failingEvents.waitUntilReady();

    const job = await failingQueue.add("send", validData, {
      attempts: 2,
      backoff: { type: "fixed", delay: 50 },
    });
    await job.waitUntilFinished(failingEvents, 10_000).catch(() => {}); // expect failure

    const failedJobs = await failingQueue.getFailed();
    expect(failedJobs.some((j) => j.id === job.id)).toBe(true);

    await failingWorker.close();
    await failingEvents.close();
    await failingQueue.close();
  });
});
```
