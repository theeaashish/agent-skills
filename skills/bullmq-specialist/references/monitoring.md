# Monitoring and Observability

---

## Bull Board (Development + Internal)

```ts
import { createBullBoard } from '@bull-board/api';
import { BullMQAdapter } from '@bull-board/api/bullMQAdapter';
import { ExpressAdapter } from '@bull-board/express';
import basicAuth from 'express-basic-auth';

const serverAdapter = new ExpressAdapter();
serverAdapter.setBasePath('/internal/queues');

createBullBoard({
  queues: [
    new BullMQAdapter(emailQueue, { readOnlyMode: false }),
    new BullMQAdapter(dlqQueue, { readOnlyMode: true }), // DLQ is view-only
  ],
  serverAdapter,
  options: {
    uiBasePath: '/internal/queues',
    miscLinks: [{ text: 'Back to App', url: '/' }],
  },
});

// Always protect Bull Board — never expose publicly
app.use(
  '/internal/queues',
  basicAuth({ users: { admin: process.env.BULL_BOARD_PASSWORD! }, challenge: true }),
  serverAdapter.getRouter(),
);
```

---

## QueueEvents — Full Observability Hook

> **Connection requirement**: `QueueEvents` must use its **own dedicated IORedis instance**
> — never share with a Queue or Worker. BullMQ internally blocks on Redis streams inside
> QueueEvents; sharing a connection causes missed or disrupted event processing.

> **Fan-out warning**: QueueEvents is not a consumer group. Every listener receives
> every event. Do not increment billing counters, append audit rows, send alerts, or
> create DLQ jobs from QueueEvents in every worker pod unless the side effect is
> idempotent. Prefer one dedicated event-service deployment for those actions.

> **`stalled` is an event, not a job state**: BullMQ emits `stalled` when its internal
> checker detects that an `active` job has not renewed its lock within `stalledInterval`.
> The job is then moved back to `waiting` (or to `failed` after `maxStalledCount` stalls).
> You cannot query "stalled jobs" from the queue — only listen to the event as it fires.

```ts
import { QueueEvents } from 'bullmq';

// Use eventsConnection (dedicated IORedis instance — never share with Queue or Worker)
const events = new QueueEvents('emails', { connection: eventsConnection });

events.on('waiting',   ({ jobId })              => metrics.increment('queue.waiting',   { queue: 'emails' }));
events.on('active',    ({ jobId, prev })         => metrics.increment('queue.active',    { queue: 'emails' }));
events.on('completed', ({ jobId, returnvalue })  => metrics.increment('queue.completed', { queue: 'emails' }));
events.on('failed',    ({ jobId, failedReason }) => {
  metrics.increment('queue.failed', { queue: 'emails' });
  logger.error({ jobId, reason: failedReason }, 'Job failed');
});
events.on('stalled', ({ jobId }) => {
  // Fired by BullMQ's internal stall checker — indicates worker crashed or
  // lost its lock. The job has already been re-queued or moved to failed.
  // Alert from a singleton event-service, or dedupe alerts by queue+jobId+event.
  metrics.increment('queue.stalled', { queue: 'emails' });
  alerting.triggerOnce(`emails-${jobId}-stalled`, `Stall event for job ${jobId}`);
});
events.on('delayed',   ({ jobId, delay })        => metrics.increment('queue.delayed',   { queue: 'emails' }));
```

---

## Prometheus Metrics Pattern

```ts
import { Queue } from 'bullmq';
import { Gauge } from 'prom-client';

const queueDepthGauge = new Gauge({
  name: 'bullmq_queue_depth',
  help: 'Number of waiting jobs per queue',
  labelNames: ['queue', 'state'],
});

// Scrape queue depths periodically — not per-job (too expensive)
const scrapeQueues = async (queues: Queue[]) => {
  for (const queue of queues) {
    const counts = await queue.getJobCounts(
      'waiting', 'active', 'delayed', 'failed', 'completed',
    );

    for (const [state, count] of Object.entries(counts)) {
      queueDepthGauge.set({ queue: queue.name, state }, count);
    }
  }
};

// Register scrape interval — every 15s matches Prometheus default scrape
setInterval(() => scrapeQueues([emailQueue, dlqQueue]), 15_000);
```

---

## Key Metrics to Alert On

| Metric | Alert Threshold | Meaning |
|---|---|---|
| `queue.failed` rate | > 5% of `queue.completed` | Systemic failure — check downstream |
| `queue.stalled` count | Any non-zero | Worker crashed mid-job |
| `queue.waiting` depth | > SLA-based threshold | Processing falling behind |
| `queue.delayed` depth | Sudden spike | Rate limiter or downstream rejection |
| Worker process uptime | Restart loop | Processor crash — check logs |

---

## Sentry Integration

```ts
import * as Sentry from '@sentry/node';

worker.on('failed', (job, err) => {
  Sentry.withScope((scope) => {
    scope.setTag('queue', 'emails');
    scope.setContext('job', {
      id: job?.id,
      name: job?.name,
      data: job?.data,
      attemptsMade: job?.attemptsMade,
    });
    Sentry.captureException(err);
  });
});
```
