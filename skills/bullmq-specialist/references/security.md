# Security & Compliance for BullMQ

Job queues are an often-overlooked attack surface. Jobs persist in Redis,
appear in Bull Board logs, and flow through worker process memory.

---

## The Core Rule

**Never store raw PII, secrets, or tokens in job payloads.**

Redis is:

- Not encrypted at rest by default
- Visible in Bull Board to anyone with access
- Logged by many APM/monitoring tools
- Included in Redis backups

```ts
// ❌ WRONG — PII and secrets in payload
await emailQueue.add('send', {
  to: 'user@example.com',          // PII
  name: 'Jane Doe',                // PII
  stripeToken: 'tok_abc123',       // SECRET
  ssn: '123-45-6789',              // SENSITIVE PII
});

// ✅ CORRECT — payload contains only reference IDs
await emailQueue.add('send', {
  userId: 'usr_abc123',            // internal ID only
  templateId: 'welcome-email',
  // worker fetches user data from DB at execution time
});

// Worker fetches PII at runtime — never stored in Redis
const emailWorker = new Worker('email:send', async (job) => {
  const user = await db.user.findUnique({ where: { id: job.data.userId } });
  // user.email, user.name — live in DB, not Redis
  await sendEmail({ to: user.email, name: user.name, ... });
});
```

---

## Payload Encryption (When IDs Aren't Enough)

When job context itself is sensitive (e.g., medical record processing):

> **Production note — envelope encryption with KMS**: A single static `JOB_ENCRYPTION_KEY` in
> an env var is the minimum bar. For SOC2/HIPAA/PCI scope, use envelope encryption: a per-job
> data key (DEK) is generated, used to encrypt the payload, and itself encrypted by a Key
> Encryption Key (KEK) held in AWS KMS / GCP KMS / HashiCorp Vault. Store the encrypted DEK
> alongside the ciphertext in `job.data`. Benefits: KEK rotation does not require re-encrypting
> every queue, KMS audit logs record every decrypt, and the raw key never lives in app memory.
> Libraries: `@aws-sdk/client-kms` + `@aws-crypto/client-node`, GCP `@google-cloud/kms`,
> `node-vault`. Always include a `kid` (key-id) field with the ciphertext so old jobs continue
> to decrypt after rotation.

```ts
import { createCipheriv, createDecipheriv, randomBytes } from "crypto";

const ENCRYPTION_KEY = Buffer.from(process.env.JOB_ENCRYPTION_KEY!, "hex"); // 32 bytes
const ALGORITHM = "aes-256-gcm";

export const encryptPayload = (data: unknown): string => {
  const iv = randomBytes(12);
  const cipher = createCipheriv(ALGORITHM, ENCRYPTION_KEY, iv);
  const json = JSON.stringify(data);
  const encrypted = Buffer.concat([
    cipher.update(json, "utf8"),
    cipher.final(),
  ]);
  const tag = cipher.getAuthTag();
  // iv + tag + ciphertext — base64 encoded
  return Buffer.concat([iv, tag, encrypted]).toString("base64");
};

export const decryptPayload = <T>(encrypted: string): T => {
  const buf = Buffer.from(encrypted, "base64");
  const iv = buf.subarray(0, 12);
  const tag = buf.subarray(12, 28);
  const ciphertext = buf.subarray(28);
  const decipher = createDecipheriv(ALGORITHM, ENCRYPTION_KEY, iv);
  decipher.setAuthTag(tag);
  const decrypted = Buffer.concat([
    decipher.update(ciphertext),
    decipher.final(),
  ]);
  return JSON.parse(decrypted.toString("utf8")) as T;
};

// Usage — encrypt before enqueue, decrypt inside processor
await sensitiveQueue.add("process", {
  payload: encryptPayload({ patientId: "p_123", action: "generate-report" }),
});

const sensitiveWorker = new Worker("sensitive", async (job) => {
  const { patientId, action } = decryptPayload<SensitiveData>(job.data.payload);
  // process...
});
```

---

## Audit Logging

For compliance (SOC2, HIPAA, GDPR) — log job lifecycle events to an append-only store.

**Critical:** wire audit writes from the **worker** (process-local), not from `QueueEvents`. `QueueEvents` fans out to every replica — a 5-pod deployment using `QueueEvents.on('active')` writes 5 audit rows per real start, breaking non-repudiation (auditors can no longer tell which event was "the" start). Worker events fire once on the replica that actually owned the job.

If audit writes must come from outside the worker process (e.g., the auditor service is a separate deployment), use a deterministic audit-row key (`audit:<jobId>:<event>:<attemptsMade>`) and rely on the audit-store's unique constraint to dedupe — every replica writes, only one row lands.

```ts
interface AuditEvent {
  jobId: string;
  queueName: string;
  jobName: string;
  event: "queued" | "started" | "completed" | "failed" | "dlq";
  attemptsMade: number;
  tenantId?: string;
  actorId?: string;
  timestamp: string;
  durationMs?: number;
}

// ✅ Process-local — fires once on the replica that owned the job.
emailWorker.on("active", async (job) => {
  await auditLog.append({
    auditKey: `audit:${job.id}:started:${job.attemptsMade}`, // unique-constraint dedupe
    jobId: job.id!,
    queueName: "email:send",
    jobName: job.name,
    event: "started",
    attemptsMade: job.attemptsMade,
    tenantId: (job.data as any).tenantId,
    timestamp: new Date().toISOString(),
  });
});

emailWorker.on("completed", async (job) => {
  await auditLog.append({
    auditKey: `audit:${job.id}:completed:${job.attemptsMade}`,
    jobId: job.id!,
    queueName: "email:send",
    jobName: job.name,
    event: "completed",
    attemptsMade: job.attemptsMade,
    timestamp: new Date().toISOString(),
  });
});
```

