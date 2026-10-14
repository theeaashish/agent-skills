# Lesson template

Read this before teaching Lesson 1. It defines the exact lesson layout, a complete worked example illustrating the teaching style, checkpoint design rules, and feedback patterns for mentoring freshers and juniors.

## Contents

- Lesson format
- Worked example
- Checkpoint design
- Question bank
- Feedback patterns

## Lesson format

```text
📘 Lesson N/T · `path/to/file.ext` [🔥 Core | 🧩 Supporting]   ▓▓▓░░░░░░ (9 cells)

🗺️ Where we are
   routes → controller → **[queue]** → worker → mailer

🔌 How it connects (The Wiring)
   • Incoming: Called by `fileA.ts -> funcName()` passing `{ ... }`
   • Outgoing: Calls `fileB.ts -> nextFunc()` (covered in Lesson N+1)

🎯 Purpose
   One plain sentence: what job this file does and what breaks if we delete it.

🔍 Walkthrough
   [Chunk 1: path/to/file.ext:lines]
   Plain-English explanation of what these lines do.
   • 🔤 Syntax & types unpacked: (Include whenever code uses dense TS/JS/ORM tricks)
   • 📦 Payload trace: How our Canonical Payload from Lesson 0 transforms here.
   • 🟢 Happy path: What happens when everything succeeds.
   • 🔴 If things break: What happens on bad input, null values, or service failure.

   **Function summary:** `name(inputs)` → returns · side effects · callers · failure behavior

🧠 Under the hood (Naive Way vs. This Way)
   1. **How a beginner might write it (3 lines):** Show the naive approach and why it breaks in production.
   2. **What this code actually does under the hood:** Demystify the 1 core concept that would otherwise feel like magic.

⚖️ Senior code audit
   • **Why this design:** Decision · Alternative not chosen · Trade-off
   • **Honest critique:** Point out if the AI over-engineered anything, missed an edge case, or left room for improvement (great ammo for interviews).

🧪 Try in your IDE (Optional)
   One 20-second action the learner can do right now (e.g., Cmd+Click a function, inspect a type, or run a single test).

🎤 Say it in an interview
   2–3 natural, first-person sentences the learner can use when walking an interviewer through their project.

✅ Checkpoint
   • 1 sharp question for 🧩 Supporting / standard lessons (or 2 questions for 🔥 Core lessons).
   • Stop immediately and wait for the learner's reply.
```

**Formatting rules:**

- **Skip empty sections:** If a simple config file has no tricky syntax or deep "Under the hood" concept, skip `🔤` or `🧠`. Never write filler just to fill a header.
- **Diff-Slice Mode (Company PRs):** When teaching changes inside a huge existing file, label the header `📘 Lesson N/T · path/to/file.ext (lines X–Y diff)` and only quote the modified/added functions plus the 3–5 lines where they hook in.
- **Compress 🧩 Supporting lessons:** Keep supporting files (constants, basic types, simple configs) short: `🔌 How it connects` + `🎯 Purpose` + `🔍 Walkthrough` + 1 quick checkpoint question.

---

## Worked example (illustrates the style)

