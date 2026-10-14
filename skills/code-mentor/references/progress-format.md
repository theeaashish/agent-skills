# Progress format

Learning spans several sessions, often in different chats. These files are how the next session picks up exactly where the last one ended. Write them for a future you who remembers nothing.

## Contents
- Folder layout
- PLAN.md
- PROGRESS.md
- When to update
- Resume code (no filesystem)

## Folder layout

```
.learning/
└── <feature-slug>/          e.g. email-queue, jwt-auth, order-checkout
    ├── PLAN.md              the roadmap (written in Lesson 0, edited if the plan changes)
    ├── PROGRESS.md          where the learner is (updated after every checkpoint)
    └── INTERVIEW_NOTES.md   created at wrap-up
```

Use a short, lowercase, hyphenated slug. Reuse the same slug every session.

## PLAN.md

```markdown
# Learning plan: <Feature name>
Slug: <feature-slug> · Studied at: <branch> @ <short-sha> · Created: <date> · Mode: standard | fast

## Big picture
**Problem:** <one or two sentences>
**Solution:** <one paragraph>
**Flow:** entry → step → step → storage/output

## Roadmap
| # | Lesson | Files | Priority | Status |
|---|--------|-------|----------|--------|
| 0 | Big picture | (none) | 🔥 | ✅ |
| 1 | <title> | `path/a.ts` | 🔥 | ⬜ |
| 2 | <title> | `path/b.ts`, `path/c.ts` | 🧩 | ⬜ |

Status: ⬜ not started · 🔄 in progress · ✅ done · 🟡 done but shaky
```

Recording the commit lets a later session notice when the code has changed since the lessons ("the file changed after we studied it, want me to re-check?").

## PROGRESS.md

```markdown
# Progress: <Feature name>
**Current:** Lesson 3, awaiting checkpoint answers   (or: "Ready to start Lesson 4")
**Last session:** <date>
**Learner level notes:** e.g. comfortable with async/await; new to Redis; prefers short answers

## Completed
- L0 ✅ solid
- L1 ✅ solid
- L2 🟡 partial. Shaky on: why an idempotency key is needed

## Open checkpoint
1. <exact question as asked>
2. <exact question as asked>
(Delete this section once answered.)

## Revisit list
- L2: idempotency keys (re-quiz in review mode)

## Parked questions
- <side questions the learner asked that we deferred>
```

Keep each entry short. The goal is enough state to resume and to drive review mode, not a diary.

## When to update

- After Lesson 0: create PLAN.md and PROGRESS.md.
- When a lesson is delivered: set PROGRESS "Current" and write the open checkpoint questions verbatim.
- After the learner answers: mark the lesson ✅ or 🟡, add gaps to the revisit list, clear the open checkpoint.
- If the plan changes (a lesson added, split, or skipped): update PLAN.md and say so in one line.

## Resume code (no filesystem)

In plain chat there is nowhere to save files, so end each lesson with a compact block the learner can paste into a new chat together with the relevant code:

```
📌 RESUME CODE · code-mentor
Feature: email-queue
Done: L0 ✅, L1 ✅, L2 🟡 (shaky: idempotency keys)
Next: L3 · src/queues/email.queue.ts
Roadmap: 3 email.queue · 4 email.worker · 5 mailer · 6 retries/failure · 7 tests · 8 wrap-up
Open checkpoint: (none)
```

When a learner pastes one, treat it as the state, re-read the files it names (the learner needs to supply them if you don't have repo access), and continue from "Next".
