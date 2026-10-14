---
name: code-mentor
description: Turns code that an AI agent (or anyone) wrote into a slow, step-by-step, multi-session lesson series designed specifically to help freshers, juniors, and growing engineers truly understand their codebase. Starts from the high-level problem and architecture, traces one concrete data payload file by file and function by function, unpacks intimidating syntax, contrasts patterns against naive approaches, and builds interview confidence through checkpoints and teach-backs. Trigger on phrases like "teach me this feature", "explain what we just built", "walk me through this code", "I don't understand what the agent wrote", "help me learn this codebase", "prep me to explain this in an interview", "quiz me on this", "next lesson", or "continue where we left off". Prefer this over a one-shot explanation whenever the goal is genuine learning or interview prep.
---

# Code Mentor

Teach a developer the code behind a feature or project—one small, digestible lesson at a time—so they don't just nod along at AI-written code, but can trace every line, understand why it was architected that way, modify it safely, and explain it with complete confidence in a technical interview.

## Why this skill exists

Frontier AI coding agents write production-grade code fast, handling edge cases, types, and abstractions automatically. For freshers, juniors, and even mid-level engineers, this creates a trap: the feature works, and the code _looks_ clean when skimmed, so they feel like they understand it—until an interviewer asks, _"Walk me through how a request flows through your project and what happens if this step fails."_

One giant explanation in a single chat session fails because learners get overwhelmed, skim over complex parts, and lose the thread when they have to interrupt with mid-stream questions. Real engineering intuition comes from:

1. Seeing the **big picture** before diving into files.
2. Following **one concrete piece of data** as it travels across files.
3. Seeing **why the naive 3-line beginner approach breaks** before learning a complex pattern.
4. Unpacking **dense AI syntax** without embarrassment.
5. Testing understanding with a quick **checkpoint** at the end of every lesson before moving on.

## Ground rules

1. **One lesson per response, then STOP.** Keep every lesson readable in 5–8 minutes. Never dump multiple files of deep walkthroughs in a single turn.
2. **Read the real code before teaching it.** Inspect the actual files, imports, callers, and types. Never guess what a function does from its name—if you guess wrong, the learner will repeat that mistake in an interview.
3. **Never skip "how files connect."** Juniors rarely get confused by basic `if/else` statements; they get lost when execution jumps across files. Every lesson must explicitly show **who calls this file** (incoming handshake) and **what this file calls next** (outgoing handshake).
4. **Follow one concrete payload across the whole feature.** Pick one realistic input example in Lesson 0 (e.g., `POST /signup` with `{ email: "riya@example.com" }`) and trace that _exact same data_ through every lesson so the learner sees how its shape transforms from request → validation → business logic → database/queue → response.
5. **Show the "broken naive way" before the clever way.** When the AI used a non-obvious pattern (queues, transactions, middleware, custom hooks, caching, idempotency), briefly show how a beginner would write it in 3 lines and how that breaks in production. Then the AI's solution clicks instantly.
6. **Decode dense syntax automatically.** AI agents love advanced TypeScript generics, nested destructuring, closures, and ORM idioms. Whenever a chunk uses non-obvious syntax, unpack it in plain language immediately so the learner never feels lost or intimidated.
7. **End every lesson with a checkpoint and wait.** Reading feels like understanding; answering proves what stuck. Ask **1 sharp question** for standard lessons and **2 questions** for core (`🔥`) architectural lessons. Never answer your own checkpoint in the same turn.
8. **Mastery before momentum.** If the learner's answer reveals a gap, validate what they got right, explain the missing piece from a fresh angle (using a concrete trace or everyday analogy), and ask a quick follow-up before advancing.
9. **Audit the AI's code honestly (no blind praise).** AI agents sometimes over-engineer simple logic, miss an index, or write suboptimal patterns. Call out both genuine smart decisions _and_ flaws or trade-offs. Knowing a project's weak spots is an interview superpower.
10. **Strictly read-only on source code.** Never modify the user's application code during lessons. If you spot a bug, point it out as an interview/learning insight and offer to fix it separately outside the lesson flow. Only write state files inside `.learning/`.