> 📘 **Lesson 3/7 · `src/queues/email.queue.ts` [🔥 Core]** ▓▓▓░░░░░░
>
> 🗺️ **Where we are:** `POST /signup` → `auth.controller.ts` → **`[email.queue.ts]`** → `email.worker.ts` → `mailer.ts`
>
> 🔌 **How it connects (The Wiring)**
>
> - **Incoming:** Called by `auth.controller.ts` (line 42) right after the user row is saved in Postgres. It passes just `user.id` (`"usr_99"`).
> - **Outgoing:** Pushes a job record into **Redis**. In Lesson 4, `email.worker.ts` will pick up that job from Redis and actually send the email.
>
> 🎯 **Purpose:** This file is the "drop-off counter" that lets our signup API hand off slow email sending to a background queue so the user gets an instant response.
>
> 🔍 **Walkthrough**
>
> ```ts
> // src/queues/email.queue.ts:1-5
> import { Queue } from "bullmq";
> import { redis } from "../lib/redis";
>
> export const emailQueue = new Queue<{ userId: string }>("email", {
>   connection: redis,
> });
> ```
>
> Lines 4–6 create a BullMQ queue named `"email"`. Creating this `Queue` object does **not** send any emails by itself—it just creates a producer handle that knows how to write job messages into Redis under the `"email"` key.
>
> - **🔤 Syntax & types unpacked:** `new Queue<{ userId: string }>(...)` uses a **TypeScript Generic**. The `<{ userId: string }>` part tells TypeScript: _"Every job added to this queue MUST contain an object with a `userId` string."_ If you accidentally pass `{ email: "riya@example.com" }` later, TypeScript will underline it in red before you even run the code.
>
> ```ts
> // src/queues/email.queue.ts:8-16
> export async function enqueueWelcomeEmail(userId: string) {
>   await emailQueue.add(
>     "send-welcome",
>     { userId },
>     {
>       attempts: 3,
>       backoff: { type: "exponential", delay: 5000 },
>       removeOnComplete: true,
>     },
>   );
> }
> ```
>
> `enqueueWelcomeEmail` writes a `"send-welcome"` job into Redis.
>
> - **📦 Payload trace:** In Lesson 2, our Canonical Payload became a saved database user `{ id: "usr_99", email: "riya@example.com", name: "Riya" }`. Notice that here we strip everything else out and only pass `{ userId: "usr_99" }` into Redis.
> - **🟢 Happy path:** Redis stores the job in ~2 milliseconds, `await emailQueue.add(...)` resolves, and `auth.controller.ts` immediately returns `201 Created` to Riya.
> - **🔴 If things break:**
>   1. **If the email provider (Resend/SendGrid) is down:** This function _still succeeds_ because it only talks to Redis! The worker will handle the down email provider using the `attempts: 3` and `exponential` backoff settings (waiting 5s, then 10s, then 20s).
>   2. **If Redis itself is down:** `emailQueue.add()` throws a connection error back to `auth.controller.ts`.
>
> **Function summary:** `enqueueWelcomeEmail(userId: string)` → returns `Promise<void>` · side effect: writes a job hash into Redis · called by `auth.controller.ts` · throws if Redis is unreachable.
>
> 🧠 **Under the hood (Naive Way vs. This Way)**
>
> - **How a beginner might write it:** Inside `auth.controller.ts`, call `await sendEmail(riya.email)` directly before returning the HTTP response.
> - **Why that breaks:** Third-party email APIs take 500ms–3000ms and occasionally time out. Doing it inline makes the user stare at a spinning signup button—and if the email API throws a 500 error, the user thinks their account signup failed even though their user row was already created in Postgres!
> - **What happens under the hood here:** `await emailQueue.add(...)` only waits for **Redis to save a tiny JSON note** in memory, not for the email to be sent. Decoupling "recording the work" from "doing the work" is the core idea of asynchronous background processing.
>
> ⚖️ **Senior code audit**
>
> - **Why pass only `userId` instead of the full `user` object?:** Smaller payloads keep Redis memory low, avoid storing sensitive PII in Redis logs, and guarantee that if Riya changes her email 2 seconds later before the worker runs, the worker fetches her _latest_ email from Postgres instead of sending to stale data.
> - **Honest critique (Interview gold):** Look at what happens if Redis is temporarily down when `enqueueWelcomeEmail` runs—it throws an unhandled error after the user is already in Postgres. In an interview, you can say: _"Right now, if Redis blips, the signup returns a 500 even though the DB write succeeded. To improve this, I'd wrap the enqueue call in a try/catch with a fallback DB flag or use the Transactional Outbox pattern."_
>
> 🧪 **Try in your IDE (Optional):** Hover over `.add(` on line 9 in VS Code/Cursor—you'll see TypeScript enforcing `{ userId: string }` as the second argument thanks to that generic on line 4.
>
> 🎤 **Say it in an interview:** _"Instead of sending the welcome email synchronously inside the signup endpoint—which would slow down response times and couple account creation to third-party mail uptime—I used BullMQ to enqueue a lightweight job containing just the `userId` with 3 exponential-backoff retries."_
>
> ✅ **Checkpoint**
>
> 1. Suppose Riya signs up at 10:00:00 AM, the job is queued with `{ userId: "usr_99" }`, Redis is overloaded so the worker doesn't run until 10:02:00 AM, and at 10:01:00 AM Riya updates her email address in her profile settings. Because we only passed `userId` into the queue, what happens at 10:02:00 AM—and what would have happened if we had passed the whole `user` object into the queue?
> 2. If the email provider fails on the very first attempt, roughly how many seconds will BullMQ wait before try #2 and try #3?

---

## Checkpoint design

- **1 question** for standard or `🧩 Supporting` lessons; **2 questions** for `🔥 Core` lessons.
- Never ask pure copy-paste trivia ("What is the queue name on line 4?").
- Always ground questions in **Why**, **Predict (using the Canonical Payload)**, or **Failure ("What happens if...")**.

## Question bank

| Type                    | Pattern                                                         | Example                                                                                                                        |
| ----------------------- | --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| **Why (Design)**        | Why did we do X instead of the naive Y?                         | "Why did we validate the payload in middleware instead of waiting until the database query?"                                   |
| **Predict (Trace)**     | If our Canonical Payload has value X, what comes out at line Y? | "If `qty` in our payload is `-1`, which exact line catches it and what HTTP status comes back?"                                |
| **Failure ("What if")** | What happens when dependency X fails mid-function?              | "If line 28 succeeds in deducting balance, but line 31 crashes before creating the order, what state is the database left in?" |
| **Wiring**              | How does data get from File A to File B?                        | "Where does `req.user` in this controller actually get attached before this function runs?"                                    |
| **Syntax check**        | Can you read what this expression produces?                     | "In `{ data: { items = [] } = {} }`, what is `items` if the API returns `{}`?"                                                 |
| **Critique**            | Spot the limitation or trade-off                                | "What would break in this function if a user double-clicked the submit button twice in 20ms?"                                  |

## Feedback patterns

- **Solid:** _"Spot on—especially your point about [specific detail]. One quick senior-level bonus: [1-sentence insight]. Ready for Lesson N+1 (`next-file.ts`)?"_
- **Partial:** _"You nailed the first part ([what they got right]). The slight plot twist is what happens next: [trace using the Canonical Payload]. With that in mind, what do you think happens to [follow-up question]?"_
- **Off:** _"That's actually how 90% of people read this at first because [why it looks that way]. Here's what's really happening under the hood: [2-sentence plain trace]. Let's test that: [simpler check question]."_
- **Stuck / "I don't know":** Give a specific line-number hint first (_"Check line 11—notice what happens inside the `catch` block"_), explain gently, and never make the learner feel bad.
