# Lesson template

Read this before teaching the first lesson. It covers the format, a worked example (illustrating style, not content to copy), and the checkpoint question bank.

## Contents
- Lesson format
- Worked example
- Checkpoint design
- Question bank
- Feedback patterns

## Lesson format

```
📘 Lesson N/T · `path/to/file.ext`            ▓▓▓░░░░░░   (progress bar, 9 cells)

🗺️ Where we are
   routes → controller → **queue** → worker → provider      ← "you are here" in bold

🎯 Purpose
   One sentence: why this file exists and what would break without it.

🔍 Walkthrough
   [chunk 1: file:lines]  → explanation (line by line if dense)
   [chunk 2: file:lines]  → explanation
   Function summary: name(inputs) → output · side effects · called by · calls · on failure

🧠 Under the hood
   One concept that would otherwise feel like magic.

⚖️ Why this way
   Decision · the alternative · the trade-off.
   (If inferring intent, say so: "likely because… , worth confirming in the PR/commit".)

🎤 Say it in an interview
   2–3 sentences in first person the learner could say aloud.

✅ Checkpoint
   1. ...
   2. ...
   Reply in your own words (rough is fine). Say `hint` if stuck, or `next` to skip.
```

Notes:
- Skip a section when it has nothing real to say (a config file may have no "Under the hood"). Empty sections are filler.
- The "Where we are" trail uses the actual lesson files, so the learner always sees the whole story and their position.
- For 🧩 supporting lessons, compress to Purpose + Walkthrough + one-line checkpoint.

## Worked example (illustrates the style)

> 📘 **Lesson 3/8 · `src/queues/email.queue.ts`** ▓▓▓░░░░░░
>
> 🗺️ **Where we are:** `POST /signup` → `auth.controller` → **`email.queue`** → `email.worker` → `mailer`
>
> 🎯 **Purpose:** This file is the "drop-off counter". The signup request leaves a note saying "send this person a welcome email" and returns immediately; a separate worker does the slow part later.
>
> 🔍 **Walkthrough**
> ```ts
> // src/queues/email.queue.ts:1-4
> import { Queue } from "bullmq";
> import { redis } from "../lib/redis";
>
> export const emailQueue = new Queue("email", { connection: redis });
> ```
> Line 4 creates a queue named `"email"`. Nothing is sent here: a `Queue` object is only a handle that pushes job records into Redis under that name. `connection: redis` says which Redis to use, so producer and worker meet at the same place.
>
> ```ts
> // src/queues/email.queue.ts:6-13
> export async function enqueueWelcomeEmail(userId: string) {
>   await emailQueue.add("welcome", { userId }, {
>     attempts: 3,
>     backoff: { type: "exponential", delay: 5000 },
>     removeOnComplete: true,
>   });
> }
> ```
> `add(jobName, data, options)` stores a job. We pass only `userId`, not the whole user, so the job stays small and the worker reads fresh data when it runs. `attempts: 3` means one try plus two retries. With exponential backoff at 5000 ms, retries wait roughly 5 s, then 10 s. `removeOnComplete` deletes finished jobs so Redis doesn't fill up.
>
> **Function summary:** `enqueueWelcomeEmail(userId)` → resolves once the job is stored · side effect: writes to Redis · called by `auth.controller` · on Redis failure it throws, so the caller decides what to do.
>
> 🧠 **Under the hood:** `await add(...)` finishes when Redis has *stored* the job, not when the email is sent. That gap is the whole point of a queue.
>
> ⚖️ **Why this way:** Sending inside the request would be simpler, but a slow or failing mail provider would make signup slow or fail. The queue trades a little complexity (Redis, a worker) for a fast signup and automatic retries.
>
> 🎤 **Say it in an interview:** "I moved the welcome email to a BullMQ queue so signup doesn't depend on the mail provider. Jobs retry with exponential backoff, and the payload is just the user ID so the worker always reads current data."
>
> ✅ **Checkpoint**
> 1. Why do we pass `userId` instead of the full user object into the job?
> 2. The mail provider is down for a minute. Walk me through what happens to a job from `add` until it finally succeeds or fails.

## Checkpoint design

Two or three questions, mixing types. At least one *why* and one *what if*, because those can't be answered by copying a line from the lesson. Keep each answerable in a few sentences. Don't reuse wording from the lesson, or the question tests reading rather than understanding.

## Question bank

| Type | Pattern | Example |
|---|---|---|
| Recall | What does X return / where does Y get set? | "What does `verifyToken` return when the token is expired?" |
| Why | Why this choice instead of the obvious one? | "Why a queue rather than sending inline?" |
| Predict | What happens if… (trace with concrete values) | "If two requests arrive with the same idempotency key, what does the second one get?" |
| Failure | What breaks when X is down, slow, or duplicated? | "The worker crashes after sending but before marking done. What's the user-visible effect?" |
| Modify | How would you change it to do Z? | "How would you limit retries to 5?" |
| Locate | Where in the code is Q handled? | "Where would you look if users report double emails?" |
| Critique | What's the weakest part, and what would you improve? | "What is missing from this file's error handling?" |
| Teach-back | Explain it as if to a teammate | "Explain in three sentences what this function is for." |

Use Recall sparingly. Weight toward Why, Predict, and Failure, which is where interviews live.

## Feedback patterns

- **Solid:** "Yes: [restate the key idea in one line]. One level-up: [insight]." Then offer the next lesson.
- **Partial:** "You've got [right part]. The missing piece is [gap]." Then a concrete re-trace with values, and a fresh question.
- **Off:** "That's a common way to read it, because [why it's tempting]. Here's what actually happens: [short trace]." Then a fresh question.
- **Don't know:** hint first ("Look at line 9: what does `attempts` control?"), then a scaffolded explanation, then ask again more simply.
- Keep feedback specific to what they wrote. Generic praise ("Great job!") teaches nothing and erodes trust.
