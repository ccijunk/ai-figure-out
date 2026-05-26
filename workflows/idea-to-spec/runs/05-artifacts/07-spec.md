# 07 — Spec (final)

## Goal

Complete artifact schemas for every file produced or consumed by the `spec-to-code` workflow: sections, format contract, producer, consumer, and validation surface.

## Non-goals

- Agent prompts (`prompts/` files) — come after schemas.
- Bash script implementations.
- Workflow topology changes.
- Flowctl executor mechanics.

---

## Artifact classes

| Class | Files | Validator |
|---|---|---|
| Script-produced | `requirement.md`, `issue.md`, `repo-root.txt`, `branch-name.txt`, `pr-url.txt` | format only |
| Human input | `issue-url.txt` | presence only |
| Human gate | `verdict.txt`, `domain-review.md`, `testability-review.md`, `code-review.md`, `test-review.md` | verdict format; `## Rejection reason` when BLOCKED |
| AI-produced | `clarify.md`, `design.md`, `test-design.md`, `implementation.md`, `test-results.md`, `final-review.md`, `reflect.md` | named sections non-empty |
| Persistent memory | `workflow:memory/<role>.md` × 6 | append safety only |

---

## YAML fixes (from gap analysis)

1. `fetch_issue` outputs: add `issue: issue.md`.
2. `ba`, `architect`, `developer`, `test_developer` inputs: remove `reject_reason: reject-reason.txt`.
3. `reflect` inputs: add `memory_reviewer: workflow:memory/reviewer.md`.

---

## Schema definitions

---

### `issue-url.txt`
**Producer:** human (pre-run)  
**Consumers:** `fetch_issue`, `create_branch`  
**Format:** single line, valid GitHub issue URL, no trailing whitespace.  
**Example:** `https://github.com/org/repo/issues/42`

---

### `requirement.md`
**Producer:** `fetch-issue.sh`  
**Consumers:** `create_branch`, `ba`, `create_pr`

```
## Title
[Issue title from GitHub]

## Body
[Full issue body markdown, verbatim from GitHub API]

## Labels
[Comma-separated label names, or "none"]

## Assignees
[Comma-separated GitHub usernames, or "none"]

## Milestone
[Milestone title, or "none"]
```

Validator: all 5 sections present and non-empty (except Labels/Assignees/Milestone which may be "none").

---

### `issue.md`
**Producer:** `fetch-issue.sh`  
**Consumers:** `ba`  
**Format:** raw GitHub issue markdown — title as `# <title>`, then body verbatim. No added sections. BA uses this for domain context; `requirement.md` is for structured metadata.

---

### `repo-root.txt`
**Producer:** `fetch-issue.sh`  
**Consumers:** `create_branch`, `developer`, `create_pr`  
**Format:** single line, absolute path to the cloned target repo root. No trailing slash. No trailing whitespace.  
**Example:** `/home/user/projects/my-app`

---

### `branch-name.txt`
**Producer:** `create-branch.sh`  
**Consumers:** `create_pr`  
**Format:** single line, valid git branch name. Convention: `issue/<number>-<slug>` (e.g., `issue/42-add-user-auth`). No trailing whitespace.

---

### `pr-url.txt`
**Producer:** `create-pr.sh`  
**Consumers:** terminal output (end of workflow)  
**Format:** single line, valid GitHub PR URL.

---

### `verdict.txt`
**Producer:** human gate nodes (`human_domain_gate`, `human_testability_gate`, `human_code_review`, `human_test_review`)  
**Consumer:** transition engine (routing)  
**Format:** exactly one of the following two lines, trimmed, no other content:
```
APPROVED
```
or
```
BLOCKED
```
**Lifecycle note:** `verdict.txt` is scoped to the gate's current review cycle. The orchestrator re-reads it fresh after each human gate node completes. Prior verdicts are not retained in this file — they are recorded in the gate's review `.md`. The file is safe to overwrite on retry.

---

### `domain-review.md`
**Producer:** `human_domain_gate` (human)  
**Consumers:** `ba` (retry feedback), `architect` (context)  

```
## Verdict
[Copy the verdict here as well: APPROVED or BLOCKED]

## Review notes
[Freeform. What was evaluated, what was found. Can be brief if APPROVED.]

## Rejection reason
[Required field. Write "N/A" if APPROVED.
 If BLOCKED: one focused paragraph stating what specifically must change in clarify.md
 for the next BA iteration to pass. Be directive — vague rejections produce poor retries.]
```

Validator: `## Rejection reason` section present. If `## Verdict` line contains `BLOCKED`, section body must not be `N/A` or empty.

---

### `testability-review.md`
**Producer:** `human_testability_gate` (human)  
**Consumers:** `architect` (retry feedback), `test_arch` (context)

