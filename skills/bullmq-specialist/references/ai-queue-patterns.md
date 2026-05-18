# AI/LLM Queue Patterns

LLM and embedding workloads have unique characteristics: variable latency (1s–120s),
high token cost, provider rate limits, and GPU resource constraints.
Standard queue patterns need adjustment for these realities.

---

## LLM Inference Queue

```ts
interface InferenceJobData {
  _v: 1;
  requestId: string; // for deduplication + tracing — use as jobId
  tenantId: string;
  model: "claude-sonnet-4-7" | "claude-haiku-4-5" | "gpt-4o" | "gemini-pro";
  promptTokens: number; // estimated — for budget tracking
  systemPromptRef?: string; // reference to prompt template in DB (preferred)
  userMessageRef?: string; // reference to user input in DB (preferred for long inputs / PII)
  systemPromptInline?: string; // only for short, non-sensitive prompts
  userMessageInline?: string;
  maxTokens: number;
  callbackUrl?: string; // webhook on completion for async flows — see "Webhook delivery" below
}

interface InferenceJobResult {
  requestId: string;
  outputText: string;
  totalTokens: number;
  latencyMs: number;
  model: string;
}

export const inferenceQueue = new Queue<InferenceJobData, InferenceJobResult>(
  "ai:inference",
  {
    connection: queueConnection,
    defaultJobOptions: {
      attempts: 3,
      backoff: { type: "custom" }, // uses fullJitterBackoff from advanced-retry.md
      // ⚠️ BullMQ v5 does NOT have a `timeout` job option (removed from Bull v3).
      // Enforce timeouts INSIDE the processor with AbortController / Promise.race.
      removeOnComplete: { count: 500, age: 3600 }, // shorter retention — results are large
      removeOnFail: { count: 2000 },
    },
  },
);

const inferenceWorker = new Worker<InferenceJobData, InferenceJobResult>(
  "ai:inference",
  async (job) => {
    const startMs = Date.now();
    await job.updateProgress({ phase: "calling-llm", model: job.data.model });

    // Enforce a hard timeout INSIDE the processor — BullMQ v5 has no `timeout` job opt.
    const ac = new AbortController();
    const timer = setTimeout(() => ac.abort(), 120_000);
    let result;
    try {
      result = await callLLM(job.data, { signal: ac.signal });
    } finally {
      clearTimeout(timer);
    }

    await job.updateProgress({
      phase: "complete",
      tokens: result.usage.total_tokens,
    });

    // Webhook delivery is its own at-least-once problem — see "Webhook delivery" below.
    // Do NOT fire-and-forget here. Enqueue a dedicated webhook job with idempotency.
    if (job.data.callbackUrl) {
      await webhookQueue.add(
        "deliver",
        {
          url: job.data.callbackUrl,
          requestId: job.data.requestId,
          result,
        },
        {
          // Same source request → same webhook job. Prevents duplicate deliveries
          // even if the inference job retries after a worker crash post-LLM-call.
          jobId: `webhook:${job.data.requestId}`,
        },
      );
    }

    return {
      requestId: job.data.requestId,
      outputText: result.content,
      totalTokens: result.usage.total_tokens,
      latencyMs: Date.now() - startMs,
      model: job.data.model,
    };
  },
  {
    connection: workerConnection,
    concurrency: 5, // conservative — LLM calls hold connections
    lockDuration: 180_000, // 3min lock — must exceed your in-processor timeout.
    // BullMQ v5 auto-renews at lockDuration/2; there is no
    // `lockRenewTime` option (Bull v3 carryover — silently ignored).
  },
);
```

> **Why no `timeout` option?** BullMQ v5 removed the `timeout` job option. Setting it has no
> effect — the job will run until the processor returns or throws. If you rely on the option
> for safety, you will have hung jobs holding locks for `lockDuration` (which auto-renews) —
> effectively forever. Always enforce timeouts with `AbortController` in the processor.

---

## Embedding Pipeline

Embeddings are typically fast (< 2s) but high-volume. Batch them.