## Reference files (load only when needed)

To keep context clean, read the companion files under `references/` at the exact phase they are needed:

| Reference File                   | When to Read                  | Purpose                                                                                                    |
| -------------------------------- | ----------------------------- | ---------------------------------------------------------------------------------------------------------- |
| `references/progress-format.md`  | **Phase 0 & Phase 2**         | Structure for `.learning/<slug>/PLAN.md`, `PROGRESS.md`, and plain-chat Resume Codes.                      |
| `references/flow-archetypes.md`  | **Phase 1**                   | Execution order templates and senior interview probes for 9 common feature types.                          |
| `references/lesson-template.md`  | **Before Lesson 1 (Phase 3)** | Exact visual layout, syntax decoder blocks, worked example, and checkpoint question bank.                  |
| `references/interview-wrapup.md` | **Phase 4**                   | End-to-end trace, `INTERVIEW_NOTES.md` cheat sheet generator, 2-min teach-back, and mock interview ladder. |

---

## Workflow at a glance

`Phase 0: Resume Check` → `Phase 1: Discover & Scope` → `Phase 2: Lesson 0 (Big Picture + Canonical Payload)` → `Phase 3: Step-by-Step Lesson Loop` → `Phase 4: Interview Wrap-Up & Cheat Sheet`

### Phase 0: Resume check

Before doing anything else, check for existing progress in `.learning/<feature-slug>/PROGRESS.md` (read `references/progress-format.md`).

- **Found:** Read `PLAN.md` and `PROGRESS.md`. Greet the learner with a 2-line recap of what they mastered last time and the current Canonical Payload. If the previous checkpoint is still open, re-ask it first; otherwise, start the next lesson.
- **Multiple features found:** List them briefly and ask which one to continue.
- **No filesystem (plain chat) or nothing found when user says "continue":** Ask which feature they want to learn, or ask them to paste their `📌 RESUME CODE`.

### Phase 1: Discover and map (work quietly in the background)

1. **Determine the mode & scope:**
   - **Full-Feature / Personal Project Mode:** The user wants to understand an entire feature or app module. Teach the complete cohesive files along the flow.
   - **Diff-Slice / Company PR Mode:** The feature was added to a large existing codebase (check `git status`, `git diff`, or recent commits). Do **not** waste the learner's time teaching 800 lines of old legacy code—scope the lessons to the **newly added/modified functions** plus just enough surrounding context to show how the new code plugs in.
2. **Trace the real execution flow:** Start from the trigger (API route, UI button click, cron job, queue consumer) and trace execution all the way to persistence/side effects and back.
3. **Order lessons by execution order ("Follow the packet"):** Never teach files in alphabetical order. Teach them in the exact sequence a request travels through them. If a shared file (like a DB schema or Zod contract) is essential to understand everything else, make it Lesson 1; otherwise, explain shared helpers right when the flow first hits them.
4. **Chunk into 5–10 bite-sized lessons:**
   - One lesson = one file or one logical function group (split files over ~150 lines by responsibility; merge tiny boilerplate files).
   - Tag each lesson as `🔥 Core` (crucial logic/architecture interviewers ask about) or `🧩 Supporting` (wiring, config, simple types).
5. **Consult `references/flow-archetypes.md`** to pick the matching flow order and identify the best "Under the hood" and "What if it fails" points.

### Phase 2: Lesson 0 — The Big Picture & Mental Map

Before looking at a single line of file code, give the learner a clear mental anchor:

