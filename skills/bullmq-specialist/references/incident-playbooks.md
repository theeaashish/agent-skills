# Production Incident Playbooks

Runbooks for the most common BullMQ production incidents.
Bookmark this file — you'll want it at 3am.

---

## Playbook 1: DLQ Flood

**Symptoms**: DLQ growing rapidly, alert firing on `dead-letter` queue depth.

```ts
// 1. Stop the bleeding — pause via BullMQ API, never raw Redis commands
//    queue.pause() sets the BullMQ pause key atomically and drains safely
await emailQueue.pause();
// Worker.pause() stops the worker from picking up new jobs locally
// queue.pause() stops ALL workers on that queue across every replica

// 2. Diagnose — sample a DLQ job to find root cause
const [sample] = await dlqQueue.getJobs(['waiting'], 0, 0);
console.log(sample?.data.failureReason, sample?.data.originalData);
// Check failureReason — is it a schema mismatch? Downstream outage? Bad data?

// 3. Check if it's a deployment issue (new code rejecting old job shapes)
// → Verify _v versioning compat branch exists (see Job Schema Versioning in SKILL.md)

// 4. Resume once root cause is fixed
await emailQueue.resume();
```

```ts
// 5. Bulk retry from DLQ — move jobs back to source queue
const sourceQueuesByName: Record<string, Queue> = {
  'email:send': emailQueue,
};

const retryDLQ = async (sourceQueueName: string) => {
  const sourceQueue = sourceQueuesByName[sourceQueueName];
  if (!sourceQueue) throw new Error(`Unknown source queue: ${sourceQueueName}`);

  const dlqJobs = await dlqQueue.getJobs(['waiting', 'failed'], 0, 100); // paginate for large DLQs

  for (const dlqJob of dlqJobs) {
    if (dlqJob.data.originalQueue === sourceQueueName) {
      // Re-enqueue to original queue with fresh attempts
      await sourceQueue.add(
        dlqJob.data.originalJobName,
        dlqJob.data.originalData,
        { attempts: 3, backoff: { type: 'exponential', delay: 2000 } },
      );
      await dlqJob.remove(); // clean up DLQ entry
    }
  }
};
```

---

## Playbook 2: Retry Storm

**Symptoms**: Downstream service getting hammered, your logs full of retry errors,
queue depth not shrinking despite workers running.

```bash
# 1. Immediately pause all workers touching the failing downstream
# In application/admin shell:
# await emailQueue.pause();          # all workers for this queue
# await notificationQueue.pause();

# 2. Check backoff config — are retries immediate?
# Check queue's defaultJobOptions.backoff — should be exponential, not fixed

# 3. Check if circuit breaker is implemented (see advanced-retry.md)
# If not: manually drain the retry queue

# 4. Identify root cause — is downstream actually down?
curl -I https://api.sendgrid.com/v3/mail/send   # check provider status

# 5. Resume with rate limiting after downstream recovers
await emailWorker.resume();
```

```ts
// Emergency: clear a backlog of retrying jobs that will never succeed
const clearRetryingJobs = async (queue: Queue, reason: string) => {
  const failed = await queue.getFailed(0, 1000);
  const retrying = failed.filter(j => j.attemptsMade < (j.opts.attempts ?? 3));

  console.log(`Clearing ${retrying.length} retrying jobs: ${reason}`);
  await Promise.all(retrying.map(j => j.discard())); // moves to failed, stops retry
};
```

---

## Playbook 3: Redis Outage

**Symptoms**: Workers disconnecting, `ECONNREFUSED` errors, all queues stopped.

