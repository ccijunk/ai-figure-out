# 07 — Spec (final)

## Goal

18 self-contained template files for every artifact in the `spec-to-code` workflow. Each template is fillable by an executor without reading the workflow spec.

## Non-goals

- Workflow YAML, executor prompts, validator implementation.
- Filling templates with real run data.
- Knowledge-base archival / rotation mechanism (acknowledged as an open risk; deferred).

## Approach

- **18 files** in `workflows/spec-to-code/templates/` (KB files in a `knowledge-base/` subdirectory).
- **Structure per file:** `## Template guide` block → `---` → fillable sections.
- **Validator contract:** (a) checks named sections exist and are non-empty in the artifact; (b) for sentinel artifacts, checks first line of first fillable section is exactly `APPROVED` or `BLOCKED`; (c) does NOT check for the presence or absence of `## Template guide` (brittle — see Resolutions #1).
- **Four artifact classes:**
  - *Sentinel* (gate artifacts): `domain-review.md`, `testability-review.md`, `code-review.md`, `test-review.md`.
  - *Freeform*: `clarify.md`, `design.md`, `test-design.md`, `implementation.md`, `test-implementation.md`, `final-review.md`, `reflect.md`.
  - *Format spec* (machine-generated): `test-results.md`.
  - *Human input*: `issue.md`.
  - *Accumulating*: `knowledge-base/<role>.md`.

## Key decisions

- **Validator does not check for `## Template guide` in output.** The rejection signal was brittle (a quoted mention in `reflect.md` would cause a false rejection). Role prompts instruct executors to omit the guide; enforcement is behavioral, not mechanical.
- **KB files have a soft size limit.** Each KB file's guide states: "When this file exceeds 100 lines, move entries older than 5 runs to `knowledge-base/archive/<role>.md`." This gives executors a concrete archival trigger. The orchestrator is not responsible for archiving.
- **`test-results.md` guide uses `format-contract` instead of `do-not`.** Rules for machine-generated output are labeled `format-contract` — constraints on the runner's output schema, not behavioral guidance for an LLM.
- **`final-review.md` prerequisite clarified.** The template guide states the prerequisite as context for the executor, but notes it is enforced by the orchestrator (join gate), not the validator.
- **KB update actor is `reflect` directly.** Reflect writes to `knowledge-base/<role>.md` files. The `reflect.md` artifact lists what was updated so the run is auditable. Contradiction between draft and KB template is resolved: both say reflect writes.

## Milestones

1. All 18 template files defined. (This spec.)
2. Validator implemented: section existence + sentinel checks.
3. Templates referenced in `spec-to-code.yaml` node `outputs` declarations.
4. One full workflow run validates templates against real executor output.

## Open questions

1. **KB archival timing.** The 100-line soft limit gives executors a trigger, but "move to archive" requires a write outside the KB file. Does the executor do this during a run, or does a maintenance task handle it? Deferred.
2. **Section rename policy.** Renaming a section in a template breaks the validator. Define a deprecation period (e.g., old name accepted for 3 runs, then dropped). Deferred to workflow YAML spec run.
3. **`issue.md` validation.** The human fills `issue.md` before the workflow starts. Should the BA be able to return feedback on a thin issue, or does the BA have to work with whatever is provided? Currently: BA works with it; thin issues produce thin `clarify.md` which architect rejects.

---

## Template files (complete content)

---

### `issue.md`

```markdown
## Template guide
role:        human (issue author)
node:        input — provided before workflow starts
inputs:      none
prerequisite: none
sentinel:    no
quality:     Enough context for the BA to model the domain without back-and-forth.
             If "done when" is missing, the BA has no acceptance anchor.
do-not:
  - Do not describe the solution — describe the problem.
  - Do not assume the reader has technical context about the codebase.
  - Do not leave "done when" blank.

---

## Problem
[What is going wrong or missing? One paragraph.]

## Context
[Who is affected and why does it matter? Background the BA needs to understand the domain.]

## Done when
[How will you know this is solved? 2–5 observable outcomes, no implementation details.]
```

---

### `clarify.md`

```markdown
## Template guide
role:        BA
node:        ba
inputs:      issue.md, knowledge-base/ba.md
prerequisite: none
sentinel:    no
quality:     Architect can evaluate non-contradiction, lifecycle completeness, and bounded context
             orthogonality from this artifact alone — without asking follow-up questions.
do-not:
  - Do not make technology choices — that is the architect's job.
  - Do not write business rules that contradict each other.
  - Do not leave any Lifecycle coverage row empty.

---

## Problem statement
[What the software must do in business terms. One paragraph.]

## Domain entities
[Table: entity name | brief definition | key attributes]

## Business rules
[Numbered list. Each rule independently statable.]

## Lifecycle coverage
[All four rows required:
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
[Write "None" if not applicable.]
```

---

### `domain-review.md`

```markdown
## Template guide
role:        Architect (reviewer mode)
node:        architect_domain_gate
inputs:      clarify.md, knowledge-base/architect.md
prerequisite: none
sentinel:    yes — first line of ## Verdict must be exactly APPROVED or BLOCKED
quality:     BLOCKED names specific rules or sections with actionable gaps.
             APPROVED confirms all three criteria passed with evidence — not just "looks fine."
do-not:
  - Do not write APPROVED without checking all three criteria explicitly.
  - Do not write vague issues ("the model is unclear") — name the rule number or section.
  - Do not propose solutions — state the gap only.

---

## Verdict
APPROVED

## Criteria checked
[Three rows required:
 Criterion                         | Result      | Evidence or "n/a"
 No contradicting business rules   | pass / fail |
 Lifecycle coverage complete       | pass / fail |
 Bounded contexts orthogonal       | pass / fail |]

## Issues
[Write "None" if APPROVED.
 If BLOCKED: numbered list — section in clarify.md | what the issue is | why it blocks.]
```

---

### `design.md`

```markdown
## Template guide
role:        Architect (designer mode)
node:        architect
inputs:      clarify.md, domain-review.md, knowledge-base/architect.md
prerequisite: domain-review.md ## Verdict must be APPROVED (enforced by orchestrator)
sentinel:    no
quality:     Test-arch can evaluate testability and developer can start TDD from this artifact alone.
             No ambiguous ownership boundaries. Every technology choice names what it rejected.
do-not:
  - Do not make technology choices without naming the rejected alternative.
  - Do not leave ## Trade-offs made empty — write at least one entry.
  - Do not describe implementation details (loops, data structures) — that is the developer's job.

---

## DDD structure
[Table: name | type (aggregate/entity/value-object/domain-service) | responsibility | bounded context]

## System architecture
[Layers: domain / application / infrastructure / interface.
 Components and relationships. ASCII diagram acceptable.]

## Technology choices
[Table: concern | choice | one-line justification | rejected alternative]

## Integration points
[External systems, APIs, events. Write "None" if not applicable.]

## Trade-offs made
[Numbered list: what was sacrificed | what was gained | why this bet was made.
 At least one entry. "No trade-offs" is not a valid entry.]

## Open questions for test-arch and developer
[Write "None" if not applicable.]
```

---

### `testability-review.md`

```markdown
## Template guide
role:        Test-Architect (reviewer mode)
node:        test_arch_testability_gate
inputs:      design.md, clarify.md, knowledge-base/test-arch.md
prerequisite: domain-review.md ## Verdict must be APPROVED (enforced by orchestrator)
sentinel:    yes — first line of ## Verdict must be exactly APPROVED or BLOCKED
quality:     BLOCKED names specific aggregates, dependency paths, or side-effects — not general concerns.
             APPROVED confirms all three criteria passed with evidence.
do-not:
  - Do not approve aggregates with no observable output.
  - Do not approve direct cross-context dependencies without a declared interface.
  - Do not propose fixes — name the criterion that fails; architect resolves.

---

## Verdict
APPROVED

## Criteria evaluation
[Three rows required:
 Criterion                                             | Result      | Evidence or "n/a"
 Every aggregate has ≥1 observable output             | pass / fail |
 No direct cross-context dependency without interface | pass / fail |
 Every external side-effect is injectable/replaceable | pass / fail |]

## Issues
[Write "None" if APPROVED.
 If BLOCKED: numbered list — criterion failed | location in design.md | what architect must change.]
```

---

### `test-design.md`

```markdown
## Template guide
role:        Test-Architect (designer mode)
node:        test_arch
inputs:      design.md, clarify.md, testability-review.md, knowledge-base/test-arch.md
prerequisite: testability-review.md ## Verdict must be APPROVED (enforced by orchestrator)
sentinel:    no
quality:     Developer can write TDD unit tests from the feature test cases. Test-developer can
             implement integration/E2E tests from the regression suite stubs.
do-not:
  - Do not assign integration/E2E tests to the developer — they own unit tests only.
  - Do not write regression stubs without a runner command.
  - Do not leave ## Edge cases and failure scenarios empty.

---

## Test strategy
[Unit / integration / E2E scope. Developer owns unit tests (TDD). Test-developer owns integration/E2E.]

## Feature test cases
[Table: ID | scenario | precondition | action | expected outcome | owner (developer / test-developer)]

## Regression suite
[Table: ID | repo-relative file path | runner command | trigger (every-commit / every-release)]

## Edge cases and failure scenarios
[At least one entry required.]

## Test tooling
[Frameworks and runners. Version-pinned.]
```

---

### `implementation.md`

```markdown
## Template guide
role:        Developer (TDD)
node:        developer
inputs:      design.md, test-design.md, clarify.md, knowledge-base/developer.md
prerequisite: testability-review.md ## Verdict must be APPROVED (enforced by orchestrator)
sentinel:    no
quality:     Architect can verify DDD adherence without reading every source file.
             TDD trace shows which test IDs drove which decisions.
do-not:
  - Do not describe every line of code — summarise structure and decisions only.
  - Do not leave TDD trace empty if test-design.md had IDs that influenced implementation.
  - Do not mark limitations "TODO" without stating whether they are tech debt or bugs.

---

## Summary
[What was built. One paragraph.]

## Code structure
[File / module layout. Tree form.]

## TDD trace
[Table: test ID | implementation decision it drove | rationale
 Write "None" only if no test IDs influenced any implementation decision.]

## Key implementation decisions
[Choices that deviate from or extend design.md.]

## Known limitations
[Tech debt and bugs. Label each clearly.]
```

---

### `test-implementation.md`

```markdown
## Template guide
role:        Test-Developer
node:        test_developer
inputs:      test-design.md, design.md, knowledge-base/test-developer.md
prerequisite: testability-review.md ## Verdict must be APPROVED (enforced by orchestrator)
sentinel:    no
quality:     Every test ID from test-design.md appears in the coverage map with a status.
             No test ID is unaccounted for.
do-not:
  - Do not implement unit tests — those are owned by the developer.
  - Do not mark tests "skipped" without a reason.
  - Do not omit the coverage map.

---

## Summary
[What was implemented. Which test IDs are covered.]

## Test structure
[File layout of tests/integration/ and tests/e2e/. Tree form.]

## Coverage map
[Table: test ID | file path | status (implemented / skipped / blocked) | reason if skipped/blocked]

## Notes
[Implementation choices, workarounds, or deviations from test-design.md. Write "None" if not applicable.]
```

---

### `test-results.md`

```markdown
## Template guide
role:        runner (automated — not LLM-filled)
node:        test_execution
inputs:      tests/unit/, tests/integration/, tests/e2e/, test-design.md (for regression stub IDs)
prerequisite: none (fires after developer and test-developer both complete)
sentinel:    no
format-contract:
  - All test IDs from test-design.md ## Regression suite must appear in ## Regression suite status.
  - All failures must appear in ## Failures with a stack trace (max 20 lines).
  - Run metadata fields (runner, timestamp, commit) are required and must be non-empty.

---

## Run metadata
runner:    
timestamp: 
commit:    

## Summary
passed:  
failed:  
skipped: 
coverage: 

## Failures
[Numbered list: test ID | file:line | error message | stack trace (max 20 lines)
 Write "None" if no failures.]

## Regression suite status
[Table: stub ID | file path | result (passed / failed / not-run) | failure detail if applicable]
```

---

### `code-review.md`

```markdown
## Template guide
role:        Architect (reviewer mode)
node:        architect_code_review
inputs:      implementation.md, src/, design.md, clarify.md, knowledge-base/architect.md
prerequisite: none (fires after developer completes; parallel to test_arch_test_review)
sentinel:    yes — first line of ## Verdict must be exactly APPROVED or BLOCKED
quality:     BLOCKED names specific DDD violations with file:line references.
             APPROVED does not re-assess general code quality — that is final_review's scope.
do-not:
  - Do not assess general code quality, style, or security — DDD adherence only.
  - Do not approve if domain logic is found in infrastructure or interface layers.
  - Do not approve if bounded context boundaries are crossed without a declared interface.

---

## Verdict
APPROVED

## DDD adherence assessment
[One paragraph: bounded context integrity, aggregate invariants, domain logic placement.]

## Findings
[Write "None" if APPROVED with no observations.
 If BLOCKED: numbered list — location (file:line) | DDD violation | severity (blocker/major/minor).]
```

---

### `test-review.md`

```markdown
## Template guide
role:        Test-Architect (reviewer mode)
node:        test_arch_test_review
inputs:      test-results.md, test-design.md, test-implementation.md, knowledge-base/test-arch.md
prerequisite: none (fires after test_execution; parallel to architect_code_review)
sentinel:    yes — first line of ## Verdict must be exactly APPROVED or BLOCKED
quality:     States regression pass rate and feature test pass rate explicitly.
             Distinguishes between "tests not implemented" and "tests failing."
do-not:
  - Do not approve if any regression stub from test-design.md is "not-run" without explanation.
  - Do not approve if blocker-severity failures exist in test-results.md.
  - Do not BLOCK for minor test quality issues that do not affect correctness.

---

## Verdict
APPROVED

## Coverage assessment
Feature test case pass rate: X / N
Regression suite pass rate:  X / N

## Findings
[Write "None" if APPROVED with no observations.
 If BLOCKED: numbered list — test ID | issue | severity (blocker/major/minor).]

## Gaps
[Test IDs from test-design.md with no implementation or persistent failures.
 Write "None" if not applicable.]
```

---

### `final-review.md`

```markdown
## Template guide
role:        Reviewer (generalist)
node:        final_review
inputs:      implementation.md, code-review.md, test-results.md, test-review.md, clarify.md, design.md
prerequisite: code-review.md APPROVED AND test-review.md APPROVED — enforced by orchestrator join gate,
              not by the validator. This node does not run until both signals arrive.
sentinel:    no
quality:     Covers what specialist reviewers did not: code quality, security, maintainability, UX.
             Does not re-litigate DDD adherence or test coverage.
do-not:
  - Do not re-assess DDD adherence — code-review.md covered that.
  - Do not re-assess test coverage — test-review.md covered that.
  - Do not leave ## Outstanding items blank — write "None" explicitly.

---

## Summary
[Overall assessment. One paragraph.]

## Findings
[Numbered list: concern | location | severity (blocker / major / minor)
 Scope: code quality, security, maintainability, UX, cross-cutting concerns.]

## Outstanding items
[Table: item | owner | disposition (block-ship / tech-debt / wontfix)
 Write "None" if not applicable.]

## Verdict
[One of: "Ship" / "Ship with caveats" / "Do not ship — rework required"]
```

---

### `reflect.md`

```markdown
## Template guide
role:        Meta (retrospective)
node:        reflect
inputs:      all artifacts from this run, knowledge-base/ (current state)
prerequisite: final_review completes (non-blocking — workflow complete regardless)
sentinel:    no
quality:     Carry-forward entries are specific and actionable. Gate statistics surface systematic
             problems. KB updates are written to the KB files directly by this role.
do-not:
  - Do not write carry-forward entries without a reason.
  - Do not leave Gate statistics empty — a clean run should show 0 retries.
  - Do not defer KB updates to "orchestrator or human" — write them to knowledge-base/<role>.md directly.

---

## Run summary
[What was built. Gate cycle counts per gate.]

## What worked
[By role: specific practices or sections that produced high-quality output.]

## What degraded
[By role: specific sections that were thin, skipped, or noisy.]

## Gate statistics
[Table:
 Gate                       | First-attempt result | Total retries | Max retries hit?
 architect_domain_gate      |                      |               |
 test_arch_testability_gate |                      |               |
 architect_code_review      |                      |               |
 test_arch_test_review      |                      |               |]

## Carry-forward
[Table: role | change to role prompt or template section | reason
 Write "None" if no changes needed.]

## Knowledge base updates written
[Table: file updated | entry type (heuristic / anti-pattern) | content summary
 Write "None" if no KB entries were added this run.]
```

---

### `knowledge-base/ba.md`

```markdown
## Template guide
role:        BA
node:        ba (read on every invocation; updated by reflect after runs)
This file is permanent — do not strip this guide. Append entries; do not overwrite.
Size limit: when this file exceeds 100 lines, move entries from runs older than 5 runs to
knowledge-base/archive/ba.md before appending new ones.

---

## Role prompt
You are the BA for this project. Produce clarify.md.
Before submitting, verify:
1. No two business rules contradict each other.
2. All four lifecycle phases have entries (Create / Operate / Edge case / Terminate).
3. Bounded contexts are stated, even if there is only one.

## Heuristics learned
[Append as: - [YYYY-MM-DD] <observation>]

## Anti-patterns to avoid
[Append as: - [YYYY-MM-DD] <anti-pattern> — <why it fails>]
```

---

### `knowledge-base/architect.md`

```markdown
## Template guide
role:        Architect
node:        architect_domain_gate, architect, architect_code_review (read on all three invocations)
This file is permanent — do not strip this guide. Append entries; do not overwrite.
Size limit: when this file exceeds 100 lines, move entries from runs older than 5 runs to
knowledge-base/archive/architect.md.

---

## Role prompt
You are the Architect for this project. You are invoked in three modes:
1. Domain gate: review clarify.md. Check: non-contradiction, lifecycle completeness, bounded context
   orthogonality. Write domain-review.md with APPROVED or BLOCKED.
2. Design: transform the approved domain model into a DDD code structure. Write design.md.
3. Code review: verify implementation.md adheres to design.md's DDD structure. Write code-review.md.
In all three modes, stay within your axis — do not assess testability (test-arch) or code quality (final-review).

## Heuristics learned
[Append as: - [YYYY-MM-DD] <observation>]

## Anti-patterns to avoid
[Append as: - [YYYY-MM-DD] <anti-pattern> — <why it fails>]
```

---

### `knowledge-base/test-arch.md`

```markdown
## Template guide
role:        Test-Architect
node:        test_arch_testability_gate, test_arch, test_arch_test_review (read on all three invocations)
This file is permanent — do not strip this guide. Append entries; do not overwrite.
Size limit: when this file exceeds 100 lines, move entries from runs older than 5 runs to
knowledge-base/archive/test-arch.md.

---

## Role prompt
You are the Test-Architect for this project. You are invoked in three modes:
1. Testability gate: review design.md. Three criteria must all pass for APPROVED:
   (a) every aggregate has ≥1 observable output,
   (b) no direct cross-context dependency without a declared interface,
   (c) every external side-effect is injectable or replaceable.
2. Test design: produce test-design.md — lifecycle safeguards, feature test cases, regression suite.
   Developer owns unit tests. Test-developer owns integration/E2E.
3. Test review: review test-results.md against test-design.md. Write test-review.md with APPROVED or BLOCKED.
Stay within your axis — do not assess DDD adherence (architect) or code quality (final-review).

## Heuristics learned
[Append as: - [YYYY-MM-DD] <observation>]

## Anti-patterns to avoid
[Append as: - [YYYY-MM-DD] <anti-pattern> — <why it fails>]
```

---

### `knowledge-base/developer.md`

```markdown
## Template guide
role:        Developer
node:        developer (read on every invocation)
This file is permanent — do not strip this guide. Append entries; do not overwrite.
Size limit: when this file exceeds 100 lines, move entries from runs older than 5 runs to
knowledge-base/archive/developer.md.

---

## Role prompt
You are the Developer for this project. Write code TDD-style:
read the test IDs in test-design.md, write unit tests first, then make them pass.
Follow the DDD structure in design.md.
Fill implementation.md with your summary, code structure, TDD trace, and known limitations.
Do not write integration or E2E tests — those are the test-developer's responsibility.

## Heuristics learned
[Append as: - [YYYY-MM-DD] <observation>]

## Anti-patterns to avoid
[Append as: - [YYYY-MM-DD] <anti-pattern> — <why it fails>]
```

---

### `knowledge-base/test-developer.md`

```markdown
## Template guide
role:        Test-Developer
node:        test_developer (read on every invocation)
This file is permanent — do not strip this guide. Append entries; do not overwrite.
Size limit: when this file exceeds 100 lines, move entries from runs older than 5 runs to
knowledge-base/archive/test-developer.md.

---

## Role prompt
You are the Test-Developer for this project. Implement integration and E2E tests from test-design.md.
You do not write unit tests — those are the developer's responsibility.
Fill test-implementation.md with a coverage map: every test ID from test-design.md must have a row.
Write runnable test code in tests/integration/ and tests/e2e/ using the runner commands in test-design.md.

## Heuristics learned
[Append as: - [YYYY-MM-DD] <observation>]

## Anti-patterns to avoid
[Append as: - [YYYY-MM-DD] <anti-pattern> — <why it fails>]
```

---

## Resolutions

**#1 — `## Template guide` validator rejection is brittle (major)**
`fixed` — Validator no longer checks for the presence of `## Template guide` in output. Enforcement moves to role prompts ("omit the guide section from your output"). Role prompt guidance is in each KB file's `## Role prompt` section.

**#2 — KB file has no size bound — context budget risk (major)**
`fixed` — Each KB template guide now states: "when this file exceeds 100 lines, move entries from runs older than 5 runs to `knowledge-base/archive/<role>.md`." The executor (reflect) is responsible for archiving before appending. 100 lines is a soft heuristic, not a hard limit; it keeps the file within typical context budgets.

**#3 — `test-results.md` `do-not` rules target a machine-generated file (minor)**
`fixed` — Rules renamed from `do-not` to `format-contract`. The distinction signals these are output format requirements for the runner, not behavioral instructions for an LLM.

**#4 — `final-review.md` prerequisite implies validator handles a cross-artifact join (minor)**
`fixed` — Template guide now explicitly states: "enforced by orchestrator join gate, not by the validator." The validator checks only this artifact's own sections.

**#5 — `reflect.md` contradicts KB update actor (minor)**
`fixed` — `reflect.md` guide now says "write them to knowledge-base/<role>.md directly." KB template guides say "updated by reflect after runs." Both say the same actor. No contradiction.

**#6 — `design.md` trade-offs enforcement is illusory (minor)**
`rejected` — The critique is correct that "non-empty" validation cannot detect "no significant trade-offs were made" as a dodge. But semantic validation is explicitly out of scope for v1. The `do-not` rule and the `## Trade-offs made` section name are sufficient signal — an executor that writes a vacuous entry is failing the role prompt, not the validator. Rejecting this keeps v1's validation model consistent.