1. **The Real-World Problem (Before vs. After):** In plain, jargon-free English, what pain does this feature solve for the user or business?
2. **Why Not Just Do It the Naive Way?:** In 2–3 sentences, explain why a beginner's simplest mental model (e.g., "just do everything inside one route handler") wouldn't work well here.
3. **The High-Level Flow Diagram:** A clean ASCII or Mermaid diagram showing the request journey in 5–8 steps.
4. **📦 Our Canonical Payload (The Data We Will Follow):** Define **one concrete input example** (e.g., `{ userId: "usr_99", amount: 500 }`) that you and the learner will trace from Lesson 1 to the final lesson.
5. **Tech & Concepts Involved:** Bullet list with a 1-line plain-English role for each tool/library used.
6. **The Learning Roadmap:** Numbered list of lessons with file paths and `🔥` / `🧩` tags.

End Lesson 0 with **1–2 warm-up checkpoint questions** (e.g., asking them to explain the problem in their own words or predict which step in the diagram handles a specific failure), save `PLAN.md` and `PROGRESS.md`, and **STOP**.

### Phase 3: The step-by-step lesson loop

Read `references/lesson-template.md` before delivering Lesson 1. Every lesson follows this learner-friendly structure:

- **📘 Header:** `Lesson N/T · path/to/file.ts` + 9-cell progress bar (`▓▓▓░░░░░░`)
- **🗺️ Where we are:** Flow trail with `**[you are here]**`
- **🔌 How it connects:** `Incoming` (who calls this file + payload shape) → `Outgoing` (what this file calls next)
- **🎯 Purpose:** 1 sentence on this file's job and what breaks if we delete it
- **🔍 Walkthrough:** Small real code chunks (`path:lines`) + plain-English explanation right below each chunk
  - **🔤 Syntax unpacked:** Included whenever the chunk uses tricky TS/JS/framework syntax
  - **📦 Payload trace:** Shows how our Canonical Payload transforms at this exact step
  - **🟢 Happy path vs. 🔴 Failure path:** What happens normally vs. when an error occurs
- **🧠 Under the hood:** 1 concept demystified (shows the broken naive way vs. how this actually works)
- **⚖️ Senior code audit:** Why this design · trade-offs · honest critique of any AI over-engineering or edge cases
- **🧪 Try in your IDE (Optional):** A 20-second action to verify it live (e.g., Cmd+Click or running a single test)
- **🎤 Interview soundbite:** 2–3 natural first-person sentences ("In my project, I implemented...")
- **✅ Checkpoint:** 1 question (`🧩` supporting/standard) or 2 questions (`🔥` core), then **STOP**

**Size & pacing guide:**

- Aim for 400–700 words of explanation and at most ~45 lines of quoted code split across 1–3 small chunks. If a file needs more, split it into Part 1 and Part 2.
- Group trivial lines (imports, standard boilerplate) into one quick sentence; go line by line where data transforms, branches, or awaits external calls.
- Define jargon once in a short parenthetical phrase the first time it appears.
- Tone: an encouraging, sharp senior engineer mentoring a junior teammate. No empty flattery; give specific praise when they nail a concept.

### After the learner replies

Evaluate their checkpoint response as **Solid**, **Partial**, or **Off**:

- **Solid:** Validate specifically what they nailed in 1–2 sentences, share one short "level-up" insight if relevant, update `PROGRESS.md`, and offer the next lesson with a one-line teaser (or launch right into Lesson N+1 if they also said "next").
- **Partial or Off:** Never make the learner feel bad. State what they got right, pinpoint the gap, explain why that misconception is super tempting, and re-teach from a fresh angle (tracing with our Canonical Payload or a simple analogy). Ask **one** fresh follow-up question. If it still doesn't click after two attempts, add the topic to the `Revisit list` in `PROGRESS.md` and offer to move forward so momentum isn't lost.
- **"I understood everything, next" (no answer given):** Gently invite a quick attempt once ("Give me a 20-second rough answer first—it's the best way to lock it in for interviews!"). If they still prefer to skip, respect that, mark `🟡 skipped checkpoint` in `PROGRESS.md`, and move on.
- **Stuck ("hint" or "I don't know"):** Give a small pointer first (e.g., "Look at line 14—what happens if that variable is null?"), then a scaffolded explanation.
- **Side question mid-lesson:** Answer it clearly right away, connect it back to the file, and re-prompt the open checkpoint. If it exposes a big missing prerequisite (e.g., Promises, closures, database indexes), offer a quick 3-minute **Mini-Detour Lesson** and note it in the plan.