```
## Verdict
[APPROVED or BLOCKED]

## Review notes
[What was evaluated. Which design sections were checked.]

## Rejection reason
[Required field. Write "N/A" if APPROVED.
 If BLOCKED: one focused paragraph — which testability criterion failed, which DDD component,
 what architect must change in design.md.]
```

Validator: same as `domain-review.md`.

---

### `code-review.md`
**Producer:** `human_code_review` (human)  
**Consumers:** `developer` (retry feedback), `final_review` (context)

```
## Verdict
[APPROVED or BLOCKED]

## Review notes
[Code quality, correctness, adherence to design.md. Freeform.]

## Rejection reason
[Required field. Write "N/A" if APPROVED.
 If BLOCKED: one focused paragraph — what specifically must change in the implementation
 for the next developer iteration to pass.]
```

Validator: same as `domain-review.md`.

---

### `test-review.md`
**Producer:** `human_test_review` (human)  
**Consumers:** `test_developer` (retry feedback), `final_review` (context)

```
## Verdict
[APPROVED or BLOCKED]

## Review notes
[Test coverage, test quality, which test IDs passed/failed. Freeform.]

## Rejection reason
[Required field. Write "N/A" if APPROVED.
 If BLOCKED: one focused paragraph — which test IDs failed or were missing, what
 test_developer must fix in the next iteration.]
```

Validator: same as `domain-review.md`.

---

### `clarify.md`
**Producer:** `ba` (opencode)  
**Consumers:** `human_domain_gate`, `architect`, `test_arch`, `reflect`

```
## Template guide
role:        BA
node:        ba
inputs:      issue.md, requirement.md, workflow:memory/ba.md
             (on retry: domain-review.md as feedback)
sentinel:    no
quality:     Architect can evaluate non-contradiction, lifecycle completeness, and bounded
             context orthogonality without asking follow-up questions.
do-not:
  - Do not make technology choices.
  - Do not write contradicting business rules.
  - Do not leave any Lifecycle coverage row empty.
  - On retry: do not ignore the Rejection reason in domain-review.md.

---

## Problem statement
[What the software must do in business terms. One paragraph.]

## Domain entities
[Table: entity name | brief definition | key attributes]

## Business rules
[Numbered list. Each rule independently statable.]

## Lifecycle coverage
Phase     | Key events | Rules that apply | Exit condition
Create    |            |                  |
Operate   |            |                  |
Edge case |            |                  |
Terminate |            |                  |

## Bounded contexts
[Domain partitions and their interfaces. Write "N/A — single context" if not applicable.]

## Acceptance criteria
[Given / When / Then. One criterion per lifecycle phase minimum.]

## Open questions
[Write "None" if not applicable.]
```

---

### `design.md`
**Producer:** `architect` (opencode)  
**Consumers:** `human_testability_gate`, `test_arch`, `developer`, `test_developer`, `final_review`, `reflect`

```
## Template guide
role:        Architect
node:        architect
inputs:      clarify.md, workflow:memory/architect.md, repo:ARCHITECTURE.md
             (on retry: testability-review.md as feedback)
sentinel:    no
quality:     Test-arch can evaluate testability and developer can start TDD from this alone.
             Every technology choice names what it rejected.
do-not:
  - Do not make technology choices without naming the rejected alternative.
  - Do not leave ## Trade-offs made empty.
  - Do not describe implementation details (loops, data structures).
  - On retry: address each criterion in testability-review.md ## Rejection reason explicitly.

---

## DDD structure
[Table: name | type (aggregate/entity/value-object/domain-service) | responsibility | bounded context]

## System architecture
[Layers: domain / application / infrastructure / interface.
 Components and relationships. ASCII diagram acceptable.]

## Technology choices
[Table: concern | choice | justification | rejected alternative]

## Integration points
[External systems, APIs, events. Write "None" if not applicable.]

## Trade-offs made
[Numbered list: what was sacrificed | what was gained | why.
 At least one entry required.]

## Open questions for test-arch and developer
[Write "None" if not applicable.]
```

---

### `test-design.md`
**Producer:** `test_arch` (opencode)  
**Consumers:** `developer`, `test_developer`, `final_review`, `reflect`

```
## Template guide
role:        Test-Architect
node:        test_arch
inputs:      design.md, clarify.md, workflow:memory/test-arch.md, repo:docs/sdet/TEST-ARCHITECTURE.md
sentinel:    no
quality:     Developer can write TDD unit tests from the feature test cases.
             Test-developer can implement integration/E2E from regression stubs.
do-not:
  - Do not assign integration/E2E tests to the developer.
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
**Producer:** `developer` (opencode)  
**Consumers:** `human_code_review`, `final_review`, `reflect`, `create_pr`

```
## Template guide
role:        Developer (TDD)
node:        developer
inputs:      design.md, test-design.md, workflow:memory/developer.md
             (on retry: code-review.md as feedback)
