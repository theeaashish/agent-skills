---
name: debugging
description: Investigates, diagnoses, fixes, and verifies software bugs and unexplained behavior using evidence rather than guesswork. Use whenever a user reports runtime errors, exceptions, wrong results, regressions, failing tests or builds, flaky behavior, hangs, races, async or lifecycle defects, integration failures, environment-specific differences, production incidents, or failures spanning services and queues. Apply this skill before proposing patches, retries, rewrites, or dependency blame, even when the request sounds like a quick fix. Debugging leads the causal investigation of concrete failures and may be combined with specialized skills for localized domain analysis or implementation. Use another skill alone only when there is no concrete failure to diagnose, such as pure optimization, general test strategy, or observability architecture.
---

# Debugging

## Purpose

Find the causal explanation for an observed software failure, correct it with the smallest safe change, and demonstrate that the original failure is resolved without unjustified collateral behavior changes.

Investigation comes before patching. A plausible edit is not a diagnosis, and a passing command after an edit is not by itself proof of a fix.

## Scope and boundaries

This skill owns the general reasoning process for concrete failures:

- turn a report into a precise, falsifiable failure statement;
- reproduce or observe the failure and preserve a baseline;
- gather evidence and separate it from interpretation;
- trace control flow, data flow, state, timing, and boundaries;
- localize the first point where actual behavior diverges from expected behavior;
- test competing causal hypotheses;
- select and implement the narrowest justified fix;
- verify the original failure, relevant adjacent behavior, and residual risk.

Coordinate rather than duplicate adjacent skills:

- **Testing** owns broad test strategy, framework design, and suite architecture. Use tests here as experiments and regression protection.
- **Observability** owns telemetry architecture, dashboards, retention, and alerting. Add only the targeted diagnostics needed to discriminate hypotheses, then preserve generally useful signals.
- **Performance engineering** owns profiling and optimization when behavior is correct but inefficient. This skill owns causal investigation when latency, resource use, or throughput is itself an unexpected failure.
- **Reliability and distributed systems** own resilience policy, SLOs, topology, consistency models, and architectural redesign. This skill traces a concrete incident across those boundaries and hands off systemic changes when appropriate.
- **Database optimization** owns slow-query analysis, indexing, schema efficiency, pagination, tuning, and scaling. For concrete query-correctness, transaction, persistence, or data-integrity failures, this skill leads the investigation and may co-invoke the database specialist.
- **Infrastructure, framework, and language specialists** own domain-specific analysis and implementation. Invoke them during or after localization without surrendering evidence discipline.
- **Coding standards and component-system patterns** govern implementation quality after the cause is understood; they do not replace diagnosis.

Do not expand a bug fix into broad cleanup, optimization, redesign, or dependency migration unless the evidence shows it is necessary and the user accepts that scope.

## Investigation vocabulary

Keep these categories distinct in notes and reports:

- **Symptom**: The user-visible or externally reported failure, such as a crash, wrong value, timeout, or missing event.
- **Observation**: A fact directly supported by an artifact or experiment, including its conditions and source.
- **Hypothesis**: A proposed mechanism that could explain observations and makes testable predictions.
- **Root cause**: The actionable causal mechanism within the investigation scope that is sufficient to explain the failure. Distinguish it from the triggering event and contributing conditions; real failures may have more than one necessary cause.
- **Fix**: A change intended to remove or correctly handle the cause. A mitigation that only reduces impact is not yet a fix.
- **Verified result**: Evidence collected after the change under stated conditions showing what now works and what was checked for regressions.

Use calibrated language:

- Say **observed** when directly evidenced.
- Say **supports**, **weakens**, or **rules out** for hypothesis updates.
- Say **probable cause** when evidence is strong but causal confirmation is unavailable.
- Say **confirmed cause** only when the mechanism explains the evidence and a discriminating experiment, intervention, or controlled comparison behaves as predicted.
- Say **unresolved** when the available evidence cannot distinguish the remaining explanations.

