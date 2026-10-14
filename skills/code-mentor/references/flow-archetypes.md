# Flow archetypes

Pick the archetype (or blend of 2–3) that matches the feature you are teaching. Use its **Flow** to order your lessons, its **Junior Confusion Traps** to anticipate where the learner will struggle, and its **Interview Probes** to fuel the `🧠 Under the hood` and `⚖️ Senior code audit` sections.

## Contents

- API endpoint / CRUD
- Authentication and authorization
- Background jobs and queues
- Realtime (WebSockets, SSE)
- Frontend feature
- Database schema, ORM, and migrations
- Caching
- AI / LLM / agent features
- Third-party integrations and webhooks

---

## API endpoint / CRUD

- **Execution Flow:** `route / router` → `middleware` → `schema validation (Zod/DTO)` → `controller / procedure` → `service / business logic` → `data access / ORM` → `response serialization` → `central error handler`
- **Where Juniors Get Confused (The Traps):**
  - Losing track of how `req` or `ctx` gets populated by middleware before reaching the controller.
  - Not understanding why validation, controller, and service are split into separate layers instead of written inside one function.
  - Thinking frontend validation is enough and wondering why backend Zod/DTO validation is duplicated.
- **Naive Way vs. Production Way:**
  - _Naive:_ Put raw SQL/ORM queries, `if (!req.body.email)` checks, and `res.status(200)` all inside one 150-line route handler without transactions.
  - _Production:_ Validate shape at the boundary (Zod), enforce business rules in a service, wrap multi-table writes in a DB transaction, and map domain errors to HTTP status codes cleanly.
- **Interview Probes:** What happens if validation fails vs. if a DB unique constraint fails? Why use a transaction here? What if the user sends the exact same POST request twice (idempotency)?

## Authentication and authorization

