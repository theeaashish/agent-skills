---
name: performance-engineering
description: >-
  Use whenever a task asks to investigate, explain, improve, design for,
  benchmark, load-test, or verify software runtime performance: slow or degraded
  behavior, regressions, latency or tail latency, throughput, CPU, memory,
  allocation, storage or network I/O bottlenecks, concurrency contention or
  scaling, capacity, saturation, resource efficiency, scalability, or
  performance-sensitive design—even when the user only says “slow,” “does not
  scale,” “uses too much memory,” or “make it faster.” Provides the
  technology-agnostic method for representative measurement, bottleneck analysis,
  hypothesis testing, proportionate changes, and proof. Pair it with
  domain-specific skills for implementation details. Do not use for a
  correctness-only bug or generic cleanup with no performance objective.
---

# Performance Engineering

## Mission

Treat performance as an empirical property of a system under a specified workload, not as a visual property of code.

Translate a performance concern into a measurable objective, identify the limiting mechanism, make the smallest justified change, and verify the result under comparable conditions. Preserve correctness, reliability, operability, and maintainability while doing so.

A plausible optimization is still a hypothesis. Do not call it an improvement until evidence shows that it improved the intended outcome.

## Own the performance argument

This skill owns the reasoning chain from complaint to verified conclusion:

- define what “better” means;
- characterize the workload and environment;
- establish a baseline;
- localize the bottleneck;
- form and falsify hypotheses;
- compare candidate changes and tradeoffs;
- implement only when authorized and justified;
- verify local and end-to-end effects;
- report confidence, limitations, and bottleneck movement.

Compose with adjacent skills rather than replacing them:

| Area | Performance engineering owns | Adjacent skill owns |
|---|---|---|
| Debugging | Performance-specific causal investigation | General correctness failures and debugging techniques |
| Observability | The measurements needed to answer the current question | Telemetry platform, dashboard, and alert design |
| System design | Budgets, capacity models, and architecture choices justified by workload | Broader product and architecture design |
| Reliability | Overload, queueing, retry, timeout, and saturation effects on performance | Availability, resilience, incident response, and recovery |
| Testing | Performance experiment design and performance acceptance criteria | General test strategy and functional coverage |
| Domain optimization | End-to-end objective, evidence quality, and final verification | Technology-specific profiling, diagnosis, and implementation |

When a database, frontend, runtime, network, infrastructure, distributed-system, cache, queue, or AI/LLM specialist is available, use it for domain-specific technique. Retain ownership of the original performance objective and verify the specialist change at the system boundary that matters.

## Select the task mode

Identify the mode before acting because each mode needs a different conclusion.

| Mode | Primary question | Required outcome |
|---|---|---|
| Diagnosis | Why is observed behavior slow or degraded? | Bottleneck or ranked hypotheses tied to evidence; do not edit unless requested |
| Optimization | How should measured behavior be improved? | Justified change plus before/after verification |
| Scale design | What will meet an expected workload? | Explicit assumptions, budgets, capacity model, risks, and validation plan |
| Regression investigation | What changed between known-good and degraded behavior? | Controlled comparison that isolates the causal change or narrows the boundary |
| Proof | Did a specific optimization work? | Comparable results and a verdict: improved, unchanged, regressed, or inconclusive |

Treat the table as the primary mode and required conclusion, not as a mutually exclusive workflow. A “diagnose and fix” task can sequence diagnosis → optimization → proof when the user authorized all three.

Do not turn an investigation-only request into an unsolicited rewrite. Do not treat a design projection as proof, or a microbenchmark win as an end-to-end optimization.

## Core rules

1. **Define the outcome before optimizing.** Name the metric, affected path or operation, workload, scope, target, and guardrails. If no target exists, establish one with the user or label any inferred target.
2. **Measure before and after.** Capture a baseline before changing the variable under test whenever execution is possible.
3. **Treat leads as hypotheses.** Code inspection, complexity analysis, profiles, traces, and generic best practices can point toward a cause; none alone proves the proposed change will improve the user-visible objective.
4. **Distinguish activity from limitation.** A component can be busy without being the bottleneck. Look for evidence that it limits completed work or critical-path latency.
5. **Preserve semantics unless a tradeoff is explicit.** Faster but stale, lossy, less durable, less secure, or less reliable is a product decision, not a free optimization.
6. **Optimize the relevant system boundary.** A local gain matters only if it survives integration and does not create a larger downstream cost.
7. **State uncertainty.** Separate observed facts, inferences, assumptions, predictions, and verified results.
8. **Prefer proportionate changes.** Favor the simplest reversible change whose likely impact exceeds measurement noise and justifies its complexity and risk.
9. **Accept negative results.** “No meaningful change” and “inconclusive” are valid outcomes; do not rescue a failed hypothesis with unsupported claims.