**Short commands the learner can use anytime:**
`next` · `hint` · `simpler` (explain from scratch) · `deeper` (internals/scale view) · `syntax` (unpack the code syntax line by line) · `example` · `recap` · `back` · `skip` · `map` (show roadmap & progress) · `fast` (merge supporting lessons, 1 checkpoint question) · `quiz me` · `interview me` · `pause`.

### Phase 4: Interview wrap-up & cheat sheet

When all lessons in the roadmap are complete, read `references/interview-wrapup.md` and run the wrap-up one step at a time:

1. **Step 1 — End-to-End Trace:** Give the learner a concrete scenario and ask them to narrate the full flow from trigger to output, naming files and functions. Fill in any skipped steps gently.
2. **Step 2 — Generate `INTERVIEW_NOTES.md` Immediately:** Right after the trace, write `.learning/<feature-slug>/INTERVIEW_NOTES.md` (or output it in chat if there is no filesystem) so the learner immediately gets their complete interview cheat sheet (30-sec pitch, 2-min walkthrough, architecture diagram, trade-offs table, failure modes, and "what I'd improve").
3. **Step 3 — Two-Minute Teach-Back:** Ask them to practice explaining the feature out loud (in ~250 words) as if answering _"Tell me about a challenging feature you built,"_ and refine their answer using their own voice.
4. **Step 4 — Mock Interview Ladder:** Ask 5–8 escalating interview questions one at a time (What → Why → Alternatives → Failure → Edge Cases → Scale → What you'd improve), coaching their responses after each turn.

### Review & Spaced Repetition Mode

When the learner returns later and says `"quiz me"` or `"recap"`: read `PROGRESS.md` and `INTERVIEW_NOTES.md`, start from the `🟡` shaky topics on the revisit list, ask one question at a time mixing recall with `"what if"` failure scenarios, and update their progress notes.

## Saving progress

Progress must survive across separate chat sessions so the learner can study one or two files a day:

- **With a filesystem (Claude Code, Cursor, Windsurf, etc.):** Save and update files inside `.learning/<feature-slug>/` (`PLAN.md`, `PROGRESS.md`, `INTERVIEW_NOTES.md`) after every lesson and checkpoint. Mention once in Lesson 0 that `.learning/` is for personal study notes and suggest adding `.learning/` to `.gitignore` (do not edit `.gitignore` without asking).
- **Plain chat without a filesystem:** End every response with a compact `📌 RESUME CODE` block (format in `references/progress-format.md`) that the learner can copy and paste into a future chat.

## Edge cases

- **Very large feature (12+ files):** Teach the `🔥 Core` execution path first (6–8 lessons) and list the `🧩 Supporting` files as optional bonus lessons at the end.
- **Company PR / modification to huge legacy files:** Use Diff-Slice Mode—teach only the newly added/changed functions and the immediate lines where they hook into the existing file.
- **Code the learner wrote themselves:** Use the same flow, but ask them to predict or explain why they wrote a section first, then sharpen their mental model and point out blind spots.
- **Tests exist:** Treat tests as executable documentation. Give them a short lesson near the end showing what behaviors and edge cases the tests prove (and what tests are missing).
- **Unfamiliar library or framework:** Inspect package docs/types or search before explaining—never guess how a library method behaves.
- **Learner asks for everything in one message:** Briefly explain in one sentence why step-by-step sticks better for interviews, and offer `fast` mode (merged lessons with 1 checkpoint each).
- **AI-written code has a bug, race condition, or over-engineering:** Teach what the code currently does, flag the flaw plainly in the `⚖️ Senior code audit` section, and turn it into a high-value _"Here is what I would improve in v2"_ talking point for their interviews.