- **Execution Flow:** `credential input` → `password hashing / OAuth callback` → `token / session generation` → `cookie / header transport` → `auth middleware (verification)` → `RBAC / permission guard` → `refresh / logout / revocation`
- **Where Juniors Get Confused (The Traps):**
  - Confusing **Authentication** (_who are you?_) with **Authorization** (_are you allowed to touch this resource?_).
  - Confusing **Hashing** (one-way, like bcrypt/Argon2 for passwords) with **Encoding/Signing** (JWTs—anyone can base64-decode a JWT payload, the secret only verifies it wasn't tampered with!).
  - Not knowing how `req.user` or `ctx.session` magically appears in protected routes.
- **Naive Way vs. Production Way:**
  - _Naive:_ Store plain JWTs in `localStorage` with a 30-day expiry and check user roles manually inside every route.
  - _Production:_ HttpOnly/Secure cookies (or short-lived access tokens + rotating refresh tokens) verified in reusable middleware, plus resource-level ownership checks (preventing IDOR attacks where User A passes User B's `orderId`).
- **Interview Probes:** Why hash passwords with salt instead of encrypting them? Can a client read the payload inside a JWT? Why `HttpOnly` cookies vs. `localStorage` (XSS vs. CSRF)? How do you revoke a stateless JWT if a user's account is compromised?

## Background jobs and queues

- **Execution Flow:** `API / cron trigger` → `producer (queue.add)` → `broker (Redis / RabbitMQ / SQS)` → `worker process` → `external side effect / DB update` → `retry / exponential backoff` → `dead-letter / failure handling`
- **Where Juniors Get Confused (The Traps):**
  - Thinking `await queue.add()` waits for the worker to finish the job (it only waits for the broker to store the job!).
  - Not realizing the **Producer** and the **Worker** can run in completely separate Node.js processes or servers.
  - Not understanding why a job needs to be **idempotent** (safe to run twice if the worker crashes right after sending an email/charging a card but before telling Redis it finished).
- **Naive Way vs. Production Way:**
  - _Naive:_ Run slow emails, PDF generation, or webhook fan-outs directly inside the HTTP request, or fire an un-awaited `sendEmail()` promise that vanishes if the server restarts.
  - _Production:_ Persist minimal job payload in Redis/broker, process with controlled concurrency, use exponential backoff, and guard side effects with idempotency checks.
- **Interview Probes:** Why pass only an ID in the job payload instead of the full object? What happens if the worker process crashes mid-job? How does exponential backoff prevent hammering a struggling downstream API? How do you handle poison-pill jobs that fail all retries?

## Realtime (WebSockets, SSE, Pub/Sub)

- **Execution Flow:** `connection handshake & auth` → `channel / room join` → `inbound event / mutation` → `pub/sub broadcast (Redis adapter)` → `client listener` → `UI state update` → `disconnect / reconnect recovery`
- **Where Juniors Get Confused (The Traps):**
  - Assuming `io.to(room).emit()` works across multiple backend servers without a Redis Pub/Sub adapter.
  - Forgetting what happens when a user's Wi-Fi drops for 5 seconds and reconnects (missed events).
- **Naive Way vs. Production Way:**
  - _Naive:_ Poll `GET /messages` every 1 second from the frontend, or hold in-memory WebSocket maps on a single server with no auth on handshake.
  - _Production:_ Authenticate on handshake, use SSE for one-way streams or WebSockets with a pub/sub broker for two-way rooms, and fetch missed state on reconnect.
- **Interview Probes:** Why did you choose WebSockets vs. Server-Sent Events (SSE) vs. polling? What happens when you scale from 1 backend server to 4 backend servers behind a load balancer?

## Frontend feature (React / Next.js / State)

- **Execution Flow:** `route / page load` → `server vs. client boundary` → `user interaction / event handler` → `local state / form hook` → `data fetching / mutation (tRPC / ORPC / React Query / Server Action)` → `cache invalidation / optimistic update` → `re-render & loading/error/empty UI`
- **Where Juniors Get Confused (The Traps):**
  - Mixing up **Server Components** (run once on the server, zero client bundle) and **Client Components** (`"use client"`, interactive hooks) in Next.js.
  - Copying server data into `useState` via `useEffect` instead of letting the query cache (TanStack Query / SWR) own server state.
  - Stale closures inside `useEffect` or event handlers, and prop drilling vs. context/store.
- **Naive Way vs. Production Way:**
  - _Naive:_ `useEffect(() => { fetch(...).then(setData) }, [])` with no race-condition cleanup, no loading skeleton, no error boundary, and manual window reloads after a mutation.
  - _Production:_ Type-safe query/mutation hooks (or Server Actions), automatic cache invalidation on mutation success, disabled submit buttons during pending state, and proper loading/empty/error states.
- **Interview Probes:** Why didn't you store this API response in `useState`? What triggers a re-render in this component? How do you prevent double-submission if the user clicks twice? What does the user see while the request is in flight or fails?

## Database schema, ORM, and migrations

- **Execution Flow:** `schema definition (Prisma / Drizzle / SQL)` → `relations & foreign keys` → `indexes & unique constraints` → `migration file` → `ORM queries / transactions in code`
- **Where Juniors Get Confused (The Traps):**
  - Not realizing that an ORM call inside a `for` loop causes the **N+1 query problem** (100 database network round-trips instead of 1 `JOIN` or `WHERE IN`).
  - Not knowing what a database **index** actually does under the hood (avoiding a full table scan) or why you need composite unique constraints.
  - Relying on application-level `findUnique` before `create` instead of a DB `@unique` constraint (which fails under concurrent requests—race condition!).
- **Naive Way vs. Production Way:**
  - _Naive:_ Check `const exists = await db.user.findFirst(...)` in JS without a unique constraint in the DB, or run two related writes without `db.$transaction()`.
  - _Production:_ Enforce invariants in the database schema (`@unique`, foreign keys, `onDelete` rules), index columns used in `WHERE`/`ORDER BY`, and batch/join related reads.
- **Interview Probes:** Why put a unique constraint in the DB if the backend code already checks for duplicates? Which queries in this feature would become slow at 1 million rows without an index? What happens to child records when a parent row is deleted?

## Caching

- **Execution Flow:** `incoming request` → `cache key construction` → `cache lookup (Redis / memory)` → `HIT (return fast)` vs. `MISS (query DB → write to cache with TTL → return)` → `cache invalidation on write`
- **Where Juniors Get Confused (The Traps):**
  - Forgetting to invalidate or update the cache when the underlying database row is edited, causing users to see stale data.
  - Caching user-specific data under a global cache key (leaking User A's data to User B!).
- **Naive Way vs. Production Way:**
  - _Naive:_ Cache forever with no TTL, or forget to include filter/user parameters in the cache key.
  - _Production:_ Deterministic cache keys (`user:${userId}:profile`), explicit TTLs, explicit invalidation on mutation, and graceful fallback to the DB if Redis is down.
- **Interview Probes:** How is the cache key structured? How did you pick the TTL? What happens when the user updates their data—do you delete the cache key or update it? What if Redis goes down—does the whole API crash or fall back to Postgres?

## AI / LLM / agent features

- **Execution Flow:** `user input & guardrails` → `context / RAG retrieval` → `prompt assembly` → `LLM API call (streaming or structured output)` → `tool / function execution loop` → `Zod output parsing & validation` → `persistence & token/cost/error handling`
- **Where Juniors Get Confused (The Traps):**
  - Trusting raw LLM text output to be valid JSON without schema validation (`Zod`) or retry/repair logic.
  - Not understanding how **tool calling** works (the LLM doesn't run your database query itself—it returns a structured JSON request asking _your code_ to run the function and feed the result back!).
- **Naive Way vs. Production Way:**
  - _Naive:_ Concatenate raw user input into a string prompt, call `JSON.parse(response)` blindly, and let a single 30-second LLM call block without streaming or timeouts.
  - _Production:_ Separate system/user messages, enforce structured output schemas, cap tool-call loop iterations, stream responses to the UI, and handle rate limits (HTTP 429) with fallbacks.
- **Interview Probes:** What happens if the LLM hallucinates invalid JSON or calls a tool with bad arguments? How do you prevent an infinite agent loop? How do you control latency and token costs as conversation history grows?

## Third-party integrations and webhooks

- **Execution Flow:** `outbound API client + secrets` → `request + idempotency key` → `provider processes async` → `inbound webhook POST endpoint` → `cryptographic signature verification (HMAC)` → `idempotency check (eventId)` → `DB state update` → `200 OK fast response`
- **Where Juniors Get Confused (The Traps):**
  - Wondering why we need a **webhook** at all when we already got a response from the initial API call.
  - Not understanding why anyone on the internet could send a fake `POST /webhook/stripe` saying `"payment_succeeded"` unless we verify the **cryptographic signature** against the raw request body.
  - Not realizing Stripe/GitHub/Razorpay will send the **exact same webhook event multiple times** if your server takes too long or blips.
- **Naive Way vs. Production Way:**
  - _Naive:_ Trust `req.body.status === "paid"` without signature verification, and credit the user's wallet every time the webhook hits without checking if `event.id` was already processed.
  - _Production:_ Verify the HMAC signature using the raw body buffer, store processed `eventId`s in a unique DB table inside a transaction, and return `200 OK` quickly.
- **Interview Probes:** How do you stop an attacker from forging a fake payment webhook with Postman? What happens if the webhook provider sends the same event twice? Why must a webhook endpoint return `200 OK` quickly instead of doing heavy work inline?

---

## Always include (when present in the feature)

- **Environment & Config validation:** How missing `.env` variables are caught at startup rather than crashing at 2 AM in production.
- **Error Handling & Logging:** How errors bubble up from deep functions to a clean client-safe response without leaking stack traces.
- **Tests:** A short final lesson showing what scenarios the unit/integration tests prove and what edge cases are still untested.