> Audit writes are still **at-least-once**: the worker can crash after writing the row but
> before BullMQ records completion, triggering a retry that re-runs the processor and (if you
> log `started` from `worker.on('active')`) re-logs the start. The unique `auditKey` keyed on
> `(jobId, event, attemptsMade)` collapses duplicates to a single row — the attempts counter
> guarantees that legitimate retries each get their own audit row.

---

## GDPR — Right to Erasure

Two failure modes plague naive GDPR erasure of queue data:

1. **OOM via unbounded fetch.** `queue.getWaiting()` (and friends) without a range fetches **every job in that state**. On a backlog of 500k waiting jobs this loads ~500MB of Job objects into Node memory and OOMs the eraser pod. Always paginate.
2. **Active and locked jobs.** `job.remove()` throws on jobs in the `active` state (they hold a lock owned by a worker token). You must either wait for the active job to finish naturally and remove it from `completed`/`failed` after, or cancel/abort the in-flight processing first.

```ts
const STATES = [
  "waiting",
  "prioritized",
  "delayed",
  "failed",
  "completed",
  "waiting-children",
] as const;
const PAGE = 1000;

export const eraseUserJobData = async (userId: string) => {
  const queues = [emailQueue, notificationQueue, invoiceQueue];

  for (const queue of queues) {
    for (const state of STATES) {
      let start = 0;
      // Paginate — never load entire queue states into memory.
      // getJobs(state, start, end, asc) returns a bounded slice.
      while (true) {
        const batch = await queue.getJobs(
          [state],
          start,
          start + PAGE - 1,
          true,
        );
        if (batch.length === 0) break;

        for (const job of batch) {
          if ((job.data as any)?.userId !== userId) continue;
          try {
            await job.remove();
          } catch (err: any) {
            // 'active' jobs are locked — they cannot be removed mid-flight.
            // Strategies:
            //  1. Mark the user for tombstoned deletion; re-run erasure after
            //     active jobs finish (cron job that drains the deletion queue).
            //  2. Signal the worker to abort the in-flight job (AbortController
            //     channel keyed by userId), then retry remove().
            if (/locked|active/i.test(err.message)) {
              await db.gdprDeletionTombstone.create({
                data: { userId, jobId: job.id!, queue: queue.name },
              });
            } else {
              throw err;
            }
          }
        }
        if (batch.length < PAGE) break;
        start += PAGE;
      }
    }
  }

  // ⚠️ Backup retention caveat: removing the job from Redis does NOT remove it
  // from RDB snapshots, AOF files, or off-site backups taken before erasure.
  // GDPR Art. 17 requires erasure from backups too — either rotate backups out
  // within your published retention window (typically 30–90 days, documented in
  // your DPA), or implement crypto-shredding (encrypt every payload with a
  // per-user key, erase the key on deletion, leaving ciphertext-only backups
  // unreadable without separate cooperation).
};
```

> **`removeOnComplete` retention is not GDPR-sufficient.** A queue with low throughput retains
> completed jobs past your retention SLA because trimming is per-event, not time-driven. Run a
> daily reaper that explicitly removes completed/failed jobs older than the retention window.

---

## Multi-Tenant Security Isolation

```ts
// Validate tenantId at queue boundary — never trust job.data.tenantId blindly
const emailWorker = new Worker<EmailJobData>("email:send", async (job) => {
  const { tenantId, userId } = job.data;

  // Re-verify tenant is active before processing
  const tenant = await db.tenant.findUnique({ where: { id: tenantId } });
  if (!tenant || tenant.status !== "active") {
    throw new UnrecoverableError(`Tenant ${tenantId} not found or suspended`);
  }

  // Verify userId belongs to tenantId — prevent cross-tenant data access
  const user = await db.user.findFirst({ where: { id: userId, tenantId } });
  if (!user)
    throw new UnrecoverableError("Cross-tenant access attempt blocked");

  // safe to proceed
});
```

---

## Bull Board Security Checklist

```
□ Bull Board behind authentication (SSO preferred; basic auth as absolute minimum)
□ Bull Board NOT exposed on public-facing port (no Ingress to 0.0.0.0)
□ Bull Board in readOnlyMode: true (no retry/remove/promote buttons in production UI)
□ Job data fields sanitized in Bull Board display — implement a custom formatter that
  redacts any field matching /token|secret|key|password|ssn|email|phone/i
□ Bull Board access logged to audit trail with the actor's identity (not just IP)
□ Production Bull Board behind VPN, mTLS, or zero-trust gateway (Cloudflare Access /
  Tailscale / IAP); never just "internal network"
□ Retention policy publicly documented if Bull Board is accessible to support staff
  who do not normally have data access (Bull Board exposes raw job.data)
□ Separate Bull Board instances per environment — staging Bull Board credentials must
  NOT decrypt production payloads
```

> **Bull Board is an audit-scope exposure.** Anyone who can reach Bull Board can read every
> job's raw `data`. If your job data passes the "IDs only" rule from the top of this file, the
> exposure is limited to "what work happens to whom". If you store encrypted payloads, Bull
> Board sees ciphertext — preferred. Treat Bull Board access at the same tier as production
> database read access in your access review.