```ts
interface EmbedJobData {
  _v: 1;
  documentId: string;
  tenantId: string;
  chunkIds: string[]; // embed multiple chunks in one job — reduces Redis overhead
  model: "text-embedding-3-small" | "text-embedding-3-large";
}

export const embedQueue = new Queue<EmbedJobData>("ai:embed", {
  connection: queueConnection,
  defaultJobOptions: {
    attempts: 5,
    backoff: { type: "exponential", delay: 1000 },
    // No `timeout` option in v5 — enforce via AbortController in the processor below.
    removeOnComplete: { count: 1000 },
  },
});

const embedWorker = new Worker<EmbedJobData>(
  "ai:embed",
  async (job) => {
    const chunks = await db.chunk.findMany({
      where: { id: { in: job.data.chunkIds } },
    });

    // Batch API call — embed all chunks in one request
    const embeddings = await openai.embeddings.create({
      model: job.data.model,
      input: chunks.map((c) => c.text),
    });

    // Upsert vectors to your vector store (Pinecone, pgvector, Qdrant)
    await vectorStore.upsert(
      embeddings.data.map((e, i) => ({
        id: chunks[i].id,
        vector: e.embedding,
        metadata: {
          documentId: job.data.documentId,
          tenantId: job.data.tenantId,
        },
      })),
    );
  },
  {
    connection: workerConnection,
    concurrency: 10, // embedding API is faster — can run more concurrently
  },
);
```

---

## Token Budget Tracking

Track token spend across jobs without blocking the processor — but **never** wire the increment to `QueueEvents.on('completed')`. That handler fires in every replica, so a 5-pod deployment would increment the billing counter 5× per real completion.

```ts
// ✅ Enqueue the token-usage write from the WORKER (process-local).
// Use a deterministic jobId so a retried inference job (which already counted tokens)
// does not double-bill: same source job → same tracking job → BullMQ collapses to one.
async function inferenceWorkerProcessor(job: Job<InferenceJobData>) {
  const result = await callLLM(job.data);

  await tokenQueue.add(
    "track",
    {
      tenantId: job.data.tenantId,
      tokens: result.usage.total_tokens,
      model: job.data.model,
      sourceJobId: job.id!,
    },
    {
      jobId: `tokens:${job.data.requestId}`, // idempotent across retries + replicas
    },
  );

  return result;
}

// The tracking worker itself is at-least-once — increments MUST be idempotent.
// Either (a) use upsert keyed on (tenantId, requestId) to insert one row per request
// and SUM at query time, or (b) use a Redis HINCRBY guarded by SETNX(requestId).
// A naive `increment` on a monthly counter is NOT safe — at-least-once delivery to
// the worker means a single requestId can be applied twice on retry.

const tokenWorker = new Worker(
  "ai:token-tracking",
  async (job) => {
    // Idempotency: insert a row keyed on (tenantId, requestId). Duplicate inserts
    // are no-ops via the unique constraint. Bill by SUM(tokens) at query time.
    await db.tokenUsageEvent.upsert({
      where: {
        tenantId_requestId: {
          tenantId: job.data.tenantId,
          requestId: job.data.sourceJobId,
        },
      },
      create: {
        tenantId: job.data.tenantId,
        requestId: job.data.sourceJobId,
        tokens: job.data.tokens,
        model: job.data.model,
        month: getMonth(),
      },
      update: {}, // no-op on duplicate
    });
  },
  { connection: workerConnection, concurrency: 20 },
);
```

> **Why not `upsert ... { increment }` on a monthly counter?** Because at-least-once delivery
> means a single source request can apply twice (worker crashed after the DB write but before
> ACKing the job; BullMQ redelivers; counter goes up again). The only safe primitive is an
> idempotent insert keyed on the source request ID. Aggregate at read time, not write time.

---

## GPU Job Routing

When you have heterogeneous GPU resources (e.g., A100 for large models, T4 for small):

