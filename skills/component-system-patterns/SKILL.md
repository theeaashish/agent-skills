---
name: component-system-patterns
description: Design and evolve production UI component architecture through evidence-based decisions about component boundaries, responsibilities, cohesion, composition, APIs, state and behavior ownership, reuse, abstraction, and compatibility. Use when a UI task requires deciding how components should be divided, coordinated, composed, shared, controlled, or safely migrated—for example decomposing a god component, consolidating needless wrappers, choosing local versus extracted versus shared code, reducing prop or configuration sprawl, relocating state, designing reusable component APIs, or reviewing component structure. Apply even when the request is framed as cleanup, splitting, consolidation, reuse, or API simplification. Do not use for styling-only changes, visual design-system documentation, framework syntax tutorials, isolated accessibility fixes, performance tuning, or test-tooling work unless the task also involves component architecture. Framework-agnostic; pair it with relevant specialist skills.
---

# Component System Patterns

## Purpose

Design component systems that are easy to understand, change, compose, and evolve without forcing every component toward either extreme:

- Do not let one component accumulate unrelated rendering, orchestration, domain rules, state, effects, and data access until every change risks the whole feature.
- Do not split cohesive UI into tiny wrappers and fragments that add names, files, props, and navigation without creating useful ownership or abstraction.

Optimize for **clear responsibility, high cohesion, intentional coupling, explicit ownership, stable contracts, and safe change**. Do not optimize for the smallest files, the fewest lines, the most components, or the most reuse.

This is an engineering decision skill, not a component syntax tutorial or visual style guide.

## Core judgment

Treat component design as a boundary problem:

> A useful component boundary encloses a coherent responsibility, gives that responsibility an appropriate owner, and exposes a contract that lets consumers use it without knowing unnecessary implementation details.

Apply these principles:

1. **Use evidence, not thresholds.** Line count, prop count, branch count, nesting depth, and number of call sites may prompt inspection, but none proves that a boundary is good or bad.
2. **Make every extraction earn its cost.** Extraction should improve ownership, cohesion, independent variation, reuse, testability of meaningful behavior, or consumer clarity. Moving markup to another file is not inherently an improvement.
3. **Prefer the narrowest appropriate scope.** Move from local code to a private subcomponent, then to a feature-level component, and only then to a shared or public component as evidence justifies a broader contract.
4. **Reuse concepts, not appearances.** Similar markup is not enough. Share components when consumers need the same semantic capability, behavior, invariants, and likely evolution.
5. **Allow deliberate duplication.** Duplication is often cheaper than a premature abstraction that couples unrelated consumers or accumulates exceptions.
6. **Give state one authority.** Keep authoritative state and the rules that mutate it together. Derive rather than mirror whenever possible.
7. **Use composition and configuration intentionally.** Composition is suited to open-ended structural variation; configuration is suited to a closed set of stable semantic modes.
8. **Preserve contracts during refactoring.** Unless behavior change is explicit, protect consumer-visible defaults, events, state lifetime, reset behavior, error behavior, and extension points.
9. **Fit the codebase before improving it.** Inspect existing conventions and abstractions before introducing a pattern familiar from another project or framework.

## Start with repository evidence

Before proposing or changing boundaries, inspect the smallest relevant component graph rather than one file in isolation.

Inspect, as available:

- the component and its direct collaborators;
- all call sites, imports, exports, registries, and package entry points;
- sibling components and feature-local conventions;
- stories, examples, tests, and documentation that reveal intended use;
- state sources, derived values, transitions, effects, commands, and emitted events;
- data-loading and domain-service boundaries that the component consumes;
- variants, conditional branches, extension points, and consumer-specific exceptions;
- recent change history when it can reveal which concerns evolve together.

Search before creating. Prefer an established local pattern when it satisfies the need. Do not infer a shared API from one implementation or one screenshot.

If consumers or compatibility requirements cannot be determined, state the uncertainty and choose the most reversible, local option.

## Model the current responsibility