Never turn temporal proximity, correlation, a stack-frame location, or a recent change into a confirmed diagnosis by wording alone.

## Core operating rules

1. **Establish expected behavior first.** Derive it from requirements, contracts, tests, documentation, invariants, prior known-good behavior, or user confirmation. Do not debug against an unstated assumption.
2. **Preserve a failing baseline.** Capture the exact command, request, input, environment, version, state, timing, and output needed to compare later results.
3. **Prefer discriminating evidence.** Gather information that changes the relative likelihood of competing explanations, not logs collected merely because they are easy to add.
4. **Find the first incorrect state.** Later exceptions and malformed outputs are often downstream consequences. Trace backward until actual and expected state first diverge.
5. **Reduce the search space deliberately.** Split by component, boundary, time, data cohort, environment, change range, or control-flow branch, then test the more informative side.
6. **Change one meaningful variable at a time when attribution matters.** Keep experimental changes separate from the final fix. If variables cannot be isolated, state that limitation.
7. **Try to falsify the leading hypothesis.** When ambiguity remains, seek counterexamples and compare credible alternatives. Do not invent a competitor after direct evidence has already isolated a straightforward cause.
8. **Treat diagnostics as evidence-bearing code.** Make instrumentation targeted, safe, and attributable. Remove temporary instrumentation after use unless its ongoing value, privacy, volume, and ownership justify keeping it; preserve useful error context rather than silencing it.
9. **Fix the violated invariant at the boundary that owns it.** Avoid patches that merely suppress the visible symptom farther downstream.
10. **Verify claims, not intentions.** Report only commands and observations actually performed. State what could not be checked.

## Operational workflow

Scale the depth and formality to ambiguity, impact, and risk. A simple deterministic defect may need only a failing baseline, a mechanistic trace, a focused correction, and verification; uncertain or high-risk failures need explicit competing hypotheses and stronger controls. In all cases, keep symptom, causal claim, change, and verified result distinct.

### 1. Triage safety and impact

Before reproducing or editing:

- assess user impact, data integrity, security exposure, financial or operational risk, and whether the failure is ongoing;
- distinguish urgent containment from diagnosis;
- protect user work and preserve logs, traces, dumps, inputs, timestamps, versions, and correlation identifiers that may expire;
- avoid destructive experiments, broad restarts, cache clears, or state mutation until their evidentiary and operational effects are understood;
- if immediate mitigation is necessary, prefer reversible controls and label the action as **mitigation**, not **root-cause fix**.

For severe incidents, stabilize first when delay would increase harm, while recording exactly what changed so the investigation baseline is not lost.

### 2. Define the failure precisely

Create a compact failure statement:

> Under [conditions], [action or event] produces [observed result], but [source of expectation] requires [expected result]. The impact is [scope], first known at [time/version], and reproduction status is [status].

Record:

- exact symptom and full error text where available;
- expected behavior and its source;
- affected and unaffected cases;
- first known bad and last known good versions or times;
- input shape, state, permissions, feature flags, topology, and dependencies;
- environment and configuration differences;
- frequency and timing pattern;
- recent relevant changes, without assuming the newest change is causal.

Use an explicit reproduction status: `unattempted`, `reproducible`, `intermittent`, `production-only`, `not reproduced`, or `blocked`. “Not reproduced” does not mean “cannot happen.”

### 3. Reproduce or improve observability

#### If reproducible

- run the narrowest authentic failing path and save its result before editing;
- minimize the reproducer while preserving the failure mechanism;
- remove irrelevant inputs, components, or steps incrementally rather than replacing the scenario with a different one;
- make uncontrolled state explicit where practical;
- prefer a focused automated regression test when it faithfully represents the failure.

A failing test is evidence, not automatically the truth: verify that its expectation matches the intended contract and that the test is exercising the claimed path.

#### If intermittent, nondeterministic, or production-only

