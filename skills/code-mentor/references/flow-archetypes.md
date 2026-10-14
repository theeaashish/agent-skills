# Flow archetypes

Pick the archetype (or combination) that matches the feature, use its flow as the lesson order, and use its probes to decide what deserves an "Under the hood" or "Why this way" moment. Real features usually blend two or three. These are starting points, so follow what the code actually does.

## Contents
- API endpoint / CRUD
- Authentication and authorization
- Background jobs and queues
- Realtime (WebSockets, SSE)
- Frontend feature
- Database schema and migrations
- Caching
- AI / LLM / agent features
- Third-party integrations and webhooks

## API endpoint / CRUD

**Flow:** route → middleware → validation → controller → service/business logic → data access → response shape → error handling

**Probes:** What is validated and where? What status codes are returned, and why? Is anything done in a transaction? What if the same request arrives twice? Where do errors turn into HTTP responses?

## Authentication and authorization

**Flow:** credential intake → hashing/verification → token or session creation → storage (cookie/header) → middleware that checks it on each request → permission checks → logout/refresh/expiry

**Probes:** Why hash rather than encrypt? What is inside the token and who can read it? Where is it stored and what attack does that choice defend against? What happens when it expires? Authentication vs. authorization: where is each enforced?

## Background jobs and queues

**Flow:** trigger → producer (enqueue) → broker (e.g. Redis) → worker → side effects → success/failure handling → retries/dead-letter → cleanup

**Probes:** Why async here? What is in the payload and why so small? What happens when the worker crashes mid-job? Is the job idempotent? Retry and backoff policy? Concurrency? How would you know a job failed?

## Realtime (WebSockets, SSE)

**Flow:** connection handshake and auth → room/channel subscription → event in → handler → broadcast → client update → disconnect/reconnect handling

**Probes:** How is the connection authenticated? How do multiple server instances share events? What happens on reconnect (missed messages)? Backpressure and rate limiting? WebSocket vs. polling vs. SSE?

## Frontend feature

**Flow:** user event → component → local state or hook → data fetching / mutation → cache or store update → re-render → loading/error/empty states

**Probes:** Where does state live and why there? What triggers a re-render? How is server data cached and invalidated? What does the user see while loading or on failure? Optimistic updates? Accessibility basics?

## Database schema and migrations

**Flow:** entities → fields and types → relations → constraints and indexes → migration → queries that use it

**Probes:** Why this relation shape (one-to-many, many-to-many)? Which indexes exist and which query does each serve? What do constraints protect against? How would a migration be applied safely on a live system?

## Caching

**Flow:** request → cache lookup → hit path / miss path → source fetch → cache write with TTL → invalidation

**Probes:** What is the cache key? TTL choice? What invalidates it? Stale data risks? Stampede when many misses happen at once?

## AI / LLM / agent features

**Flow:** user input → prompt/context assembly → model call → tool/function calls (if any) → response parsing and validation → state/memory → output → error and cost handling

**Probes:** How is the prompt built and what goes in the context window? How is structured output validated? What happens on a tool error or malformed output? How are cost, latency, and rate limits handled? How are results evaluated?

## Third-party integrations and webhooks

**Flow:** credentials/config → outbound client wrapper → request/response mapping → error and retry policy → inbound webhook verification → idempotent processing

**Probes:** Where do secrets live? How are webhook signatures verified? What if the provider sends the event twice or out of order? Timeouts and retries? What happens when the provider is down?

## Always include, when they exist

- **Config and environment**: what each variable does and what happens if it's missing.
- **Error paths**: they reveal the most about design maturity.
- **Tests**: a short lesson near the end on what behavior the tests protect and what they don't.
