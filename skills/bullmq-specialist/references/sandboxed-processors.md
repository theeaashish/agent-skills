# Sandboxed Processors

Run job processors in a separate child process. Use when:

- Processor is CPU-intensive (image resize, PDF generation, ML inference)
- Processor has memory leaks — child process restarts clean each job
- Processor crash should not kill the main worker process
- You need true parallelism beyond Node's event loop

---

## Setup

Sandboxed processors require the processor to be in a **separate file** — not inline.

```ts
// workers/email.worker.ts — main worker file
import { Worker } from "bullmq";
import path from "path";

export const emailWorker = new Worker(
  "emails",
  path.resolve(__dirname, "./processors/email.processor.js"), // compiled output path
  {
    connection: workerConnection,
    concurrency: 5,
    // Each concurrent job gets its own child process slot
  },
);
```

```ts
// workers/processors/email.processor.ts — the sandboxed processor
import { Job } from "bullmq";

// Default export — required for sandboxed processors
export default async function (
  job: Job<EmailJobData>,
): Promise<EmailJobResult> {
  await job.updateProgress(10);
  const result = await sendEmail(job.data);
  await job.updateProgress(100);
  return { messageId: result.id, sentAt: new Date().toISOString() };
}
```

> **Build requirement**: The processor file must be pre-compiled to `.js` before
> the Worker references it. Add a pre-build step or use `ts-node` with
> `--esm` carefully — sandboxed processors fork Node directly.

---

## Concurrency Behaviour

```
concurrency: 5 → up to 5 sandboxed processes running in parallel
Each process handles one job at a time (no shared state)
Memory/CPU isolated per process — crash in one doesn't affect others
```

> **IPC overhead is real.** Every job's `data`, `progress`, `updateData`, `log`, and the final
> return value cross the parent ↔ child process boundary via structured-clone IPC. For jobs with
> large `data` payloads (>100KB) this serialisation cost can dominate processor time. Pass
> reference IDs (the cardinal rule of BullMQ payload design) — sandboxed processors amplify the
> penalty for breaking it.

> **Lock ownership across the boundary**: the lock token lives in the parent Worker process,
> not the child. If the parent process is SIGKILLed, the child is orphaned and its lock will
> expire via stall detection. Set `lockDuration` long enough to cover the slowest expected job;
> do not assume the child can renew the lock itself.

---

## When NOT to Use Sandboxed Processors

- Simple I/O-bound jobs (email sending, DB writes) — overhead not worth it
- Jobs that share in-memory caches — each child gets its own memory space
- Short jobs (< 100ms) — process spawn overhead dominates