Before editing, write a brief internal model:

- **Job:** What coherent outcome does this component own?
- **Consumers:** Who uses it, and what capabilities do they rely on?
- **Responsibilities:** Which rendering, behavior, coordination, state, and data concerns are present?
- **Ownership:** Which boundary is authoritative for each state value, invariant, and side effect?
- **Variation:** What changes between uses: content, structure, behavior, state policy, or only appearance?
- **Change pattern:** Which concerns usually change together, and which change independently?
- **Contract:** Which behaviors are externally observable, even if they are not formally documented?

If the job cannot be stated coherently, investigate why. The answer may be a split, but it may also be unclear naming, mixed domain logic, or scattered state rather than a component-count problem.

## Choose boundaries using multiple dimensions

Evaluate these dimensions together; do not turn them into a mechanical score.

| Dimension | Ask |
|---|---|
| Responsibility | Does the candidate boundary own one coherent capability or outcome? |
| Cohesion | Do its data, behavior, state, and rendering naturally belong together and change together? |
| Coupling | What knowledge and coordination must cross the boundary? Does extraction reduce coupling or merely relocate it? |
| Ownership | Can one boundary clearly own the relevant state, invariants, effects, and lifecycle? |
| Reuse | Is there proven or near-term reuse of the same semantic concept, not just similar markup? |
| Change pattern | Will the candidate evolve independently, or will every change require editing parent and child together? |
| Consumer clarity | Does the proposed contract make common use obvious without exposing internals? |
| Compatibility | Can the boundary evolve without surprising existing consumers? |

A good split usually reduces the knowledge needed to understand or change one responsibility. A poor split often replaces local complexity with forwarding props, wrapper layers, synchronized state, or cross-file navigation.

## Decide whether code stays local, is extracted, or becomes shared

### Keep code local when

- it is meaningful only as part of the parent's coherent responsibility;
- its behavior, data, and markup change with the parent;
- it has no useful independent contract or ownership;
- extraction would mostly forward values and callbacks;
- the abstraction name would only restate the markup;
- the concept is still evolving and a wider API would be speculative.

Local code is not a failure of reuse. Locality protects context and keeps unstable decisions inexpensive to change.

### Extract a private or feature-local component when

- it owns a recognizable sub-responsibility with a clear contract;
- it has state or lifecycle that is naturally independent;
- it represents a distinct workflow or region that can vary independently;
- isolation removes unrelated knowledge from the parent;
- it can hide implementation details instead of requiring the parent to micromanage them;
- multiple uses within the feature share the same semantics.

Keep the extraction private unless external consumers need it. A new file does not need to become a new public API.

If the extracted unit has no independent UI identity, state, behavior, or composition role, consider whether a local helper or domain function is a better boundary than another component.

### Create or promote a shared component when

- multiple consumers need the same semantic capability and behavioral contract;
- their required variants are compatible rather than a collection of exceptions;
- the abstraction has a clear owner and a plausible evolution path;
- sharing reduces duplicated decisions, not merely duplicated tags;
- the public API can remain smaller and clearer than the implementations it replaces;
- representative consumers can be migrated and validated.

Do not use "used twice" as a promotion rule. Two uses can reveal a concept, but they can also be coincidental. Conversely, one strategically important capability may justify a shared boundary when its contract is already stable and intentionally public.

### Keep implementations separate when

- similarity is primarily visual;
- the concepts have different meanings, invariants, permissions, workflows, or owners;
- they are likely to evolve in different directions;
- a shared API needs consumer flags, special cases, or escape hatches from the start;
- the cost of coupling exceeds the cost of explicit duplication.

Revisit the decision when actual change patterns become clearer.

## Prevent god components without cosmetic decomposition

Investigate a component when it shows several of these pressures:

- unrelated reasons to change are concentrated in one place;
- feature orchestration, domain policy, remote-data mechanics, state transitions, and presentation are interleaved;
- distinct workflows are encoded as large conditional branches;
- changes to one region require understanding unrelated regions;
- many state values form separate clusters with different lifetimes or owners;
- consumer-specific behavior keeps accumulating in a nominally general component;
- the component's contract cannot be explained without describing its internal tree.

Respond by separating responsibilities, not by slicing at visual or file-size boundaries:

1. Map responsibilities and externally observable behavior.
2. Identify the authoritative state and invariants for each workflow.
3. Separate domain calculations or transport mechanics when they do not belong to the UI boundary.
4. Extract cohesive regions or behaviors one at a time.
5. Keep a coordinator component when orchestration itself is a coherent responsibility.
6. Preserve the existing external contract until a deliberate migration is planned.
7. Reassess whether the result actually reduces knowledge and coupling.

A coordinator is not automatically a god component. It may legitimately connect data, state, and children if it owns one feature-level workflow and delegates details through clear contracts.

## Prevent excessive fragmentation

Merge or inline boundaries that:

- only wrap trivial markup without semantics, behavior, reuse, or independent variation;
- forward nearly every input and event unchanged;
- force readers through several files to understand one small operation;
- split one state transition or invariant across parent and child;
- exist only to satisfy a line-count preference;
- create a public abstraction before a stable concept exists;
- add generic names such as `Container`, `Section`, or `Wrapper` without clarifying responsibility.

Do not flatten boundaries that genuinely own state, behavior, lifecycle, invariants, independent variation, or a shared semantic concept. The goal is meaningful structure, not the fewest components.

## Place state and behavior deliberately

Place authoritative state at the lowest boundary that can enforce its invariants and coordinate every consumer that must agree.

Prefer parent or coordinator ownership when:

- multiple children must stay consistent;
- state participates in feature-level submit, reset, persistence, navigation, undo, or external control;
- the owner enforces an invariant across child boundaries;
- another consumer must observe or command the same state.

Prefer child ownership when:

- state is transient and affects only that child;
- its lifetime naturally matches the child;
- no external consumer needs authoritative access;
- lifting it would make the parent manage implementation details.

Keep these safeguards:

- Do not store the same authoritative value in parent and child and synchronize the copies.
- Derive values that can be computed from authoritative inputs unless there is a justified snapshot or caching contract.
- Keep transition rules near the state and invariants they protect.
- Emit semantic events or user intent rather than leaking raw internal setters.
- Expose commands only when external orchestration genuinely requires them.
- If many values and callbacks pass through several levels, reconsider ownership or composition before adding more forwarding.
- If both caller-controlled and internally owned modes are supported, define their source-of-truth, transition, reset, and switching semantics explicitly.

Data fetching does not automatically belong inside or outside a component. Place request ownership where loading, cancellation, errors, freshness, and feature lifetime can be managed coherently. Keep reusable presentation components independent of feature-specific transport and domain policy unless those concerns are part of the component's declared semantic contract.

## Divide parent and child responsibilities

Use the parent to:

- coordinate siblings and feature-level workflows;
- own shared state and cross-child invariants;
- choose high-level structure and provide domain-relevant inputs;
- translate external application state into child capabilities.

Use the child to:

- own implementation details within its contract;
- manage self-contained interaction state and lifecycle;
- enforce local invariants;
- emit meaningful outcomes without requiring the parent to know its internal steps.

Avoid "puppet" children whose parents control every internal detail, and avoid opaque children that hide state the feature must coordinate. Move the boundary until knowledge and authority align.

## Choose composition or configuration

### Prefer composition when

- callers own content, ordering, or structural arrangement;
- regions need independent substitution or omission;
- future variation is open-ended;
- caller-provided behavior has its own state or lifecycle;
- a fixed option list would grow with each consumer.

Expose a small number of deliberate composition points. Do not expose every internal node, or consumers will depend on the implementation tree.

### Prefer configuration when

- variants form a finite, stable set of semantic modes;
- the component must enforce consistent structure or behavior;
- options express domain capabilities rather than low-level rendering toggles;
- the component can keep combinations valid and understandable.