- preserve per-occurrence context before aggregate summaries hide it;
- compare affected and unaffected cohorts by version, host, tenant, input, load, timing, dependency, configuration, and execution path;
- inspect sources of nondeterminism such as ordering, shared state, scheduling, clocks, randomness, retries, cancellation, resource pressure, cache state, and external responses;
- add the smallest privacy-safe instrumentation that can distinguish named hypotheses;
- use correlation or causation identifiers across process, service, queue, job, and retry boundaries;
- repeat trials under controlled conditions and report counts and conditions, not simply “flaky” or “passed”;
- do not treat a quiet observation window as proof of resolution without considering the baseline occurrence rate and exposure.

If faithful reproduction is impossible, proceed by increasing evidence quality, not by lowering the standard for causal claims.

### 4. Map the failing path

Trace the relevant execution rather than reading the codebase indiscriminately:

1. Start from the observable failure boundary.
2. Identify the inputs, state, and contracts entering that boundary.
3. Follow control flow and data transformations backward and forward.
4. Mark where expected invariants are established, transformed, persisted, transmitted, and consumed.
5. Compare actual and expected values or events at strategic checkpoints.
6. Stop at the earliest divergence that can causally explain downstream symptoms, then investigate why it occurs.

For distributed or asynchronous workflows, reconstruct a timeline across boundaries. Account for:

- message creation, serialization, transport, delivery, acknowledgement, and consumption;
- duplicate, delayed, missing, or out-of-order events;
- retries, timeouts, cancellation, backpressure, and partial success;
- idempotency and deduplication behavior;
- version skew and contract compatibility;
- context loss between request, background job, callback, and compensation path;
- clock differences and misleading timestamp order.

The component that reports an error may only be the first observer, not the source.

#### Conditional investigation lenses

Apply only the lenses relevant to the reported failure:

- **Build and test failures**: Isolate the first actionable failing stage; later errors may be cascades. Compare clean and incremental runs, local and automation environments, dependency and toolchain resolution, generated artifacts, and runner configuration. For flaky tests, control order, isolation, seed, time, shared state, and external dependencies before changing expectations.
- **State and lifecycle failures**: Reconstruct initialization, readiness, use, cancellation, shutdown, disposal, reconnect, and re-entry transitions. Check ownership, duplicate registration, stale callbacks, leaked state, and use-before-ready or use-after-disposal conditions.
- **Environment and configuration differences**: Compare effective runtime state, not only checked-in configuration. Include resolved settings and flags, permissions, secret availability, deployed artifact identity, dependency versions, architecture, locale and time zone, and persistent state. Compare an affected environment with the nearest known-good cohort.
- **Integration failures**: Inspect the producer output, boundary representation, transport or persistence transformation, and consumer input separately. Compare contract versions and test each side independently before assigning fault to the dependency.
- **Regressions**: Narrow the change interval between last known good and first known bad, then test candidate changes against the causal mechanism. Revert or blame a change only when behavior follows the predicted comparison.

### 5. Build and test hypotheses

Maintain a lightweight hypothesis ledger when the cause is not immediately demonstrated:

| Hypothesis | Evidence for | Evidence against | Distinguishing prediction | Next safe experiment | Status |
|---|---|---|---|---|---|

Rank hypotheses by explanatory power and consistency with evidence, not familiarity. Include credible alternatives such as malformed historical data, stale artifacts, configuration or permission differences, environment drift, lifecycle ordering, ownership violations, dependency contract changes, or interacting causes.

Choose experiments that are:

- **discriminating**: competing hypotheses predict different outcomes;
- **narrow**: few uncontrolled variables change;
- **safe**: no disproportionate production or data risk;
- **reversible**: temporary changes can be removed cleanly;
- **observable**: the result has a clear interpretation.

Update the ledger after each result. Discard falsified explanations. If an experiment supports several hypotheses equally, it did not localize the cause; choose a sharper next test. Prefer the next safe experiment with the highest likely information value. Stop with an evidence handoff when further investigation is unsafe, unavailable, or disproportionate to the remaining risk.