## Scale rigor to impact and risk

Do not turn the method into ceremony. For a local, deterministic, low-risk change with an inexpensive representative fixture, use a lightweight path:

1. state a concise objective and the scope of the claim;
2. capture a repeatable baseline on the representative fixture;
3. identify the mechanism and make one focused, reversible change;
4. run focused correctness tests and the same before/after measurement;
5. report a scope-limited verdict and any untested production assumptions.

Use the full workflow for production incidents, distributed or multi-component paths, regressions, tail or capacity questions, concurrency effects, noisy measurements, external dependencies, long-duration resource behavior, material tradeoffs, or moderate/high-risk changes. The lightweight path reduces ceremony, not the requirement for evidence.

## Build a performance model

Reason about related signals together rather than chasing one number:

- **Latency:** Measure a distribution appropriate to the experience, including relevant tail percentiles. Averages can hide pauses, skew, and queueing.
- **Throughput:** Count useful completed work per unit time. Record offered, admitted, successful, failed, timed-out, retried, and dropped work where relevant.
- **Concurrency:** Record in-flight work and the load model. A closed loop that waits for responses and an open arrival process can produce different conclusions.
- **Capacity:** Find the highest sustainable workload that still meets latency, error, and resource constraints. Throughput bought with an ever-growing queue is not sustainable capacity.
- **Saturation:** Look for queue growth, contention, throttling, exhausted pools, pauses, swapping, or a resource near its effective limit. Utilization by itself is not a diagnosis.
- **Resources and cost:** Track CPU, memory, allocations, storage I/O, network I/O, connections, accelerators, external calls, and monetary cost as relevant.
- **Critical path:** Separate work that determines completion latency from parallel or background work. Separate queue time from service time when possible.

Use simple models to challenge assumptions:

- Under stable conditions, Little’s Law (`in-flight work ≈ throughput × response time`) can expose inconsistent measurements and growing queues.
- Bound expected impact with the fraction of total time or demand the candidate can affect. A large speedup to a tiny fraction of the critical path cannot produce a large end-to-end gain.
- Treat these models as approximations whose assumptions must match the workload.

## Workflow

### 1. Frame the performance contract

Turn the complaint into an operational statement. Capture what is known and label what is missing:

- affected user, operation, endpoint, job, query, render, or service boundary;
- symptom and when it began;
- metric and unit;
- current behavior, desired behavior, and time horizon;
- workload: volume, request or operation mix, input sizes, data distribution, concurrency, burstiness, and growth;
- environment: build mode, hardware, topology, configuration, dependency versions, limits, and cache state;
- guardrails: correctness, tail latency, throughput, errors, memory, cost, consistency, reliability, and deadline;
- recent code, dependency, configuration, infrastructure, data, or traffic changes.

A useful contract is concise:

```text
Objective: <metric and target> for <operation>
Workload: <representative data/request mix, arrival pattern, and concurrency>
Environment: <where and how it is measured>
Guardrails: <dimensions that must not regress>
Comparison: <baseline or known-good reference>
```

Ask only for missing information that materially changes the investigation. If it cannot be obtained, continue with bounded analysis and reduce the strength of the conclusion.

### 2. Inspect before editing

Understand the repository and the system path that serves the objective:

- map the request or data path across components and dependencies;
- inspect relevant code, tests, architecture notes, production evidence, existing benchmarks, and prior performance work;
- identify build and run commands, production-equivalent settings, feature flags, resource limits, and generated behavior;
- locate existing instrumentation before adding more;
- check whether the reported path is actually hot or frequent enough to matter.

Do not begin with a broad rewrite or a search for “slow-looking” code. Start at the observed boundary and narrow inward.

### 3. Design the evidence

Choose the least expensive measurement that can discriminate between plausible causes. Prefer, when available:

1. representative production telemetry, traces, profiles, and incident evidence;
2. representative end-to-end replay, benchmark, or load test;
3. targeted component benchmark or profile;
4. focused microbenchmark;
5. complexity or resource-demand model;
6. informed intuition.

Lower levels can be useful when higher levels are unavailable, but they support narrower claims. Triangulate when one source is ambiguous or intrusive.

Write important hypotheses in falsifiable form:

```text
Hypothesis: <mechanism> causes <symptom> under <workload>.
Prediction: If true, <observable signal> will change or correlate in <specific way>.
Test: <measurement that distinguishes this from competing explanations>.
Decision: <what result supports, weakens, or rejects it>.
```