sentinel:    no
quality:     Architect can verify DDD adherence without reading every source file.
             TDD trace shows which test IDs drove which decisions.
do-not:
  - Do not describe every line of code — summarise structure and decisions only.
  - Do not leave TDD trace empty if test IDs influenced implementation.
  - On retry: address each point in code-review.md ## Rejection reason explicitly.

---

## Summary
[What was built. One paragraph.]

## Code structure
[File / module layout. Tree form.]

## TDD trace
[Table: test ID | implementation decision it drove | rationale
 Write "None" only if no test IDs influenced any implementation decision.]

## Key implementation decisions
[Choices that deviate from or extend design.md. Write "None" if not applicable.]

## Known limitations
[Tech debt and bugs. Label each clearly. Write "None" if not applicable.]
```

---

### `test-results.md`
**Producer:** `test_developer` (opencode)  
**Consumers:** `human_test_review`, `final_review`, `reflect`, `create_pr`

```
## Template guide
role:        Test-Developer
node:        test_developer
inputs:      test-design.md, design.md, workflow:memory/test-developer.md
             (on retry: test-review.md as feedback)
sentinel:    no
quality:     Human reviewer can determine pass/fail status for every test ID from test-design.md
             without running the tests themselves.
do-not:
  - Do not omit any test ID from test-design.md ## Feature test cases.
  - Do not write a narrative failure description — use the structured Failures format.
  - Do not mark tests skipped without a reason.
  - On retry: address each point in test-review.md ## Rejection reason explicitly.

---

## Run metadata
runner:    [framework name and version]
timestamp: [ISO 8601]
commit:    [git SHA, short form]

## Summary
passed:   [count]
failed:   [count]
skipped:  [count]
coverage: [percentage or "not measured"]

## Test results by ID
[Table: test ID | owner | status (passed / failed / skipped) | failure summary if failed]

## Failures
[Numbered list per failure:
 test ID:       [from test-design.md]
 file:line:     [test file and line number]
 error:         [error message, one line]
 stack trace:   [max 10 lines]
 Write "None" if no failures.]

## Regression suite status
[Table: stub ID | file path | result (passed / failed / not-run) | note if not-run]
```

---

### `final-review.md`
**Producer:** `final_review` (opencode)  
**Consumers:** `reflect`, `create_pr`

```
## Template guide
role:        Reviewer (generalist)
node:        final_review
inputs:      implementation.md, code-review.md, test-results.md, test-review.md, clarify.md,
             design.md, workflow:memory/reviewer.md
prerequisite: human_code_review APPROVED AND human_test_review APPROVED
              (enforced by orchestrator — both gate nodes must complete before this fires)
sentinel:    no
quality:     Covers what human gate reviews did not: code quality, security, maintainability,
             cross-cutting concerns. Does not re-litigate decisions already reviewed.
do-not:
  - Do not re-assess DDD adherence — code-review.md covered that.
  - Do not re-assess test coverage — test-review.md covered that.
  - Do not leave ## Outstanding items blank — write "None" explicitly.

---

## Summary
[Overall assessment. One paragraph.]

## Findings
[Numbered list: concern | location | severity (blocker / major / minor)
 Scope: code quality, security, maintainability, cross-cutting concerns only.]

## Outstanding items
[Table: item | owner | disposition (block-ship / tech-debt / wontfix)
 Write "None" if not applicable.]

## Verdict
[One of: "Ship" / "Ship with caveats" / "Do not ship — rework required"]
```

---

### `reflect.md`
**Producer:** `reflect` (opencode)  
**Consumers:** memory files (reflect writes them directly), run archive

```
## Template guide
role:        Meta (retrospective)
node:        reflect
inputs:      all run artifacts + all 6 workflow:memory/ files
sentinel:    no
quality:     Memory updates are specific and actionable. Gate statistics surface systematic
             problems. Carry-forward entries have reasons, not wishes.
do-not:
  - Do not write carry-forward entries without a reason.
  - Do not leave Gate statistics empty — a clean run shows 0 retries.
  - Do not defer memory updates — write them to workflow:memory/<role>.md directly.

---

## Run summary
[What was built. One sentence per node on whether it passed first-attempt or required retries.]

## What worked
[By role: specific practices that produced high-quality output.]

## What degraded
[By role: specific sections that were thin, skipped, or noisy.]

## Gate statistics
Gate                   | First-attempt result | Total retries
human_domain_gate      |                      |
human_testability_gate |                      |
human_code_review      |                      |
human_test_review      |                      |

## Carry-forward
[Table: role | change to prompt or template section | reason
 Write "None" if no changes needed.]

