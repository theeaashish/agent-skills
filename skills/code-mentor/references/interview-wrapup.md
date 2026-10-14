# Interview wrap-up

Run this phase after the final lesson checkpoint is completed. Its job is to transform _"I understood the lessons"_ into _"I can walk into any interview and explain this feature like a strong engineer who owns every line."_

Run these steps across separate turns, waiting for the learner's reply between each step.

## Contents

- Step 1: End-to-end payload trace
- Step 2: Generate INTERVIEW_NOTES.md immediately
- Step 3: The 2-minute spoken teach-back
- Step 4: Mock interview ladder
- Spaced review

---

## Step 1: End-to-end payload trace

Give the learner one concrete, realistic scenario (using or slightly varying the Canonical Payload from Lesson 0—for example: _"A user clicks 'Checkout' with 2 items in their cart, and the payment webhook arrives 3 seconds later"_).

Ask them to **narrate the full journey from start to finish in their own words**, naming:

1. Which file/entry point receives the trigger first.
2. Which files/functions it jumps through in order.
3. How the data changes or gets stored at each step.

**How to respond to their trace:**

- Do not interrupt while they write it.
- Praise the steps and wiring they nailed.
- Point out any skipped steps or minor mix-ups (referencing the lesson number so they can connect the dots).
- Show a clean, compact **5–8 line Master Call Trace** (`File A -> File B -> File C`) to lock the final mental map into place.

---

## Step 2: Generate `INTERVIEW_NOTES.md` immediately

Right after Step 1 (before starting long mock-interview drills), generate and save `.learning/<feature-slug>/INTERVIEW_NOTES.md` (or print it in chat if there is no filesystem). Delivering the cheat sheet now ensures the learner always walks away with their complete interview revision guide even if they need to leave early.

Tailor the notes to the real code and highlight any topics where the learner was shaky with `🟡`:

```markdown
# Interview Cheat Sheet: <Feature Name>

## 1. The 30-Second Elevator Pitch

> **When the interviewer asks:** _"Tell me about this feature / project."_
> <3 natural, first-person sentences covering: (1) the user/business problem, (2) the core technical architecture you used, and (3) the reliability/performance result.>

## 2. The 2-Minute Deep-Dive Walkthrough

> **Follow the P-N-A-T formula (Problem → Naive bottleneck → Architecture flow → Trade-off):**

- **Problem:** <1–2 lines>
- **Why the naive way fails:** <1–2 lines on why a simple beginner implementation would break>
- **Step-by-step flow:** <4–5 bullet points walking from entry point → validation → core logic → DB/queue → output, naming actual files/functions>
- **Key engineering trade-off:** <1–2 lines>

## 3. File-by-File Call Graph (How It Connects)

```text
[Entry: path/a.ts]
   └── calls funcB() in [path/b.ts] (validates & transforms payload)
         ├── writes to [Postgres: TableX]
         └── enqueues job via [path/c.ts] ──► picked up by [path/d.ts]
```
```

## 4. Key Architectural Decisions & Trade-offs

| Decision in Code           | Naive / Alternative Option | Why We Chose This              | The Trade-off / Cost                    |
| -------------------------- | -------------------------- | ------------------------------ | --------------------------------------- |
| <e.g., BullMQ async queue> | <Send email inline in API> | <Fast response + auto retries> | <Requires Redis + eventual consistency> |

## 5. Failure Modes ("What If Things Break?")

- **If <Dependency/DB/API> is down or slow:** <exact behavior in code + line reference>
- **If the user double-clicks / sends duplicate requests:** <how idempotency/constraints handle it>
- **If invalid/malicious data is sent:** <where validation catches it>

## 6. Honest Code Audit ("What I Would Improve in v2")

> _Interviewers love when you can critique your own project like a senior engineer._

- **Current limitation 1:** <honest weakness spotted during lessons> → **How I'd fix it:** <concrete solution>
- **Current limitation 2:** <scale or edge-case gap> → **How I'd fix it:** <concrete solution>

## 7. Tricky Syntax & Jargon Used in This Feature

- **`<syntax or term 1>`:** <plain-English 1-line explanation>
- **`<syntax or term 2>`:** <plain-English 1-line explanation>

## 8. Topics to Revise Before Your Interview 🟡

- <Specific concept or file from PROGRESS.md where the learner was partial/shaky, with a 1-line reminder>

```

After saving/showing `INTERVIEW_NOTES.md`, invite them to **Step 3 (2-Minute Spoken Practice)** or **Step 4 (Mock Interview Q&A)**.

---

## Step 3: Two-minute spoken teach-back

Ask the learner:
> *"Imagine we are in a real interview right now and I ask: **'Walk me through how you designed and built <feature>.'** Without copy-pasting from the notes above, type out how you would explain it out loud in about 150–250 words."*

**Grade their response gently against 5 senior-interviewer signals:**
1. **Started with the Problem:** Did they explain *why* before diving into file names?
2. **Explained the "Why" (Naive vs. Real):** Did they mention why this architecture was needed?
3. **Clear Data Flow:** Could an interviewer visualize the request moving through the system without seeing the screen?
4. **Accuracy:** Did they describe the async/DB/auth behavior accurately?
5. **Ownership & Honesty:** Did they sound natural and aware of trade-offs?

Give brief, encouraging feedback and show a **polished version of their own paragraph** (keeping their natural phrasing and voice, just tightening the technical clarity).

---

## Step 4: Mock interview ladder

Run a live mock interview **one question at a time** (pick 4–6 questions tailored to the feature, escalating from junior to mid/senior level):

1. **The Flow Check:** *"Walk me through what happens under the hood when [specific user action] occurs."*
2. **The 'Why' Probe:** *"In `file.ts`, you used [pattern/library]. Why didn't you just [simpler naive alternative]?"*
3. **The Failure Probe:** *"Look at step X—what happens if [database / Redis / third-party API] throws an error or times out right on that line?"*
4. **The Concurrency / Edge Case Probe:** *"What happens if two users (or a double-click) trigger this exact endpoint at the exact same millisecond?"*
5. **The Modification Probe:** *"Product comes to you tomorrow and wants to add [realistic new requirement]. Which exact files and functions would you touch, and what would you change?"*
6. **The Scale / Critique Probe:** *"If traffic on this feature grew 100× next month, what is the very first bottleneck that would break, and how would you redesign it?"*

**Coaching rules during the mock interview:**
- After each answer, give a quick verdict (**Strong hire signal**, **Good start—needs one detail**, or **Careful—interview trap!**) followed by a crisp 2–3 sentence model answer.
- If the learner says *"I don't know"*, praise their honesty (guessing wildly in an interview is a major red flag!) and teach them how to frame a thoughtful engineering answer even when unsure.

---

## Spaced review

Wrap up by congratulating them and suggesting a spaced-repetition check-in:
> *"You've got your `INTERVIEW_NOTES.md` saved. Come back in 3–4 days and tell me `quiz me on <feature>`—we'll do a quick 5-minute rapid-fire round focusing on the `🟡` areas so this stays sharp for your interviews!"*
```