### 6. Confirm the cause

Before making a behavioral correction, establish a causal account proportionate to the failure's ambiguity and risk:

- What first becomes incorrect?
- Under what conditions does it become incorrect?
- Through what mechanism does that state create the reported symptom?
- Why are relevant unaffected cases unaffected?
- When credible alternatives remain, which observation or intervention distinguishes this account from them?

Confirmation can come from a minimal reproducer, a controlled intervention, a known-good versus known-bad comparison, history narrowing, invariant checks, or equivalent evidence. For a straightforward deterministic defect, direct reproduction plus a mechanistic trace may be sufficient. For an ambiguous or high-impact failure, the combined evidence should explain the failure scope and meaningful counterevidence.

If confirmation is blocked, do not invent certainty. Report a probable cause or unresolved hypotheses and identify the missing evidence. A narrow, reversible correction may still be justified when delay carries greater risk, but make the uncertainty, authorization, rollback, and verification plan explicit and limit the causal claim accordingly.

### 7. Select the smallest safe fix

“Smallest” means the narrowest behavioral correction at the ownership boundary of the violated invariant, not merely the fewest changed lines.

Compare candidate fixes by:

- whether they remove or correctly handle the established cause, including an explicitly labeled probable cause when action before confirmation is justified;
- blast radius and compatibility with existing contracts;
- behavior for historical, malformed, partial, or concurrent state;
- migration and deployment implications;
- reversibility and rollback path;
- new complexity or failure modes;
- ability to verify the result;
- preservation of useful diagnostics.

Prefer restoring an existing invariant or contract. Broaden the change only when the causal mechanism crosses a boundary and a local fix would leave the system inconsistent.

Keep diagnosis and cleanup separate. Avoid opportunistic refactors, broad rewrites, upgrades, formatting churn, or unrelated fixes that obscure attribution and review.

### 8. Implement without masking

During implementation:

- preserve existing error context or improve it when it is insufficient;
- keep assertions and contract checks that reveal invalid state;
- handle errors only where the code can add context, recover correctly, compensate, or deliberately translate the contract;
- make retry, fallback, and timeout behavior conditional on a demonstrated transient failure model;
- ensure concurrency fixes establish an explicit ownership, ordering, atomicity, or synchronization rule rather than merely altering timing;
- update tests only when the intended behavior is established independently; never weaken a correct expectation just to obtain a passing suite;
- document non-obvious compatibility or rollout constraints, not the obvious mechanics of the patch.

### 9. Verify in layers

Verification should prove both **fix effectiveness** and **absence of likely regressions**.

1. **Original failure**: Re-run the exact baseline or closest faithful reproducer and show fail-before/pass-after where possible.
2. **Causal path**: Add or run a focused regression check that would fail if the mechanism returned.
3. **Boundary behavior**: Check nearby inputs, negative cases, state transitions, error paths, and alternate callers sharing the changed contract.
4. **Integration behavior**: Verify relevant boundaries when serialization, persistence, processes, services, queues, or external systems are involved.
5. **Broader confidence**: Run proportionate builds, static checks, and test suites, beginning with the changed area and expanding according to blast radius.
6. **Operational behavior**: For intermittent or production failures, compare repeated trials or canary signals under equivalent exposure and define observation and rollback criteria.

A compilation success does not verify runtime behavior. A focused passing test does not establish broad safety. A broad suite passing without exercising the original failure does not prove the bug is fixed.

If verification is constrained, state exactly what ran, what passed or failed, what remains unverified, and the risk that follows.

### 10. Close with an evidence-based report

Keep the report proportional to the task, but include:

- **Failure**: expected versus observed behavior, impact, and reproduction status;
- **Evidence**: decisive observations and experiments, including relevant contradictory evidence;
- **Diagnosis**: confirmed or probable cause, causal mechanism, and confidence limits;
- **Change**: files or components changed and why this is the narrowest safe correction;
- **Verification**: exact checks run and their results;
- **Residuals**: remaining uncertainty, unverified paths, rollout/rollback needs, or follow-up work.

