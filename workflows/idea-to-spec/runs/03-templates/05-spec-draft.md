# 05 — Spec Draft

## Goal

Produce 18 self-contained template files for every artifact in the `spec-to-code` workflow, each with a visible instruction header and a fillable body.

## Non-goals

- Writing workflow YAML or executor prompts.
- Filling template content with actual run data.
- Building the validator (it reads section names defined here; implementation is separate).
- Designing the knowledge-base update mechanism beyond what the template captures.

## Approach

- **18 files**, flat in `workflows/spec-to-code/templates/` (KB files in a `knowledge-base/` subdirectory).
- **Structure per file:** `## Template guide` (metadata block) → `---` → fillable sections.
- **Validator contract:** (a) rejects artifacts containing `## Template guide`; (b) checks named sections exist and are non-empty; (c) for sentinel artifacts, checks first line of first section is `APPROVED` or `BLOCKED`.
- **Four artifact classes** handled differently in the guide:
  - *Sentinel*: gate artifacts — `domain-review.md`, `testability-review.md`, `code-review.md`, `test-review.md`.
  - *Freeform*: `clarify.md`, `design.md`, `test-design.md`, `implementation.md`, `test-implementation.md`, `final-review.md`, `reflect.md`.
  - *Machine-generated*: `test-results.md` — format spec, not fillable; role is `runner (automated)`.
  - *Human input*: `issue.md` — jargon-free, 5-minute fill time.
  - *Accumulating*: `knowledge-base/<role>.md` — guide stays in file permanently; executors append below existing content.

## Key decisions

