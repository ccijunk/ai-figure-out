# 07 — Spec (final)

## Goal

Define the 7-role set for the `spec-to-code` workflow: final role list with justifications, per-role artifact contracts, and fillable artifact templates with named sections.

## Non-goals

- Writing `spec-to-code.yaml` (follow-up run).
- Defining executor prompts (prompts use these templates; templates come first).
- Role memory persistence schema (deferred; see Open questions).
- Executor assignment per role.

## Approach

Pipeline: `ba → architect → test-arch → developer → reviewer → tester → meta`

The fan-out between test-arch and developer is **sequential, not parallel.** test-arch gates architect before developer starts. This resolves the timing contradiction in the draft.

- **ba** — models the business domain across its full lifecycle (creation, operation, edge cases, termination). Evaluated by the architect on non-contradiction and completeness. Does NOT make technology choices.
- **architect** — (1) challenges `clarify.md`; writes `APPROVED` or `BLOCKED`; (2) if approved, transforms domain model into DDD code structure. Does NOT write code. Does NOT design tests.
- **test-arch** — (1) challenges `design.md` on testability using explicit criteria (see template); (2) writes `APPROVED` or `BLOCKED`; (3) if approved, designs lifecycle safeguards: test cases + regression stubs. Does NOT execute tests.
- **developer** — starts only after test-arch writes `APPROVED`. Inputs: `clarify.md`, `design.md`, `test-design.md`. Implements code aligned to test IDs. Does NOT design tests. Does NOT review.
- **reviewer** — static critique: correctness, maintainability, adherence to `design.md`. Does NOT run tests.
- **tester** — dynamic behavior: executes test strategy using runner commands from `test-design.md`. Severity-tags findings. Does NOT fix code.
- **meta** — retrospective. Non-blocking. Does NOT re-open decided items.

Two adversarial gates, both using the `APPROVED` / `BLOCKED` sentinel:
- **Architect gates BA.** `design.md ## Domain model review` verdict. If `BLOCKED`, BA re-runs.
- **Test-arch gates Architect.** `test-design.md ## Testability review` verdict. If `BLOCKED`, architect re-runs and revises `design.md`, then test-arch re-evaluates.

## Key decisions

- **Pipeline is ba → architect → test-arch → developer (sequential).** The draft's parallel fan-out broke because developer was blocked on test-arch anyway. Sequential is honest; the state machine declares `test-arch → developer` as a dependency edge.
- **Both gates use the same parseable sentinel.** First line of `## Domain model review` and `## Testability review` must be exactly `APPROVED` or `BLOCKED`. Orchestrator pattern-matches this line; validator rejects the artifact if neither is present.
- **Testability gate has explicit criteria, not examples.** Three checkable criteria: (1) every aggregate has at least one observable output; (2) no direct cross-context dependency without an interface; (3) every external side-effect is injectable or replaceable. Test-arch applies these criteria; `BLOCKED` lists which criteria fail.
- **`clarify.md` has an explicit lifecycle section.** `## Lifecycle coverage` with four required rows: Create / Operate / Edge case / Terminate. Architect uses this to check completeness.
- **`reflect.md ## Role memory updates` removed.** Placeholder with instructions to leave blank is not a template section. Removed; added to Open questions.

## Milestones

1. Role contracts + template sections defined. (This spec.)
2. Templates validated against one representative story (validation run).
3. `spec-to-code.yaml` node `outputs` declarations + transition edges written from these templates.
4. Artifact validator updated with sentinel checks for both gates.

## Open questions

1. **BA/architect domain model boundary.** Does `clarify.md` include DDD constructs (aggregates, bounded contexts) or only business rules and entities in plain language? Defer to validation run.
2. **Role memory schema.** `memory/<role>.md` format is undefined. Until defined, meta has no structured output for the learning loop. Follow-up run required.
3. **review / test-report cross-reference.** Excluded from v1. Revisit after validation run.

---

## Appendix — Artifact templates

---

### `clarify.md` (ba)

```
## Problem statement
[What the software must do in business terms. One paragraph.]

## Domain entities
[Table or list: entity name | brief definition | key attributes]

## Business rules
[Numbered list. Each rule must be independently statable. Do not write rules that contradict each other — architect will flag these as BLOCKED.]

## Lifecycle coverage
[Table with four required rows:
 Phase | Key events | Rules that apply | Termination condition
 Create | ... | ... | ...
 Operate | ... | ... | ...
 Edge case | ... | ... | ...
 Terminate | ... | ... | ...]

## Bounded contexts
[Domain partitions and their interfaces. "N/A — single context" is a valid entry.]

## Acceptance criteria
[Format: Given / When / Then. One criterion per lifecycle phase in Lifecycle coverage.]

## Open questions
[Unresolved business questions. Architect will evaluate these during domain model review.]
```

---

### `design.md` (architect)