```ts
// Route jobs to queues that target specific GPU node pools
const MODEL_QUEUE_MAP: Record<string, string> = {
  "claude-sonnet-4-7": "ai:inference:api", // external API — no GPU
  "claude-haiku-4-5": "ai:inference:api", // external API — no GPU
  "llama-70b": "ai:inference:gpu-a100", // large local model
  "llama-7b": "ai:inference:gpu-t4", // small local model
};

export const enqueueInference = async (data: InferenceJobData) => {
  const queueName = MODEL_QUEUE_MAP[data.model] ?? "ai:inference:api";
  const queue = getOrCreateQueue(queueName);

  return queue.add("infer", data, {
    jobId: `infer:${data.requestId}`, // deduplication
    priority: data.tenantId === "pro-tenant" ? 1 : 5, // pro tenants get priority
  });
};
```

```yaml
# K8s: separate worker deployments per GPU node pool
# ai-worker-a100-deployment.yaml
spec:
  nodeSelector:
    gpu-type: a100
  containers:
    - name: ai-worker
      env:
        - name: QUEUE_NAME
          value: "ai:inference:gpu-a100"
```

---

## Webhook Delivery (Idempotent, At-Least-Once)

Webhook delivery is at-least-once by construction (network failures cannot distinguish "delivered but no ack" from "not delivered"). Build the receiver and sender accordingly.

```ts
// Sender — separate queue, persistent delivery-state row in your DB.
interface WebhookJobData {
  url: string;
  requestId: string; // source request — caller's idempotency key
  result: unknown;
}

const webhookWorker = new Worker<WebhookJobData>(
  "ai:webhook",
  async (job) => {
    // Check delivery state BEFORE re-sending — protects against retries after a
    // network success that we never observed (e.g., worker crashed mid-fetch).
    const state = await db.webhookDelivery.findUnique({
      where: { requestId: job.data.requestId },
    });
    if (state?.deliveredAt) return { skipped: true };

    const ac = new AbortController();
    const timer = setTimeout(() => ac.abort(), 10_000);
    try {
      const res = await fetch(job.data.url, {
        method: "POST",
        headers: {
          "content-type": "application/json",
          // Send the idempotency key — receivers MUST dedupe on this.
          "idempotency-key": job.data.requestId,
          "x-bullmq-attempt": String(job.attemptsMade),
        },
        body: JSON.stringify({
          requestId: job.data.requestId,
          result: job.data.result,
        }),
        signal: ac.signal,
      });
      if (res.status >= 500) throw new Error(`Webhook ${res.status}`); // retry
      if (res.status === 429) {
        const retry = parseInt(res.headers.get("retry-after") ?? "30", 10);
        await (webhookWorker as Worker).rateLimit(retry * 1000);
        throw new Worker.RateLimitError();
      }
      if (res.status >= 400) {
        // 4xx (other than 429) — receiver rejected; do not retry forever
        throw new UnrecoverableError(`Webhook ${res.status}`);
      }
      await db.webhookDelivery.upsert({
        where: { requestId: job.data.requestId },
        create: { requestId: job.data.requestId, deliveredAt: new Date() },
        update: { deliveredAt: new Date() },
      });
    } finally {
      clearTimeout(timer);
    }
  },
  {
    connection: workerConnection,
    concurrency: 50,
  },
);
```

> **Receiver contract**: Document that your webhooks may deliver the same `requestId` more than
> once and that receivers MUST dedupe by the `Idempotency-Key` header. Many production outages
> trace to assumed-once webhook delivery.

---

## LLM Retry Semantics

```ts
async function llmProcessor(this: Worker, job: Job<InferenceJobData>) {
  try {
    return await callProvider(job.data);
  } catch (err: any) {
    // Anthropic / OpenAI error codes
    if ([400, 422].includes(err.status)) {
      throw new UnrecoverableError(`Bad request: ${err.message}`); // prompt issue
    }
    if (err.status === 401) {
      throw new UnrecoverableError("Invalid API key"); // will never self-heal
    }
    if (err.status === 429) {
      const delay = parseInt(err.headers?.["retry-after"] ?? "10", 10) * 1000;
      // Use the rate-limit primitive so this 429 doesn't consume an attempt.
      // `this` is the Worker (BullMQ binds it when calling the processor).
      await (this as Worker).rateLimit(delay);
      throw new Worker.RateLimitError();
    }
    if (err.status >= 500) {
      throw err; // provider outage — BullMQ retries with backoff
    }
    throw err;
  }
}
```