Do not make the chronology sound more certain than it was. Distinguish a verified fix, a mitigation, and an unresolved investigation.

## When evidence is incomplete

Remain useful without guessing or patching blindly. Produce an investigation handoff containing:

- confirmed facts and artifact locations;
- reproduction attempts and why they did not reproduce or were blocked;
- environment or access gaps;
- ranked hypotheses, each clearly labeled as inference;
- the cheapest safe observation or experiment that would distinguish each hypothesis;
- targeted instrumentation or concrete artifacts needed next;
- safe reversible mitigations, explicitly separated from fixes;
- the exact user, service owner, or specialist input required to continue.

A speculative code change is appropriate only when its uncertainty, risk, rollback, and verification plan are explicit and the situation justifies acting before confirmation. Prefer collecting the missing evidence.

## Production safety

In live systems:

- begin with read-only inspection and use the least privilege necessary;
- do not expose secrets, credentials, personal data, or sensitive payloads in diagnostics;
- consider telemetry sampling, retention, and clock limitations before trusting absence or ordering;
- bound diagnostic volume and performance impact;
- prefer canaries, feature controls, shadow comparisons, or limited cohorts for risky changes;
- preserve forensic evidence before restarts, cleanup, failover, or redeployment alters it;
- obtain explicit user or responsible-owner authorization before mutating production state, restarting services, changing flags or configuration, clearing data or caches, replaying events, redeploying, or executing recovery actions; reversibility does not replace authorization;
- define success signals, failure signals, observation windows, and rollback thresholds before rollout;
- separate emergency containment, recovery, corrective fix, and later prevention work.

Do not run destructive production experiments merely to gain confidence.

## Anti-patterns to prevent

Reject these common AI debugging failures:

- **Guess-and-patch**: editing based on the first plausible explanation before observing the failing path.
- **Shotgun debugging**: changing several variables, clearing state, reinstalling, upgrading, or rewriting simultaneously so attribution is lost.
- **Random logging**: adding large volumes of unstructured diagnostics without a hypothesis or privacy and volume controls.
- **Symptom suppression**: swallowing exceptions, returning defaults, weakening assertions, disabling checks, or hiding errors without restoring correctness.
- **Timing superstition**: adding arbitrary sleeps, delays, timeout increases, or retries to make races and transient failures less visible.
- **Dependency blame**: blaming a library, service, network, database, or environment without boundary evidence and contract comparison.
- **Test appeasement**: modifying tests solely to match current broken behavior, or deleting flaky tests instead of investigating nondeterminism.
- **Diagnostic rewrites**: performing broad refactors during investigation, making the cause and effect harder to isolate.
- **Single-cause fixation**: ignoring contradictions, contributing conditions, or plausible alternatives once an attractive theory appears.
- **Downstream patching**: validating or correcting corrupted state only where it crashes instead of finding where the invalid state originated.
- **Premature victory**: claiming “fixed” after compilation, one passing run, a quiet interval, or an unverified code edit.
- **Evidence destruction**: removing useful errors, logs, failing cases, artifacts, or user changes that are needed to understand the failure.

## Completion criteria

Treat the task as complete only when one of these outcomes is reported honestly:

- **Verified fix**: the cause is sufficiently established, the smallest safe correction is implemented, the original failure no longer reproduces under the baseline conditions, and proportionate adjacent checks pass.
- **Mitigated**: impact is reduced or contained, but the causal defect is not yet corrected or verification is incomplete.
- **Diagnosed, not fixed**: the cause is established, but implementation is blocked, out of scope, or requires a specialist or user decision.
- **Unresolved with next step**: evidence is insufficient, and the report identifies concrete discriminating evidence or access needed next.

Never collapse the last three outcomes into “fixed.”