```
## Domain model review
APPROVED
[or]
BLOCKED
[If BLOCKED, numbered list of conflicts or lifecycle gaps in clarify.md.
 BA must resolve all items before pipeline continues.]

## DDD structure
[Bounded contexts → aggregates → entities / value objects / domain services.
 List form: name | type | responsibility]

## System architecture
[Layer structure and major components. ASCII diagram acceptable.]

## Technology choices
[Library / framework / data store. One-line justification each.]

## Integration points
[External systems, APIs, event buses. "None" is a valid entry.]

## Open questions
[Unknowns for test-arch and developer to resolve.]
```

---

### `test-design.md` (test-arch)

Inputs: `clarify.md`, `design.md`.

```
## Testability review
APPROVED
[or]
BLOCKED
[If BLOCKED, numbered list of failed criteria from design.md:
 (1) aggregates without observable outputs
 (2) direct cross-context dependencies without an interface
 (3) external side-effects that are not injectable or replaceable
 Architect must revise design.md and test-arch re-evaluates before developer starts.]

## Test strategy
[Unit / integration / e2e scope and rationale. Philosophy (behavior-driven, property-based, etc.).]

## Feature test cases
[Table: ID | scenario | precondition | action | expected outcome]

## Regression suite
[Stubs that must pass on every feature addition or bug fix.
 Table: ID | repo-relative file path | runner command | trigger (every-commit / every-release)]

## Edge cases
[Boundary conditions and failure scenarios not in feature test cases.]

## Test tooling
[Frameworks and runners. Version-pinned.]
```

---

### `implementation.md` (developer)

Inputs: `clarify.md`, `design.md`, `test-design.md`.
Prerequisite: `test-design.md ## Testability review` must be `APPROVED`.

```
## Summary
[What was built. One paragraph.]

## Code structure
[File / module layout. Tree form acceptable.]

## Key implementation decisions
[Choices that deviate from or extend design.md.]

## Test ID alignment
[For each test ID in test-design.md that influenced an implementation choice: test ID | decision made]

## Known limitations
[Deferred items, workarounds, intentional tech debt.]
```

---

### `review.md` (reviewer)

```
## Summary
[Overall code health. One paragraph.]

## Findings
[Numbered list: location (file:line) | issue description | severity (blocker / major / minor)]

## Verdict
[One of: "Approved" / "Approved with minor issues" / "Changes required".]
```

---

### `test-report.md` (tester)

```
## Test run summary
[Pass / fail / skipped counts. Coverage percentage if available.]

## Failures
[Numbered list: test ID | description | severity (blocker / major / minor) | reproduction steps]

## Regression status
[For each entry in test-design.md ## Regression suite: ID | result (passed / failed / not-run) | failure detail if applicable]

## Verdict
[One of: "Pass" / "Fail — blockers present" / "Fail — majors only".]
```

---

### `reflect.md` (meta)

```
## What worked
[By role: practices or artifact sections that produced useful output.]

## What degraded
[By role: sections that were thin, skipped, or produced noise.]

## Carry-forward
[Table: role | change to prompt or template | reason]
```

---

## Resolutions

**#1 — Parallel fan-out breaks developer start timing (blocker)**
`fixed` — Pipeline revised to `ba → architect → test-arch → developer` (sequential). The parallel claim was dishonest; developer was already blocked on test-arch. Sequential is correct; the state machine declares the dependency edge explicitly.

**#2 — BLOCKED feedback destination contradicts itself (blocker)**
`fixed` — Conflict lists stay in-artifact (in `## Domain model review` and `## Testability review`). Open question #2 in the draft is removed. The templates are the answer.

**#3 — `clarify.md` has no lifecycle coverage section (major)**
`fixed` — Added `## Lifecycle coverage` table with four required rows: Create / Operate / Edge case / Terminate. Architect checks this table for completeness; incomplete rows are a `BLOCKED` criterion.

**#4 — Testability review criteria are examples, not a checklist (major)**
`fixed` — Three explicit criteria defined: (1) every aggregate has at least one observable output; (2) no direct cross-context dependency without an interface; (3) every external side-effect is injectable or replaceable. `BLOCKED` entries reference which criteria failed.

**#5 — Architect re-run path unspecified for test-arch gate (major)**
`fixed` — Template now states "Architect must revise `design.md` and test-arch re-evaluates before developer starts." This is the declared loop; the state machine in `spec-to-code.yaml` will encode the `test-arch BLOCKED → architect` back-edge as a transition. The spec names the obligation; the YAML spec run implements it.

**#6 — `implementation.md` mixes instructions with section descriptions (minor)**
`fixed` — "Reference test IDs" editorial note removed from `## Key implementation decisions`. Replaced with a separate `## Test ID alignment` section with a clear format: `test ID | decision made`.

**#7 — `reflect.md ## Role memory updates` is a blank stub (minor)**
`fixed` — Section removed from template entirely. Added to Open questions (#2) as a follow-up run item.