Use semantic variants instead of collections of interacting booleans. Configuration should describe intent, not recreate a UI framework in data.

### Use a hybrid when

A component has a stable semantic shell, a few closed behavioral modes, and selected regions whose content or structure belongs to the caller. Keep each mechanism responsible for the kind of variation it handles best.

Reconsider the API when it develops:

- booleans with dependent or contradictory combinations;
- props used only by one special-case consumer;
- configuration objects that encode arbitrary trees or event programs;
- one slot, callback, or override for every internal element;
- long chains of transparent prop forwarding;
- escape hatches that are becoming the normal usage path.

## Design stable, composable component APIs

Treat an API as a capability contract, not a mirror of the implementation.

Design APIs that:

- name domain or interaction intent rather than internal elements;
- make common use concise and clear;
- make valid combinations natural and invalid combinations difficult;
- provide intentional defaults;
- distinguish data, commands, events, content, and structural extension points;
- expose the minimum surface needed for known consumers without blocking required composition;
- hide internal state representation, child structure, and transport details;
- allow internal refactoring without consumer changes.

An API surface is excessive when consumers must understand implementation details, coordinate values that should be one concept, pass mutually dependent options, or configure unrelated capabilities through one component. Do not diagnose excess from a prop count alone.

When one shared component serves unrelated semantic roles, prefer separate components or a smaller shared primitive over a polymorphic mega-component. When roles share a stable shell and invariants, composition may preserve the common core without merging every behavior.

Use escape hatches sparingly. Add one only for a real constraint that cannot be represented safely by the main API, and keep it from undermining invariants or becoming an undocumented second API.

## Evolve shared components safely

Assume a shared component has more contract surface than its type signature shows. Consumers may rely on:

- defaults and fallback behavior;
- event meaning, order, timing, and cancellation;
- state initialization, preservation, and reset behavior;
- loading, empty, disabled, and error states;
- rendered semantics or structure used by integrations;
- extension points and pass-through behavior;
- controlled versus internally owned state policy.

Before changing a shared component:

1. Classify it as private, feature-internal, application-shared, or externally published.
2. Inventory consumers, exports, tests, examples, wrappers, and indirect usages.
3. State the old and proposed contracts, including defaults and lifecycle behavior.
4. Determine whether the change is additive, compatible with an adapter, deprecatable, a coordinated migration, or intentionally breaking.
5. Prefer the least surprising path that does not leave permanent ambiguity in the API.
6. Migrate representative consumers and validate them from the consumer perspective.
7. Remove compatibility code only after remaining usage is verified and the project's release policy permits it.

When external consumers cannot be inventoried, conservatively treat public types, documentation, examples, release policy, and established behavior as the observable contract.

Do not export an extracted internal component merely because it now exists. Do not silently repurpose an existing option with a different meaning. If a breaking change is the cleanest choice, make it explicit and provide a bounded migration plan rather than hiding it behind complex compatibility logic.

## Task workflows

### When creating components

1. Inspect nearby patterns and likely consumers.
2. Define the component's job, ownership, and required variation.
3. Start at feature-local scope unless sharing is already an explicit, stable requirement.
4. Design the smallest complete semantic API for known use cases.
5. Keep uncertain variation local until evidence supports an extension point.
6. Validate the API through realistic consumer code, not only the component implementation.

### When modifying components

1. Determine whether the requested behavior belongs to the existing responsibility.
2. Inspect every affected consumer before widening a shared API.
3. Add a semantic capability rather than a consumer-specific switch when possible.
4. If the request reveals a separate responsibility, extract at the narrowest useful scope.
5. Avoid unrelated restructuring unless it is necessary to make the change safe.

### When refactoring components

1. Define the observable behavior that must remain unchanged.
2. Map responsibilities, ownership, and change pressures.
3. Choose one structural improvement with a clear benefit.
4. Refactor in reversible steps while keeping the public contract stable.
5. Update call sites, tests, examples, and exports that are part of the contract.
6. Verify that the new structure reduces knowledge or coupling rather than moving it.

