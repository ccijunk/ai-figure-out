# 05 — Spec Draft

## Goal

Define the 7-role set for the `spec-to-code` workflow: final role list with justifications, per-role artifact contracts, and fillable artifact templates with named sections.

## Non-goals

- Writing `spec-to-code.yaml` (follow-up run).
- Defining executor prompts (prompts use these templates; templates come first).
- Role memory persistence schema (deferred; `reflect.md ## Role memory updates` is a stub until schema is defined).
- Executor assignment per role.

## Approach

Pipeline: `ba → architect → (test-arch ∥ developer) → reviewer → tester → meta`

- **ba** — models the business domain. Evaluated by the architect on non-contradiction and completeness. Does NOT make technology choices.
- **architect** — (1) challenges `clarify.md`; writes a parseable verdict; (2) if approved, transforms domain model into DDD code structure. Does NOT write code. Does NOT design tests.
- **test-arch** — inputs: `clarify.md` + `design.md`. (1) challenges `design.md` on testability — flags untestable coupling, missing isolation boundaries, unobservable outputs; (2) designs lifecycle safeguards: test cases + regression stubs with runner command. Does NOT execute tests. Does NOT review code.
- **developer** — inputs: `clarify.md` + `design.md` + `test-design.md` (read-only). Implements code. Does NOT design tests. Does NOT review.
- **reviewer** — static critique: correctness, maintainability, adherence to `design.md`. Does NOT run tests.
- **tester** — dynamic behavior: executes test strategy using runner commands from `test-design.md`. Severity-tags findings. Does NOT fix code.
- **meta** — retrospective. Non-blocking. Does NOT re-open decided items.

Two adversarial gates in the pipeline:
- **Architect gates BA.** Fan-out fires only after `design.md ## Domain model review` is `APPROVED`. If `BLOCKED`, BA re-runs.
- **Test-arch gates Architect.** `test-design.md ## Testability review` must be `APPROVED` before developer starts implementation. If `BLOCKED`, architect revises `design.md`.

## Key decisions

- **BA output is a domain model, not a BRD.** Non-contradiction and completeness are the architect's gate criteria, not self-assessed by BA.
- **Architect gate uses a parseable sentinel.** First line of `## Domain model review` must be exactly `APPROVED` or `BLOCKED`. Orchestrator pattern-matches this line.
- **Test-arch gate uses the same sentinel pattern.** First line of `## Testability review` must be `APPROVED` or `BLOCKED`. Symmetric to the architect gate. If `BLOCKED`, architect revises `design.md` before developer starts.
- **Regression stubs carry runner command and repo path.** Format: `ID | file path | runner command | trigger`. Tester resolves stubs using the runner command directly.
- **Developer receives `test-design.md` as input.** Read-only reference; developer aligns implementation to test IDs. Fan-out is parallel; developer reads a snapshot of `test-design.md` at fan-out time, but cannot start implementation until `test-design.md ## Testability review` is `APPROVED`.

## Milestones

1. Role contracts + template sections defined. (This spec.)
2. Templates validated against one representative story (validation run).
3. `spec-to-code.yaml` node `outputs` declarations written from these templates.
4. Artifact validator updated with the `APPROVED`/`BLOCKED` sentinel checks for both gates.

## Open questions

1. **BA/architect domain model boundary.** Does `clarify.md` include DDD constructs (aggregates, bounded contexts) or only business rules and entities in plain language? Defer to validation run.
2. **Failed gate artifacts.** When architect writes `BLOCKED` on `clarify.md`, or test-arch writes `BLOCKED` on `design.md`, should the conflict list go into the challenger's artifact or a separate feedback file routed back to the challenged role? Current spec puts it in-artifact; this may conflate the challenger's forward-looking design with its backward-looking critique.
3. **Developer start timing.** Developer receives `test-design.md` at fan-out time, but the test-arch gate may not be resolved yet. Does developer block until `APPROVED`, or start with the snapshot and re-read if `design.md` is revised?
4. **review / test-report cross-reference.** Deliberately excluded from v1. Accept the information loss; revisit after validation run.

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

## Bounded contexts
[Domain partitions and their interfaces. "N/A — single context" is a valid entry.]

## Acceptance criteria
[Format: Given / When / Then. One criterion per lifecycle stage covered in Domain entities.]

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
[If BLOCKED, numbered list of conflicts or gaps in clarify.md. BA must resolve before fan-out.]

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
[If BLOCKED, numbered list of testability issues in design.md:
 untestable coupling, missing isolation boundaries, unobservable outputs.
 Architect must revise design.md before developer starts.]

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

Inputs: `clarify.md`, `design.md`, `test-design.md` (read-only, requires `## Testability review: APPROVED`).

```
## Summary
[What was built. One paragraph.]

## Code structure
[File / module layout. Tree form acceptable.]

## Key implementation decisions
[Choices that deviate from or extend design.md. Reference test IDs from test-design.md where alignment decisions were made.]

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

## Role memory updates
[STUB — schema undefined. Leave blank until memory/<role>.md format is specified.]
```
