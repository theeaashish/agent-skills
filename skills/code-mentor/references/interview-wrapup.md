# Interview wrap-up

Run this after the last lesson. It turns "I followed the lessons" into "I can explain and defend this feature". Do the four steps as separate responses, one at a time, waiting for the learner between them.

## Contents
- Step 1: End-to-end trace
- Step 2: Two-minute teach-back
- Step 3: Mock interview
- Step 4: INTERVIEW_NOTES.md
- Spaced review

## Step 1: End-to-end trace

Give the learner one concrete scenario (for example "a new user signs up with this email"). Ask them to narrate what happens from the first event to the final result, naming files and functions as they go. Don't help until they finish or get stuck.

Then respond with: what they covered correctly, the steps they skipped or got wrong (link each to its lesson number), and one clean version of the full trace in a compact numbered list.

## Step 2: Two-minute teach-back

Ask: "Explain this feature as if to an interviewer. Cover the problem, your approach, and one trade-off. Aim for about two minutes spoken (roughly 250–300 words)."

Give feedback against this rubric (mention only what's useful):

- **Problem first:** did they open with why, not how?
- **Structure:** could someone follow the flow without seeing the code?
- **Accuracy:** anything wrong or overstated?
- **A real decision:** did they name one choice and its trade-off?
- **Honesty:** did they own limits instead of glossing over them?

Offer a tightened version of their own answer, keeping their words where possible so it still sounds like them.

## Step 3: Mock interview

Ask one question at a time, escalating in difficulty. After each answer give brief feedback and a model answer in two or three sentences. A typical ladder (choose 5–8 that fit the feature):

1. **What:** "Walk me through what this feature does."
2. **Why:** "Why did you choose <technology/approach> here?"
3. **Alternatives:** "What else could you have used, and why not?"
4. **Failure:** "What happens if <dependency> is down, slow, or returns garbage?"
5. **Edge case:** "What if the same request arrives twice or two arrive at once?"
6. **Scale:** "What breaks first if traffic grows 100×?"
7. **Security or correctness:** "How could this be abused or produce wrong data?"
8. **Improve:** "What would you change with another week?"

If the learner is honest about not knowing, praise that and teach the answer. Guessing confidently is the habit to unlearn.

## Step 4: INTERVIEW_NOTES.md

Write `.learning/<feature-slug>/INTERVIEW_NOTES.md` (or output it in chat if there's no filesystem), based on what the learner actually demonstrated. Mark shaky areas with 🟡 so they know what to revise.

```markdown
# Interview notes: <Feature name>

## 30-second pitch
<3 sentences: problem, approach, result>

## 2-minute walkthrough
<the tightened teach-back>

## Architecture in one glance
<small flow diagram>

## Key decisions and trade-offs
| Decision | Alternative | Why this | Trade-off |
|---|---|---|---|

## Failure modes and how they're handled
- <failure> → <behavior>

## What I'd improve
- <honest limitation → concrete improvement>

## Likely questions (with crisp answers)
1. Q: ... A: ...

## Jargon I should be able to define
- <term>: <one-line definition>

## Revise before the interview 🟡
- <topics that were shaky, with lesson numbers>
```

## Spaced review

Close by suggesting the learner return in a few days and say "quiz me on <feature>", then again in about two weeks. Review mode (see SKILL.md) starts from the revisit list, so the shaky topics get the most attention.