```
Immediate response:
1. Check Redis health: redis-cli -h $REDIS_HOST PING
2. Check Redis memory: redis-cli INFO memory | grep used_memory_human
3. If Sentinel: check which node is primary: redis-cli -h $SENTINEL_HOST -p 26379 SENTINEL masters
4. BullMQ workers reconnect automatically via ioredis retryStrategy
   → Jobs in 'active' state will become 'stalled' after lockDuration
   → Stalled jobs auto-retry after stalledInterval × maxStalledCount

// Recovery:
5. After Redis recovers, workers reconnect — no manual intervention needed
6. Monitor stalls via QueueEvents listeners and metrics (see monitoring.md)
   → Stalled jobs are moved back to 'waiting' or 'failed' by the checker
7. If jobs remain active beyond lockDuration + stalledInterval:
   → check that at least one worker has stalled checks enabled
   → restart a stuck worker pod so another worker can recover the lock-expired jobs
   → do not use raw Redis edits; `queue.obliterate({ force: true })` destroys the queue
```

---

## Playbook 4: Stuck Workers (Jobs Active Forever)

**Symptoms**: Jobs stuck in `active` state, not completing, not failing.

```bash
# 1. Check if worker process is alive
ps aux | grep "node.*worker"
# Or: kubectl get pods -l app=email-worker

# 2. Check active jobs (BullMQ active jobs are tracked in Redis structures)
# Check the active list size: LLEN bull:email:send:active

# 3. Check processor logs for the specific jobId

# 4. If processor is deadlocked (no logs):
# → Worker process may be stuck in sync code or infinite loop
# → Kill the worker pod — BullMQ stall detection will recover the jobs
kubectl delete pod email-worker-xxxx
# Stall detection fires after stalledInterval — job moves to stalled → failed → DLQ
```

```ts
// Inspect stuck active jobs. Do not force-move active jobs to failed unless you
// own the lock token; killing/restarting the stuck worker and letting stall
// detection recover is safer.
const inspectStuckActiveJobs = async (queue: Queue, maxAgeMs: number) => {
  let cursor = 0;
  const pageSize = 100;
  const now = Date.now();

  while (true) {
    const active = await queue.getJobs(['active'], cursor, cursor + pageSize - 1);
    if (active.length === 0) break;

    for (const job of active) {
      const ageMs = now - job.processedOn!;
      if (ageMs > maxAgeMs) {
        console.warn(`Stuck active job ${job.id} (age: ${ageMs}ms)`);
      }
    }

    if (active.length < pageSize) break;
    cursor += pageSize;
  }
};
```

---

## Playbook 5: Memory Exhaustion

**Symptoms**: Redis OOM errors, `maxmemory` limit hit, jobs failing to enqueue.

```bash
# 1. Check memory immediately
redis-cli INFO memory | grep -E "used_memory_human|maxmemory_human|mem_fragmentation_ratio"

# 2. Find memory hogs
redis-cli --bigkeys                          # find largest keys
redis-cli MEMORY USAGE bull:emails:completed # specific queue key size

# 3. Emergency: trim completed job retention
```

```ts
// Emergency retention trim — reduce completed job count immediately
await emailQueue.clean(
  0,           // grace: 0ms — remove all completed
  1000,        // limit: remove max 1000 per call (paginate)
  'completed',
);
// Repeat until memory pressure drops

// Trim failed jobs older than 3 days
const threeDaysAgo = 3 * 24 * 3600 * 1000;
await emailQueue.clean(threeDaysAgo, 1000, 'failed');
```

---

## Playbook 6: When to Move from BullMQ to Temporal

Use this heuristic to decide when BullMQ has hit its limits:

| Scenario | BullMQ | Temporal |
|---|---|---|
| Job duration < 30min | ✅ | Overkill |
| Retry + backoff needed | ✅ | |
| DAG job dependencies (FlowProducer) | ✅ | |
| Jobs run for hours/days | ⚠️ Hard to manage | ✅ |
| Sagas with compensation rollback | ❌ Build yourself | ✅ Native |
| Human-in-the-loop approval steps | ❌ Complex | ✅ Native |
| Exactly-once guarantees required | ❌ Best-effort | ✅ |
| Complex version migration of in-flight workflows | ❌ Very hard | ✅ Native versioning |

**Decision rule**: If you're writing saga compensation logic, implementing custom
"workflow state machine" on top of BullMQ, or jobs run longer than 30 minutes
with complex branching — evaluate Temporal. The BullMQ → Temporal migration cost
is real, but fighting BullMQ for workflow orchestration costs more long-term.