## Memory updates written
[Table: file updated | entry type (heuristic / anti-pattern) | content summary
 Write "None" if no memory entries were added this run.]
```

---

### `workflow:memory/<role>.md` (× 6: ba, architect, test-arch, developer, test-developer, reviewer)

All 6 files share this structure:

```
## Template guide
role:        <role>
node:        <invoked nodes>
This file is permanent — do not strip this guide. Append entries; do not overwrite.
Size limit: when this file exceeds 100 lines, move entries older than 5 runs to
memory/archive/<role>.md before appending. Reflect is responsible for archiving.

---

## Role prompt
[Role-specific instruction — see per-role content below]

## Heuristics learned
[Append as: - [YYYY-MM-DD] <observation>]

## Anti-patterns to avoid
[Append as: - [YYYY-MM-DD] <anti-pattern> — <why it fails>]
```

**Per-role `## Role prompt` content:**

**`ba.md`:**
> You are the BA for this project. Produce `clarify.md`. Before submitting, verify: (1) no two business rules contradict each other; (2) all four lifecycle phases have entries (Create / Operate / Edge case / Terminate); (3) bounded contexts are stated, even if there is only one. Do not make technology choices.

**`architect.md`:**
> You are the Architect for this project, invoked via two nodes. In `architect`: transform the approved domain model into a DDD code structure — write `design.md`. On retry, address the rejection reason in `testability-review.md` explicitly. Stay within your axis: do not assess testability (test-arch) or general code quality (final_review).

**`test-arch.md`:**
> You are the Test-Architect for this project, invoked via two nodes. In `test_arch`: design the test strategy from `design.md` and `clarify.md` — write `test-design.md`. Developer owns unit tests; test-developer owns integration/E2E. Do not assess DDD adherence.

**`developer.md`:**
> You are the Developer for this project. Write code TDD-style: read the test IDs in `test-design.md`, write unit tests first, then make them pass. Follow the DDD structure in `design.md`. Fill `implementation.md`. On retry, address the rejection reason in `code-review.md` explicitly. Do not write integration or E2E tests.

**`test-developer.md`:**
> You are the Test-Developer for this project. Implement integration and E2E tests from `test-design.md`. Do not write unit tests. Fill `test-results.md` with structured results — every test ID from `test-design.md` must appear in the results table. On retry, address the rejection reason in `test-review.md` explicitly. Implement from `test-design.md` only — do not inspect `src/` for design guidance.

**`reviewer.md`:**
> You are the Reviewer for this project. Produce `final-review.md` after human code and test reviews are both APPROVED. Cover what specialist reviews did not: code quality, security, maintainability, UX, cross-cutting concerns. Do not re-assess DDD adherence (covered by code review) or test coverage (covered by test review). Issue a `## Verdict` of "Ship", "Ship with caveats", or "Do not ship — rework required".

---

## Resolutions

**#1 — `verdict.txt` schema underspecified for retry lifecycle (major)**
`fixed` — Added lifecycle note: `verdict.txt` is scoped to the current review cycle, overwritten on each gate evaluation. Prior verdicts are recorded in the gate's review `.md`, not in `verdict.txt`. The orchestrator re-reads it fresh after each gate node.

**#2 — `## Rejection reason` effective contract is conditional, not uniform (minor)**
`fixed` — Validator spec now explicit: section must be present in all human gate `.md` artifacts; if `## Verdict` contains `BLOCKED`, the section body must not be `N/A` or empty. The conditional enforcement is stated directly rather than implied.

**#3 — `requirement.md` sections left as open question (major)**
`fixed` — `requirement.md` schema fully specified: 5 sections (`## Title`, `## Body`, `## Labels`, `## Assignees`, `## Milestone`). Script must produce all 5. This is the authoritative contract for `fetch-issue.sh`.

**#4 — Reflect archival writes outside run context (major)**
`fixed` — Reflect is given explicit write permission to `workflow:memory/` paths (including `memory/archive/`). This is documented in all 6 memory file guides as "Reflect is responsible for archiving." The flowctl permission model must allow this; if it does not, archival degrades gracefully (reflect appends without archiving, file grows past 100 lines). This is a known risk, not a silent failure.

**#5 — `test-results.md` failure format unspecified for LLM producer (major)**
`fixed` — `test-results.md` now has an explicit `## Failures` section with per-failure structure: test ID, file:line, error (one line), stack trace (max 10 lines). The `## Test results by ID` table gives reviewers a scannable overview. `do-not` rules replace run 03's `format-contract` label since this is now LLM-produced.

**#6 — `reviewer.md` role prompt undefined (major)**
`fixed` — `workflow:memory/reviewer.md` added with a fully specified `## Role prompt`: scope (code quality, security, maintainability, UX, cross-cutting), axis exclusions (DDD adherence and test coverage already reviewed), and verdict format.
