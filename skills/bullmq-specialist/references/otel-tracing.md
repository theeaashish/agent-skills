# OpenTelemetry & Distributed Tracing

Tracing a job from HTTP request → queue → worker → downstream is essential
in microservices. Without it, you're flying blind when debugging latency spikes.

---

## The Problem

```
HTTP POST /checkout
  └─ app adds job to queue ──── trace ends here (most teams stop)
       └─ worker picks up job
            └─ calls Stripe
            └─ writes to DB
            └─ calls notification service
```

Without trace propagation, you can't correlate a slow checkout with a slow worker.

---

## Setup: OTel with BullMQ

```ts
import { NodeSDK } from "@opentelemetry/sdk-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { Resource } from "@opentelemetry/resources";
import {
  trace,
  context,
  propagation,
  SpanStatusCode,
} from "@opentelemetry/api";
import { W3CTraceContextPropagator } from "@opentelemetry/core";

// Initialise once at app entrypoint — before any imports
const sdk = new NodeSDK({
  resource: new Resource({ "service.name": "email-worker" }),
  traceExporter: new OTLPTraceExporter({ url: process.env.OTEL_EXPORTER_URL }),
  textMapPropagator: new W3CTraceContextPropagator(),
});
sdk.start();
```

---

## Propagating Trace Context Into Job Data

```ts
// Producer (API handler) — inject trace context into job data
app.post("/checkout", async (req, res) => {
  const tracer = trace.getTracer("checkout-api");

  await tracer.startActiveSpan("checkout.enqueue", async (span) => {
    // Extract current trace context as a plain object — safe for Redis storage
    const traceContext: Record<string, string> = {};
    propagation.inject(context.active(), traceContext);

    await orderQueue.add("process", {
      orderId: req.body.orderId,
      _traceContext: traceContext, // carry trace through the queue boundary
    });

    span.setStatus({ code: SpanStatusCode.OK });
    span.end();
    res.json({ ok: true });
  });
});
```

---

## Restoring Trace Context in Worker

```ts
interface OrderJobData {
  _v: 1;
  orderId: string;
  _traceContext?: Record<string, string>; // injected by producer
}

const orderWorker = new Worker<OrderJobData>(
  "orders",
  async (job) => {
    const tracer = trace.getTracer("order-worker");

    // Restore parent trace context from job data
    const parentContext = job.data._traceContext
      ? propagation.extract(context.active(), job.data._traceContext)
      : context.active();

    // All spans inside this block are children of the original HTTP request span
    return context.with(parentContext, async () =>
      tracer.startActiveSpan("order.process", async (span) => {
        span.setAttribute("job.id", job.id ?? "");
        span.setAttribute("job.attempts", job.attemptsMade);
        span.setAttribute("order.id", job.data.orderId);

        try {
          await job.updateProgress(20);
          const payment = await chargePayment(job.data.orderId); // creates child span

          await job.updateProgress(70);
          await fulfillOrder(job.data.orderId); // creates child span

          span.setStatus({ code: SpanStatusCode.OK });
          return { charged: true, paymentId: payment.id };
        } catch (err: any) {
          span.recordException(err);
          span.setStatus({ code: SpanStatusCode.ERROR, message: err.message });
          throw err;
        } finally {
          span.end();
        }
      }),
    );
  },
  { connection: workerConnection },
);
```

---

## Correlation IDs (Simpler Alternative)

If full OTel is too heavy, use correlation IDs for basic request-to-job linkage.

```ts
// Producer
import { randomUUID } from "crypto";

app.post("/checkout", async (req, res) => {
  const correlationId =
    (req.headers["x-correlation-id"] as string) ?? randomUUID();

  await orderQueue.add("process", {
    orderId: req.body.orderId,
    correlationId, // carry through the queue
  });

  res.setHeader("x-correlation-id", correlationId);
  res.json({ ok: true });
});

// Worker — attach correlationId to all logs
const orderWorker = new Worker("orders", async (job) => {
  const log = logger.child({
    correlationId: job.data.correlationId,
    jobId: job.id,
  });
  log.info("Processing order");

  await processOrder(job.data.orderId);

  log.info("Order complete");
});
```

---

## Span Hierarchy

With trace propagation, your tracing UI shows the full picture:

```
[HTTP POST /checkout]                              100ms
  └─ [checkout.enqueue]                             5ms
       └─ [order.process] (worker span)            90ms
            ├─ [stripe.charge]                     60ms
            ├─ [db.order.update]                    5ms
            └─ [notification.send]                 20ms
```

This lets you pinpoint whether latency is in the queue wait time,
the payment call, or the DB write — without grep-ing logs across services.

---

## QueueEvents Tracing

```ts
// Add trace events for queue wait time analysis.
// Caveats:
//  1. `job.timestamp` is the enqueue time. For jobs that sat in `delayed` for an hour
//     before becoming `waiting`, `Date.now() - job.timestamp` reports an hour of "wait",
//     conflating intentional delay with backlog wait. Track delayed-derived jobs separately
//     by checking the `delay` option or recording a `becameWaitingAt` field in job.data.
//  2. QueueEvents fans out — every replica with a QueueEvents listener emits this metric
//     for every job. Use a metrics library that dedupes by (jobId, event) OR scope the
//     listener to one pod (leader election) OR accept N× cardinality and divide by replica
//     count in your dashboards. For pure observability, the simplest answer is to emit the
//     metric from worker.on('active') instead, which fires once per job pickup.
emailWorker.on("active", async (job, prev) => {
  if (prev !== "waiting") return; // skip delayed→active and waiting-children→active
  const queueWaitMs = Date.now() - job.timestamp;
  metrics.histogram("queue.wait_ms", queueWaitMs, { queue: "email:send" });
});
```
