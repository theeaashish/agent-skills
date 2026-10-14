# Progress format

Learning a feature spans multiple sessions across different days or chat windows. These state files allow a brand-new chat session to pick up **at the exact lesson, with the exact same Canonical Payload and learner context**, without missing a beat.

## Contents

- Folder layout
- PLAN.md format
- PROGRESS.md format
- When to update state files
- Plain-chat Resume Code (no filesystem)

---

## Folder layout

```text
.learning/
└── <feature-slug>/               e.g. email-queue, jwt-auth, order-checkout
    ├── PLAN.md                   created in Lesson 0 (big picture, canonical payload, roadmap)
    ├── PROGRESS.md               updated after every lesson & checkpoint
    └── INTERVIEW_NOTES.md        created in Phase 4 Step 2 (permanent interview study sheet)
```

Use a short, lowercase, hyphenated `<feature-slug>`. Remind the learner once in Lesson 0 that they may want to add `.learning/` to `.gitignore`.

---

## PLAN.md

```markdown
# Learning Plan: <Feature Name>

- **Slug:** `<feature-slug>`
- **Scope Mode:** `full-feature` (Personal/Module) | `diff-slice` (Company PR / Git Diff)
- **Studied at:** `<branch>` @ `<short-sha>`
- **Created:** `<YYYY-MM-DD>`
- **Pacing Mode:** `standard` | `fast`

## Big Picture

- **Problem:** <1–2 plain sentences on what user/business pain this solves>
- **Why Naive Fails:** <1–2 sentences on why the naive 3-line beginner approach breaks>
- **High-Level Flow:** `entry` → `step 2` → `step 3` → `storage/output`

## 📦 Canonical Payload (Trace This Across Every Lesson)

- **Initial Input:** `<e.g., POST /api/orders with { userId: "usr_99", itemId: "item_42", qty: 2 }>`
- **Key Transformations:**
  1. `L1 (route/zod)` → validated DTO
  2. `L2 (service)` → enriched order object with calculated price
  3. `L3 (db/queue)` → saved DB row `ord_101` + queued event `{ orderId: "ord_101" }`

## Roadmap

| #   | Lesson Title             | File(s) / Diff Slice             | Priority      | Status |
| --- | ------------------------ | -------------------------------- | ------------- | ------ |
| 0   | Big Picture & Mental Map | (architecture overview)          | 🔥 Core       | ✅     |
| 1   | <title>                  | `path/to/fileA.ts`               | 🔥 Core       | ⬜     |
| 2   | <title>                  | `path/to/fileB.ts (lines 40–95)` | 🔥 Core       | ⬜     |
| 3   | <title>                  | `path/to/fileC.ts`               | 🧩 Supporting | ⬜     |

Status legend: `⬜` not started · `🔄` in progress · `✅` mastered · `🟡` completed but shaky / skipped checkpoint
```

Saving `<short-sha>` allows future sessions to detect if the code changed since Lesson 0 (`git diff <short-sha>...HEAD -- <files>`) and alert the learner if a file was modified.

---

## PROGRESS.md

```markdown
# Progress: <Feature Name>

- **Current State:** `Lesson 3 delivered — awaiting checkpoint answer` (or `Ready to start Lesson 4`)
- **Last Updated:** `<YYYY-MM-DD>`
- **Canonical Payload Reminder:** `{ userId: "usr_99", itemId: "item_42", qty: 2 }` → `ord_101`
- **Learner Profile & Comfort Notes:**
  - _Comfortable with:_ basic TS, Express routes, `async/await`
  - _Needs extra clarity on:_ TypeScript generics, Redis queues, DB transactions
  - _Preferred style:_ loves concrete payload traces and "naive vs. real" comparisons

## Completed Lessons

- **L0:** ✅ Solid — explained the async bottleneck clearly.
- **L1 (`order.schema.ts`):** ✅ Solid — understood `z.infer<typeof OrderSchema>`.
- **L2 (`order.service.ts`):** 🟡 Partial — understood happy path, shaky on why `db.$transaction` rolls back on error.

## Open Checkpoint (Awaiting Learner Reply)

1. <Exact question 1 asked at the end of current lesson>
2. <Exact question 2 (if 🔥 Core lesson)>
   _(Clear this section once answered.)_

## Revisit List (For Phase 4 & Review Mode)

- `L2 (order.service.ts)`: Database transaction rollback behavior and race conditions when two orders hit simultaneously.

## Parked / Side Questions

- <Any off-topic question the learner asked mid-lesson that we promised to cover later>
```

---

## When to update state files

1. **End of Lesson 0:** Create `.learning/<feature-slug>/PLAN.md` and `PROGRESS.md`.
2. **When delivering Lesson N:** Update `PROGRESS.md` (`Current State` = awaiting checkpoint for Lesson N, and paste the exact `Open Checkpoint` questions) and mark Lesson N as `🔄` in `PLAN.md`.
3. **After learner answers Checkpoint N:**
   - Mark Lesson N as `✅` (or `🟡` if shaky/skipped) in `PLAN.md` and `PROGRESS.md`.
   - Log any confusion in `Revisit List` and update `Learner Profile & Comfort Notes`.
   - Clear `Open Checkpoint`.
4. **If a Mini-Detour Lesson is added or lessons are merged (`fast` mode):** Update the Roadmap table in `PLAN.md`.
5. **In Phase 4 Step 2:** Create `INTERVIEW_NOTES.md`.

---

## Plain-chat Resume Code (no filesystem)

When running in a regular chat interface without file write access, output this compact block at the very bottom of every lesson so the learner can copy-paste it into a future chat session:

```text
📌 RESUME CODE · code-mentor
Feature: <feature-slug> (<full-feature | diff-slice>)
Payload: <e.g., POST /signup { email: "riya@example.com" } -> usr_99>
Done: L0 ✅, L1 ✅, L2 🟡 (revisit: DB transactions)
Current/Next: L3 · src/queues/email.queue.ts [🔥 Core]
Roadmap remaining: 3 email.queue -> 4 email.worker -> 5 mailer -> 6 wrap-up
Open checkpoint: Q1: Why pass only userId instead of full user object?
Learner notes: unpack TS generics; keep traces concrete
```

When a learner pastes a `📌 RESUME CODE` block into a new chat, parse it immediately, ask them to paste the next file (if not already in context), and resume seamlessly without repeating completed lessons.