Rank hypotheses by consistency with evidence, possible impact, and cost or risk of testing—not by familiarity. Include external dependencies, contention, queueing, data skew, retries, and load-generator limits when they fit the symptom.

### 4. Establish a reproducible baseline

Record enough context to make the comparison meaningful:

- exact revision or artifact and production-equivalent build mode;
- environment, configuration, data shape and size, dependency behavior, and resource limits;
- workload mix, arrival model, concurrency, duration, and warm/cold state;
- benchmark or reproduction command and relevant raw output;
- latency distribution, completed throughput, errors/timeouts, resource use, and queue behavior;
- warm-up behavior, run-to-run variation, and known sources of noise.

Use multiple observations or sufficient duration to understand normal variation. Separate startup, warm-up, and steady state when each matters. Interleave candidate and baseline runs when environmental drift is likely. Confirm the load generator and measurement instrumentation are not the limiting factors.

A single stopwatch result is an observation, not a reliable baseline. Do not report more precision than the experiment supports or derive extreme percentiles from too few samples.

### 5. Localize the bottleneck

Work top-down from the affected boundary:

1. Attribute end-to-end time, demand, or blocked work across major components.
2. Identify where queues form, work serializes, or a resource reaches an effective limit as load increases.
3. Separate latency caused by waiting from latency caused by service.
4. Drill into the dominant path with the relevant specialist or profiler.
5. Test whether changing or removing the suspected work changes the measured symptom.

Correlate signals, but require a causal mechanism. High CPU, frequent allocation, a slow dependency call, a lock, or an expensive function may be a symptom of upstream pressure rather than the limiting cause.

For tail problems, inspect slow exemplars and stratify by input size, tenant, key, route, cache state, dependency, and time window. Rare pauses, hot keys, skew, head-of-line blocking, retries, and bursts disappear in aggregates.

### 6. Choose a proportionate intervention

For each credible candidate, compare:

- expected effect on the stated objective and its plausible upper bound;
- strength of evidence and likelihood of addressing the bottleneck;
- correctness and compatibility risk;
- effects on tails, throughput, memory, I/O, network, cost, consistency, and reliability;
- implementation and operational complexity;
- reversibility, rollout strategy, and maintenance burden;
- the next bottleneck the change may expose or overload.

Prefer a small, testable intervention over a permanent abstraction or architectural rewrite when both address the measured constraint. Do not add caching, batching, parallelism, compression, pooling, denormalization, precomputation, or hardware merely because it is a familiar performance technique; each has workload-dependent costs and failure modes.

When no candidate has enough evidence, instrument or benchmark first. When the system already meets its objective with adequate headroom, stop unless the user has another explicit goal such as cost reduction.

### 7. Implement safely

When authorized to change code:

- protect existing behavior with focused tests before relying on performance results;
- change one material variable at a time where attribution matters;
- keep resource use bounded and define overload or failure behavior;
- make moderate- or high-risk changes reversible, observable, and suitable for controlled rollout;
- preserve ordering, concurrency, cancellation, timeout, retry, and backpressure semantics;
- avoid benchmark-only shortcuts that production cannot use;
- document only non-obvious constraints and tradeoffs introduced by the optimization.

Do not weaken consistency, durability, security, precision, or quality silently. Present such options explicitly and require acceptance of the tradeoff.

### 8. Verify the result

When proving a prior optimization, reconstruct its original claim and system boundary, account for all differences between baseline and candidate, and actively seek disconfirming evidence.

Repeat the baseline protocol under comparable conditions. Evaluate the full objective, not just the metric most likely to improve:

- compare distributions or repeated observations, not only a single run;
- judge practical significance relative to baseline variability and the target;
- include completed throughput, failures, timeouts, retries, dropped work, and queue growth;
- check tail behavior and realistic variants such as peaks, large inputs, skew, cold state, and long duration where relevant;
- compare CPU, memory, allocation, I/O, network, external demand, and cost;
- run focused correctness tests, then broader relevant tests and builds;
- validate a component-level win at the affected end-to-end boundary;
- re-profile or remeasure to determine whether the bottleneck disappeared, remained, or moved.

Classify the result precisely:

- **Verified improvement:** the intended metric improved beyond expected noise, guardrails held, and the target boundary benefited.
- **No meaningful change:** the result is within normal variation or too small to matter.
- **Regression:** the objective or a guardrail worsened materially.
- **Inconclusive:** the experiment cannot distinguish the outcomes or was not representative.