For a god component, separate coherent responsibilities while retaining a legitimate coordinator. For an over-fragmented tree, inline forwarding wrappers while preserving boundaries that own real behavior, state, or reuse. For prop sprawl, identify whether the root problem is mixed responsibilities, the wrong state owner, or structural variation forced into configuration.

### When reviewing components

Prioritize findings by consequence:

1. incorrect or ambiguous state authority, invariants, and behavior;
2. consumer-breaking API or lifecycle risk;
3. mixed responsibilities and harmful coupling;
4. premature sharing or needless fragmentation;
5. optional opportunities whose value depends on future change.

For each material finding, provide:

- concrete evidence from the code and consumers;
- the change or maintenance risk;
- the smallest justified recommendation;
- why the recommendation is better than both leaving a god component and over-splitting it;
- compatibility and validation implications.

Do not recommend a rewrite merely because another structure is aesthetically preferable. If the current boundary is cohesive and fit for its consumers, say that no extraction is justified.

## Common AI-agent failure modes

Avoid these patterns:

- **Cosmetic decomposition:** moving sections into files while the original component still owns every decision.
- **Component ping-pong:** making readers traverse many trivial wrappers to follow one interaction.
- **Speculative abstraction:** designing variants, slots, adapters, or generic primitives for hypothetical consumers.
- **False DRY:** merging visually similar components with different semantics or futures.
- **God configuration:** replacing a god component with one giant options object or schema.
- **Boolean state-space explosion:** combining independent flags into invalid or unclear modes.
- **State over-lifting:** moving all state upward even when no coordination requires it.
- **Split-brain state:** keeping synchronized authoritative copies at multiple levels.
- **Setter leakage:** exposing implementation mutation instead of semantic events or commands.
- **Accidental public expansion:** exporting newly extracted internals without consumer need.
- **Lifecycle drift:** changing when state resets or persists as an accidental side effect of moving boundaries.
- **Contract loss:** dropping defaults, events, errors, or edge cases through a wrapper or migration.
- **Internal-tree APIs:** exposing every internal region as configurable structure.
- **Structure-locking tests:** asserting the internal component tree instead of consumer-visible behavior and contracts.
- **Refactor scope creep:** bundling boundary changes with unrelated cleanup, renaming, or formatting churn.
- **Framework transplantation:** imposing patterns from another ecosystem without repository evidence.
- **Threshold dogma:** treating any component as wrong solely because of lines, props, branches, or children.

## Work with companion skills

Use this skill to decide **where responsibilities and contracts belong**. Use relevant companion skills for their specialized mechanics:

- framework skills for idiomatic syntax, lifecycle primitives, and rendering conventions;
- accessibility skills for semantic interaction, focus, keyboard, and assistive-technology requirements;
- styling or frontend-design skills for visual systems, layout, tokens, and polish;
- frontend-performance skills for measurement and rendering, loading, and bundle optimizations;
- testing skills for framework-specific test strategy and tooling.

Let specialized constraints inform component boundaries, but do not duplicate those disciplines here.

Do not turn this work into a React tutorial, CSS guide, design-system catalog, accessibility audit, performance checklist, test-framework tutorial, generic naming convention, or file-layout style guide. Do not prescribe arbitrary maximum lines, props, branches, nesting levels, or components per file.

## Completion check

Before finishing, confirm that:

- repository patterns and affected consumers were inspected;
- each new boundary has a concrete architectural benefit;
- cohesive code was not split merely for size or visual symmetry;
- shared abstractions represent shared semantics rather than resemblance;
- authoritative state and behavior have clear owners;
- parent and child contracts avoid micromanagement and hidden coordination;
- composition and configuration match the type of variation;
- public API and observable behavior changes are intentional;
- migration and validation cover affected consumers;
- the result is simpler to explain and safer to change than the starting point.