- **Knowledge-base guide stays permanently in the file.** It becomes part of the live KB file. Executors appending new entries see the guide on every invocation and are reminded of the format. The guide is a header, not a disposable wrapper. (Resolves open question #1 from 04-decision.)
- **`test-results.md` stays in `templates/` not `schemas/`.** All artifact definitions live in one place. The guide's `role: runner (automated)` field signals the difference. (Resolves open question #2.)
- **`issue.md` template avoids DDD/workflow jargon.** It reads: "Describe what you want to build and why." Sections are: problem, context, and done-when. Human fills this before invoking the workflow.

## Milestones

1. All 18 template files defined with complete guide blocks and section names. (This spec.)
2. Validator implemented to check section existence, sentinel lines, and absence of `## Template guide`.
3. Templates referenced in `spec-to-code.yaml` node `outputs` declarations.
4. One full workflow run validates all templates against real executor output.

## Open questions

1. **Section name stability.** Once the YAML references a section name, renaming it is a breaking change. The spec should declare a section rename policy (e.g., deprecation period, aliasing). Deferred to workflow YAML spec run.
2. **KB file initial state.** When a new project starts, does the orchestrator copy the template KB files into the project's `knowledge-base/` directory, or does it reference the shared template? If projects share the same KB file, runs contaminate each other.
3. **Sentinel case sensitivity.** Spec says `APPROVED` / `BLOCKED` are case-sensitive. Should the validator be lenient (case-insensitive) to protect against executor capitalization errors, or strict to enforce discipline? Current spec: strict.

---

## Appendix — Complete template file contents

---

### `issue.md` (human input)

```markdown
## Template guide
role:        human (issue author)
node:        input — provided before workflow starts
inputs:      none
prerequisite: none
sentinel:    no
quality:     A well-written issue gives the BA enough context to model the domain without back-and-forth.
do-not:
  - Do not describe the solution — describe the problem.
  - Do not assume technical knowledge in the reader.
  - Do not leave "done when" blank — without it, the workflow has no acceptance anchor.

---

## Problem
[What is going wrong or missing? One paragraph in plain language.]

## Context
[Who is affected and why does it matter? Any relevant background the BA needs to understand the domain.]

## Done when
[How will you know this is solved? 2–5 observable outcomes, no implementation details.]
```

---

### `clarify.md` (ba)

```markdown
## Template guide
role:        BA
node:        ba
inputs:      issue.md, knowledge-base/ba.md
prerequisite: none
sentinel:    no
quality:     A well-filled clarify.md gives the architect a domain model with no internal contradictions,
             full lifecycle coverage, and bounded contexts clear enough to evaluate orthogonality.
do-not:
  - Do not describe implementation or technology — that is the architect's job.
  - Do not write business rules that contradict each other — architect will return BLOCKED.
  - Do not leave Lifecycle coverage rows empty — incompleteness is grounds for BLOCKED.

---

## Problem statement
[What the software must do in business terms. One paragraph.]

## Domain entities
[Table: entity name | brief definition | key attributes]

## Business rules
[Numbered list. Each rule independently statable.]

## Lifecycle coverage
[Table — all four rows required:
 Phase     | Key events | Rules that apply | Exit condition
 Create    |            |                  |
 Operate   |            |                  |
 Edge case |            |                  |
 Terminate |            |                  |]

## Bounded contexts
[Domain partitions and their interfaces. Write "N/A — single context" if not applicable.]

## Acceptance criteria
[Given / When / Then. One criterion per lifecycle phase.]

## Open questions
[Unresolved business questions for the architect to evaluate. Write "None" if not applicable.]
```

---

### `domain-review.md` (architect — gate)

```markdown
## Template guide
role:        Architect (reviewer mode)
node:        architect_domain_gate
inputs:      clarify.md, knowledge-base/architect.md
prerequisite: none
sentinel:    yes — first line of ## Verdict must be exactly APPROVED or BLOCKED (case-sensitive)
quality:     A well-written BLOCKED lists specific, actionable conflicts. A well-written APPROVED
             confirms which criteria were checked, not just "looks fine."
do-not:
  - Do not write APPROVED without checking all three criteria: non-contradiction, lifecycle completeness,
    bounded context orthogonality.
  - Do not write vague issues ("the model is unclear") — name the specific rule or section.
  - Do not propose solutions — state the problem only; BA resolves.

---

## Verdict
APPROVED
[Replace with BLOCKED if any criterion fails.]

## Criteria checked
[Three rows required:
 Criterion                          | Result      | Evidence
 No contradicting business rules    | pass / fail | rule numbers if fail
 Lifecycle coverage complete        | pass / fail | missing phases if fail
 Bounded contexts orthogonal        | pass / fail | coupling found if fail]

## Issues
[If APPROVED: write "None."
 If BLOCKED: numbered list. Each item: section in clarify.md | what the issue is | why it blocks.]
```

---

### `design.md` (architect)

```markdown
## Template guide
role:        Architect (designer mode)
node:        architect
inputs:      clarify.md, domain-review.md, knowledge-base/architect.md
prerequisite: domain-review.md ## Verdict must be APPROVED
sentinel:    no
quality:     A well-filled design.md gives the test-arch enough structure to evaluate testability
             and gives the developer a DDD blueprint with no ambiguous ownership boundaries.
do-not:
  - Do not make technology choices without justification — every choice must name what it rejected.
  - Do not leave ## Trade-offs made empty — a design with no trade-offs hid them.
  - Do not describe implementation details (loops, conditionals, data structures) — that is the developer's job.

---

## DDD structure
[Table: name | type (aggregate/entity/value-object/service) | responsibility | bounded context]

## System architecture
[Layer structure: domain / application / infrastructure / interface.
 Major components and their relationships. ASCII diagram acceptable.]

## Technology choices
[Table: concern | choice | one-line justification | rejected alternative]

## Integration points
[External systems, APIs, event buses, queues. Write "None" if not applicable.]

## Trade-offs made
[Numbered list. Each item: what was sacrificed | what was gained | why this bet was made.
 At least one entry required.]

## Open questions for test-arch and developer
[Write "None" if not applicable.]
```

---

### `testability-review.md` (test-arch — gate)

```markdown
## Template guide
role:        Test-Architect (reviewer mode)
node:        test_arch_testability_gate
inputs:      design.md, clarify.md, knowledge-base/test-arch.md
prerequisite: domain-review.md ## Verdict must be APPROVED
sentinel:    yes — first line of ## Verdict must be exactly APPROVED or BLOCKED (case-sensitive)
quality:     A well-written BLOCKED names specific aggregates, dependencies, or side-effects that
             fail the criteria — not general concerns about testability.
do-not:
  - Do not approve a design with untestable aggregates (no observable output).
  - Do not approve a design with direct cross-context dependencies (violates isolation boundary).
  - Do not propose fixes — name the criterion that fails; architect resolves.

---

## Verdict
APPROVED
[Replace with BLOCKED if any criterion fails.]

## Criteria evaluation
[Three rows required:
 Criterion                                              | Result      | Evidence
 Every aggregate has ≥1 observable output              | pass / fail | failing aggregates if fail
 No direct cross-context dependency without interface  | pass / fail | dependency path if fail
 Every external side-effect is injectable/replaceable  | pass / fail | side-effects at risk if fail]

## Issues
[If APPROVED: write "None."
 If BLOCKED: numbered list. Each item: criterion failed | location in design.md | what architect must change.]
```

---

### `test-design.md` (test-arch)

```markdown
## Template guide
role:        Test-Architect (designer mode)
node:        test_arch
inputs:      design.md, clarify.md, testability-review.md, knowledge-base/test-arch.md
prerequisite: testability-review.md ## Verdict must be APPROVED
sentinel:    no
quality:     A well-filled test-design.md gives the developer test IDs to align TDD against and
             gives the test-developer a regression suite with runnable stubs.
do-not:
  - Do not assign all tests to one owner — developer owns unit tests, test-developer owns integration/E2E.
  - Do not write regression stubs without a runner command — stubs without commands are unusable.
  - Do not skip edge cases — the validator checks this section is non-empty.

---

## Test strategy
[Unit / integration / E2E scope. Who owns what.
 Developer: unit tests (TDD). Test-developer: integration + E2E.]

## Feature test cases
[Table: ID | scenario | precondition | action | expected outcome | owner (developer / test-developer)]

## Regression suite
[Table: ID | repo-relative file path | runner command | trigger (every-commit / every-release)]

## Edge cases and failure scenarios
[Conditions not covered by feature test cases. At least one entry required.]

## Test tooling
[Frameworks and runners. Version-pinned.]
```

---

### `implementation.md` (developer)

```markdown
## Template guide
role:        Developer (TDD)
node:        developer
inputs:      design.md, test-design.md (read-only), clarify.md, knowledge-base/developer.md
prerequisite: testability-review.md ## Verdict must be APPROVED
sentinel:    no
quality:     A well-filled implementation.md lets the architect verify DDD adherence without
             reading every source file — the TDD trace shows which test IDs drove which decisions.
do-not:
  - Do not describe every line of code — summarise structure and decisions only.
  - Do not leave TDD trace empty if test-design.md had IDs — each deviation from design.md needs a trace entry.
  - Do not mark limitations as "TODO" without stating whether they are intentional tech debt or bugs.

---

## Summary
[What was built. One paragraph.]

## Code structure
[File / module layout. Tree form.]

## TDD trace
[Table: test ID from test-design.md | implementation decision it drove | rationale
 Write "None" only if test-design.md had no IDs that influenced implementation.]

## Key implementation decisions
[Choices that deviate from or extend design.md. Reference DDD structure where relevant.]

## Known limitations
[Intentional tech debt, deferred items, known bugs. Distinguish clearly between tech debt and bugs.]
```

---

### `test-implementation.md` (test-developer)

```markdown
## Template guide
role:        Test-Developer
node:        test_developer
inputs:      test-design.md, design.md, knowledge-base/test-developer.md
prerequisite: testability-review.md ## Verdict must be APPROVED
sentinel:    no
quality:     A well-filled test-implementation.md maps every test ID from test-design.md to an
             implementation status — no test ID should be unaccounted for.
do-not:
  - Do not implement tests that are already in scope for the developer (unit tests) — own integration/E2E only.
  - Do not mark tests "skipped" without a reason.
  - Do not omit the coverage map — the test-arch reviewer uses it to evaluate completeness.

---

## Summary
[What was implemented. Which test IDs are covered.]

## Test structure
[File layout of tests/integration/ and tests/e2e/. Tree form.]

## Coverage map
[Table: test ID | file path | status (implemented / skipped / blocked) | reason if skipped/blocked]

## Notes
[Any implementation choices, workarounds, or deviations from test-design.md.]
```

---

### `test-results.md` (test-execution — machine-generated)

```markdown
## Template guide
role:        runner (automated — not LLM-filled)
node:        test_execution
inputs:      tests/unit/, tests/integration/, tests/e2e/, test-design.md (for regression stub IDs)
prerequisite: none (fires after both developer and test-developer complete)
sentinel:    no
quality:     Complete run metadata, every failure with a stack trace, every regression stub accounted for.
do-not:
  - Do not omit failed tests — all failures must appear in ## Failures.
  - Do not omit the regression suite status — each stub ID from test-design.md must have a row.
  - Do not truncate stack traces beyond 20 lines.

---

## Run metadata
runner:    [runner name and version]
timestamp: [ISO 8601]
commit:    [SHA]

## Summary
passed:  N
failed:  M
skipped: K
coverage: X%

## Failures
[Numbered list: test ID | file:line | error message | stack trace (max 20 lines)]

## Regression suite status
[Table: stub ID | file path | result (passed / failed / not-run) | failure detail if applicable]
```

---

### `code-review.md` (architect — gate)

```markdown
## Template guide
role:        Architect (reviewer mode)
node:        architect_code_review
inputs:      implementation.md, src/, design.md, clarify.md, knowledge-base/architect.md
prerequisite: none (fires after developer completes, parallel to test_arch_test_review)
sentinel:    yes — first line of ## Verdict must be exactly APPROVED or BLOCKED (case-sensitive)
quality:     A well-written BLOCKED names specific DDD violations with file:line references.
             APPROVED confirms DDD adherence; it does not assess general code quality (that is final_review).
do-not:
  - Do not assess general code quality, style, or security — scope is DDD adherence only.
  - Do not approve if domain logic is found in infrastructure or interface layers.
  - Do not approve if bounded context boundaries are crossed without a declared interface.

---

## Verdict
APPROVED
[Replace with BLOCKED if any DDD criterion fails.]

## DDD adherence assessment
[One paragraph. Bounded context integrity, aggregate invariant enforcement,
 domain logic placement across layers.]

## Findings
[If APPROVED: write "None" or list minor observations (not blockers).
 If BLOCKED: numbered list. Each item: location (file:line) | DDD violation | severity (blocker/major/minor)]
```

---

### `test-review.md` (test-arch — gate)

```markdown
## Template guide
role:        Test-Architect (reviewer mode)
node:        test_arch_test_review
inputs:      test-results.md, test-design.md, test-implementation.md, knowledge-base/test-arch.md
prerequisite: none (fires after test_execution, parallel to architect_code_review)
sentinel:    yes — first line of ## Verdict must be exactly APPROVED or BLOCKED (case-sensitive)
quality:     A well-written review states the regression pass rate, names every blocker failure,
             and identifies gaps between test-design.md and what was actually executed.
do-not:
  - Do not approve if any regression stub from test-design.md is "not-run" without explanation.
  - Do not approve if blocker-severity failures exist in test-results.md.
  - Do not flag minor test quality issues as BLOCKED — severity judgment matters.

---

## Verdict
APPROVED
[Replace with BLOCKED if blocker failures exist or regression suite is incomplete.]

## Coverage assessment
Feature test case pass rate:  X / N passed
Regression suite pass rate:   X / N passed

## Findings
[If APPROVED: write "None" or list minor observations.
 If BLOCKED: numbered list. Each item: test ID | issue | severity (blocker/major/minor)]

## Gaps
[Test IDs from test-design.md with no implementation or persistent failures. Write "None" if not applicable.]
```

---

### `final-review.md` (final-review)

```markdown
## Template guide
role:        Reviewer (generalist)
node:        final_review
inputs:      implementation.md, code-review.md, test-results.md, test-review.md, clarify.md, design.md
prerequisite: code-review.md ## Verdict must be APPROVED AND test-review.md ## Verdict must be APPROVED
sentinel:    no
quality:     A well-filled final-review.md covers what the specialist reviewers did not: code quality,
             security, maintainability, UX. It does not re-litigate DDD adherence or test coverage.
do-not:
  - Do not re-assess DDD adherence — code-review.md already covered that.
  - Do not re-assess test coverage — test-review.md already covered that.
  - Do not leave ## Outstanding items empty — write "None" explicitly if no items remain.

---

## Summary
[Overall assessment of the deliverable. One paragraph.]

## Findings
[Numbered list: concern | location | severity (blocker / major / minor)
 Scope: code quality, security, maintainability, UX, cross-cutting concerns.]

## Outstanding items
[Issues not resolved in this run. Table: item | owner | disposition (block-ship / tech-debt / wontfix)
 Write "None" if not applicable.]

## Verdict
[One of: "Ship" / "Ship with caveats" / "Do not ship — rework required"]
```

---

### `reflect.md` (reflect)

```markdown
## Template guide
role:        Meta (retrospective)
node:        reflect
inputs:      all artifacts from this run, knowledge-base/ (current state)
prerequisite: final_review completes (non-blocking — workflow is done regardless)
sentinel:    no
quality:     A well-filled reflect.md produces actionable carry-forward entries, not general praise.
             Gate statistics surface systematic problems (a gate that retried 3 times every run needs
             a prompt fix, not another run).
do-not:
  - Do not write carry-forward entries without a reason.
  - Do not leave Gate statistics empty — even a clean run should show 0 retries.
  - Do not update knowledge-base files directly from this template — list updates here;
    the orchestrator or human applies them.

---

## Run summary
[One paragraph: what was built, how many cycles each gate triggered.]

## What worked
[By role: practices or sections that produced high-quality output. Be specific.]

## What degraded
[By role: sections that were thin, skipped, or produced noise. Be specific.]

## Gate statistics
[Table: gate | first-attempt result | total retries | max retries hit?
 architect_domain_gate      | ... | ... | ...
 test_arch_testability_gate | ... | ... | ...
 architect_code_review      | ... | ... | ...
 test_arch_test_review      | ... | ... | ...]

## Carry-forward
[Table: role | change to role prompt or template section | reason]

## Knowledge base updates
[Table: file | entry type (heuristic / anti-pattern) | content
 Write "None" if this run produced no new signal worth persisting.]
```

---

### `knowledge-base/ba.md`

```markdown
## Template guide
role:        BA
node:        ba (read on every invocation; updated by reflect after each run)
This file is permanent — do not strip this guide. Append new entries below existing ones.
Sections below are the live knowledge base. Do not overwrite; append only.

---

## Role prompt
You are the BA for this project. Your job is to produce a domain model in clarify.md.
Evaluate your output against three criteria before submitting:
1. No two business rules contradict each other.
2. All four lifecycle phases are covered (Create / Operate / Edge case / Terminate).
3. Bounded contexts are stated even if there is only one.

## Heuristics learned
[Append entries as: - [run date] <observation>]

## Anti-patterns to avoid
[Append entries as: - [run date] <anti-pattern> — <why it fails>]
```

---

### `knowledge-base/architect.md`

```markdown
## Template guide
role:        Architect
node:        architect_domain_gate, architect, architect_code_review (read on all three invocations)
This file is permanent — do not strip this guide. Append new entries below existing ones.

---

## Role prompt
You are the Architect for this project. You are invoked three times:
1. Domain gate: review clarify.md for non-contradiction, lifecycle completeness, and bounded context orthogonality.
2. Design: transform the approved domain model into a DDD code structure.
3. Code review: verify implementation.md adheres to the DDD structure in design.md.
In all three modes, scope your work to your axis — do not assess what belongs to another role.

## Heuristics learned
[Append entries as: - [run date] <observation>]

## Anti-patterns to avoid
[Append entries as: - [run date] <anti-pattern> — <why it fails>]
```

---

### `knowledge-base/test-arch.md`

```markdown
## Template guide
role:        Test-Architect
node:        test_arch_testability_gate, test_arch, test_arch_test_review (read on all three invocations)
This file is permanent — do not strip this guide. Append new entries below existing ones.

---

## Role prompt
You are the Test-Architect for this project. You are invoked three times:
1. Testability gate: review design.md against three criteria (observable outputs, interface isolation,
   injectable side-effects). Write APPROVED or BLOCKED.
2. Test design: produce test-design.md — lifecycle safeguards, feature test cases, regression suite.
3. Test review: review test-results.md against test-design.md. Write APPROVED or BLOCKED.
In all three modes, scope your work to testability and test quality — not code correctness or DDD adherence.

## Heuristics learned
[Append entries as: - [run date] <observation>]

## Anti-patterns to avoid
[Append entries as: - [run date] <anti-pattern> — <why it fails>]
```

---

### `knowledge-base/developer.md`

```markdown
## Template guide
role:        Developer
node:        developer (read on every invocation)
This file is permanent — do not strip this guide. Append new entries below existing ones.

---

## Role prompt
You are the Developer for this project. You write code TDD-style: read the test IDs in test-design.md,
write unit tests first, then make them pass. Follow the DDD structure in design.md.
Fill implementation.md with your summary, code structure, TDD trace, and known limitations.
Do not design tests beyond unit tests — integration and E2E are owned by the test-developer.

## Heuristics learned
[Append entries as: - [run date] <observation>]

## Anti-patterns to avoid
[Append entries as: - [run date] <anti-pattern> — <why it fails>]
```

---

### `knowledge-base/test-developer.md`

```markdown
## Template guide
role:        Test-Developer
node:        test_developer (read on every invocation)
This file is permanent — do not strip this guide. Append new entries below existing ones.

---

## Role prompt
You are the Test-Developer for this project. You implement integration and E2E tests from test-design.md.
You do not write unit tests — those are the developer's responsibility.
Fill test-implementation.md with a coverage map: every test ID from test-design.md must have a row.
Write runnable test code in tests/integration/ and tests/e2e/ following the runner commands in test-design.md.

## Heuristics learned
[Append entries as: - [run date] <observation>]

## Anti-patterns to avoid
[Append entries as: - [run date] <anti-pattern> — <why it fails>]
```