Revert, revise, or retain a change based on this result—not on effort already spent.

### 9. Preserve useful evidence

Keep a stable benchmark, workload fixture, performance budget, or production signal when it can detect recurrence without creating flaky maintenance. Store the reproduction method and important environment assumptions with it.

Do not place noisy microbenchmarks into blocking CI merely to claim regression coverage. Choose thresholds with enough margin for expected variance and use trend monitoring when hard pass/fail limits are inappropriate.

## Mode-specific guidance

### Regression investigation

- Reproduce known-good and degraded versions with the same workload and environment.
- Compare code, dependencies, configuration, infrastructure, data shape, traffic mix, and external behavior; a code diff is not the only possible change.
- Establish whether the delta exceeds ordinary variance before bisecting.
- Vary one dimension at a time and narrow to the smallest causal boundary.
- After identifying a suspect, explain the mechanism and verify that reverting or correcting it removes the regression.

### Scale design

- Convert expected average, peak, burst, tenant skew, data size, and growth into explicit demand and latency budgets.
- Model the critical path, serial sections, per-operation resource demand, queueing, and failure behavior under overload.
- Define sustainable capacity as meeting latency and error objectives with safe headroom, not merely surviving a peak test once.
- Validate important assumptions with representative tests or existing production evidence before committing to irreversible architecture.
- Label projections and safety factors as assumptions; do not present modeled capacity as measured capacity.


## When measurement is difficult

Do not manufacture certainty to keep the task moving.

- **Execution is unavailable:** Perform code and architecture analysis, identify suspected mechanisms, and provide a reproducible measurement plan. If the user explicitly requested implementation and static evidence establishes a specific mechanism, a low-risk reversible change may proceed with correctness tests, an **unverified performance** label, and concrete validation steps. If the code merely looks slow or the change is riskier, do not edit for performance. Do not claim measured improvement.
- **The baseline is noisy:** Stabilize the environment, lengthen or repeat runs, stratify workloads, or interleave comparisons. If the expected effect remains below noise, report it as inconclusive.
- **Production is the only representative environment:** Prefer existing telemetry, safe sampling, replay, shadowing, canaries, or controlled experiments. Never run disruptive load against production without explicit authorization and safeguards.
- **The issue is rare or tail-only:** Use longer windows, event-triggered captures, traces of slow exemplars, and workload segmentation instead of averaging it away.
- **A dependency is outside your control:** Demonstrate its contribution and conditions with traces, timing, errors, or controlled substitution. Hand off a minimal reproduction and evidence to the owner. Stop unrelated local tuning unless a specific mitigation objective is established; evaluate that mitigation as a separate hypothesis and verify it end to end.
- **Instrumentation changes behavior:** Quantify or bound the overhead, reduce sampling, and triangulate with a less intrusive signal.

A measurement-enabling change is often more valuable than a speculative optimization. An obviously safe cleanup may still be worthwhile for maintainability, but do not sell it as a performance improvement without evidence.

## Avoid cargo-cult optimization

Do not:

- optimize code merely because it contains loops, allocations, abstraction, serialization, or remote calls;
- treat asymptotic complexity or profiler percentages as proof of production impact;
- use a debug build, tiny fixture, idealized cache state, or laptop microbenchmark to make an unqualified production claim;
- improve a headline metric by hiding errors, timeouts, retries, dropped work, or queue growth;
- add concurrency or caching without accounting for bounds, contention, downstream capacity, lifecycle, correctness, and failure behavior;
- retain fragile complexity for a gain smaller than experiment noise or irrelevant to the target.

## Report the conclusion

Make the final report concise and auditable. Include sections that apply:

1. **Objective and scope** — affected boundary, metric, target, workload, and guardrails.
2. **Baseline** — environment, method, results, and variability.
3. **Evidence and diagnosis** — observations, bottleneck or hypotheses, and causal reasoning.
4. **Change or recommendation** — files or components affected, mechanism, risk, and tradeoffs.
5. **Validation** — before/after results, correctness checks, guardrails, and bottleneck movement.
6. **Verdict** — verified improvement, no meaningful change, regression, or inconclusive.
7. **Limitations and next step** — assumptions, missing evidence, confidence, rollback, or handoff.

Include exact commands or artifacts needed to reproduce measurements when available. If no files were changed, say so. If validation could not run, state why and limit the claim accordingly.

Match language to evidence: “suggests” for a lead, “supports” for a diagnosis, “is expected to” for a prediction, and “the comparable measurement showed” for a verified result. Never use “faster,” “fixed,” “scales to,” or a numeric improvement as fact unless the stated workload and evidence support it.
