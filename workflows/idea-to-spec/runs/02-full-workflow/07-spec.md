# spec-to-code: Full Workflow Spec

> Run 02 — supersedes runs 00 and 01.

---

## Overview

A software-building workflow for "solo dev + AI agents." It takes an issue as input and produces reviewed, tested code as output. Every major handoff is adversarially gated. Design and test thinking happen before implementation. Two parallel tracks (developer, test-developer) run after the test strategy is locked.

The knowledge base (role prompts + per-role memory) is read by every agent and updated by the Reflect node after each run. This is the learning loop.

---

## Workflow Shape

```
Issue
  └─ ba ──────────────────────────────────────────────► architect_domain_gate
                                                              │ BLOCKED ──► ba (retry)
                                                              │ APPROVED
                                                              ▼
                                                         architect
                                                              │
                                                              ▼
                                                    test_arch_testability_gate
                                                              │ BLOCKED ──► architect (retry)
                                                              │ APPROVED
                                                              ▼
                                                          test_arch
                                                              │
                                          ┌───────────────────┴───────────────────┐
                                          ▼                                       ▼
                                      developer                           test_developer
                                   (TDD: unit tests                    (integration / E2E
                                    + implementation)                     from test-design)
                                          │                                       │
                                          ▼                                       ▼
                               architect_code_review                      test_execution
                                          │                                       │
                                          │ BLOCKED ──► developer (retry)         ▼
                                          │ APPROVED                   test_arch_test_review
                                          │                                       │
                                          │                    BLOCKED ──► test_developer (retry)
                                          │                    APPROVED
                                          ▼                                       ▼
                                     final_review ◄──────────────────────────────┘
                                    (join: waits for both APPROVED)
                                          │
                                          ▼
                                       reflect
                                          │
                                          ▼
                                    knowledge_base (updated)
```

---

## Node Definitions

Each node entry: **role · trigger · inputs · outputs · gate logic · max retries**

---

### Node 1 — `ba`

**Role:** BA (domain modeler)
**Trigger:** Issue provided; or `architect_domain_gate` returns BLOCKED.
**Inputs:**
- `issue.md` — the problem statement (provided by human)
- `knowledge-base/ba.md` — BA role prompt + prior run memory

**Outputs:**
- `clarify.md`

**Gate:** None. Passes to `architect_domain_gate`.
**Max retries:** 3. On 3rd BLOCKED, pause for human review of `clarify.md` + `domain-review.md`.

---

### Node 2 — `architect_domain_gate`

**Role:** Architect (reviewer mode)
**Trigger:** `clarify.md` produced or revised.
**Inputs:**
- `clarify.md`
- `knowledge-base/architect.md`

**Outputs:**
- `domain-review.md`

**Gate:**
- First line of `## Verdict` must be exactly `APPROVED` or `BLOCKED`.
- `APPROVED` → proceed to `architect`.
- `BLOCKED` → `domain-review.md ## Issues` sent to BA; `ba` re-runs.

**Purpose:** Quick, cheap check before the architect invests in a full design. Rejects contradictory, incomplete, or non-orthogonal domain models early.

---

### Node 3 — `architect`

**Role:** Architect (designer mode)
**Trigger:** `architect_domain_gate` returns APPROVED.
**Inputs:**
- `clarify.md`
- `domain-review.md` (for context on what was verified)
- `knowledge-base/architect.md`

**Outputs:**
- `design.md`

**Gate:** None. Passes to `test_arch_testability_gate`.

---

### Node 4 — `test_arch_testability_gate`

**Role:** Test-Architect (reviewer mode)
**Trigger:** `design.md` produced or revised.
**Inputs:**
- `design.md`
- `clarify.md`
- `knowledge-base/test-arch.md`

**Outputs:**
- `testability-review.md`

**Gate:**
- First line of `## Verdict` must be `APPROVED` or `BLOCKED`.
- `APPROVED` → proceed to `test_arch`.
- `BLOCKED` → `testability-review.md ## Issues` sent to architect; `architect` re-runs.

**Testability criteria (all three must hold for APPROVED):**
1. Every aggregate has at least one observable output (state or event).
2. No direct cross-bounded-context dependency without a declared interface.
3. Every external side-effect (network, file, DB write) is injectable or replaceable.

---

### Node 5 — `test_arch`

**Role:** Test-Architect (designer mode)
**Trigger:** `test_arch_testability_gate` returns APPROVED.
**Inputs:**
- `design.md`
- `clarify.md`
- `testability-review.md`
- `knowledge-base/test-arch.md`

**Outputs:**
- `test-design.md`

**Gate:** None. Fork: triggers `developer` and `test_developer` in parallel.

---

### Node 6a — `developer`

**Role:** Developer (TDD)
**Trigger:** `test_arch` completes; or `architect_code_review` returns BLOCKED.
**Inputs:**
- `design.md`
- `test-design.md` (read-only; aligns implementation to test IDs)
- `clarify.md`
- `knowledge-base/developer.md`

**Outputs:**
- `implementation.md` — prose artifact
- `src/` — code files (not validated by existence check; `implementation.md` is the artifact anchor)
- `tests/unit/` — unit tests written TDD-style

**Gate:** None. Passes to `architect_code_review`.
**Max retries:** 3. On 3rd BLOCKED, human reviews `code-review.md` + `implementation.md`.

---

### Node 6b — `test_developer`

**Role:** Test-Developer
**Trigger:** `test_arch` completes; or `test_arch_test_review` returns BLOCKED.
**Inputs:**
- `test-design.md`
- `design.md`
- `knowledge-base/test-developer.md`

**Outputs:**
- `test-implementation.md` — prose artifact
- `tests/integration/` and `tests/e2e/` — test code files

**Gate:** None. Passes to `test_execution`.
**Max retries:** 3. On 3rd BLOCKED, human reviews `test-review.md` + `test-implementation.md`.

---

### Node 7 — `test_execution`

**Role:** Automated runner (no LLM)
**Trigger:** `test_developer` completes.
**Inputs:**
- `tests/` (all test files: unit from developer, integration/E2E from test_developer)
- Runner config from `test-design.md ## Test tooling`

**Outputs:**
- `test-results.md` — machine-generated artifact

**Gate:** None. Passes to `test_arch_test_review`.
**Note:** Unit tests from `developer` are also included in this run. One execution covers both tracks.

---

### Node 8 — `architect_code_review`

**Role:** Architect (reviewer mode)
**Trigger:** `developer` completes.
**Inputs:**
- `implementation.md`
- `src/` (code files)
- `design.md`
- `clarify.md`
- `knowledge-base/architect.md`

**Outputs:**
- `code-review.md`

**Gate:**
- First line of `## Verdict` must be `APPROVED` or `BLOCKED`.
- `APPROVED` → signals join gate (waits for Gate 4).
- `BLOCKED` → `code-review.md ## Findings` sent to developer; `developer` re-runs.

**Focus:** DDD adherence — bounded context integrity, aggregate invariants, no domain logic leaking into infrastructure layers. NOT a general code quality review (that's `final_review`'s role).

---

### Node 9 — `test_arch_test_review`

**Role:** Test-Architect (reviewer mode)
**Trigger:** `test_execution` completes.
**Inputs:**
- `test-results.md`
- `test-design.md`
- `test-implementation.md`
- `knowledge-base/test-arch.md`

**Outputs:**
- `test-review.md`

**Gate:**
- First line of `## Verdict` must be `APPROVED` or `BLOCKED`.
- `APPROVED` → signals join gate (waits for Gate 3).
- `BLOCKED` → `test-review.md ## Findings` sent to test_developer; `test_developer` re-runs (then `test_execution` re-runs).

**Focus:** Regression suite pass rate, coverage against `test-design.md ## Feature test cases`, severity of failures.

---

### Node 10 — `final_review`

**Role:** Reviewer (generalist)
**Trigger:** Both `architect_code_review` AND `test_arch_test_review` return APPROVED (join gate).
**Inputs:**
- `implementation.md`
- `code-review.md`
- `test-results.md`
- `test-review.md`
- `clarify.md`
- `design.md`

**Outputs:**
- `final-review.md`

**Gate:** None. Passes to `reflect`.
**Focus:** Holistic pass — code quality, security, maintainability, UX (if applicable), cross-cutting concerns the specialist reviewers didn't cover.

---

### Node 11 — `reflect`

**Role:** Meta (retrospective)
**Trigger:** `final_review` completes.
**Inputs:**
- All artifacts from this run
- `knowledge-base/` (current state)

**Outputs:**
- `reflect.md`
- Writes updates to `knowledge-base/<role>.md` for each role that had notable successes or failures

**Gate:** None. Non-blocking — workflow is complete whether or not reflect runs.

---

## Artifact Templates

---

### `clarify.md` — ba

```markdown
## Problem statement
[What the software must do in business terms. One paragraph.]

## Domain entities
[Table: entity name | brief definition | key attributes]

## Business rules
[Numbered list. Each rule independently statable. Contradictions → architect flags BLOCKED.]

## Lifecycle coverage
[Table — all four rows required:
 Phase      | Key events | Rules that apply | Exit condition
 Create     | ...        | ...              | ...
 Operate    | ...        | ...              | ...
 Edge case  | ...        | ...              | ...
 Terminate  | ...        | ...              | ...]

## Bounded contexts
[Domain partitions and their interfaces. "N/A — single context" is valid.]

## Acceptance criteria
[Given / When / Then. One criterion per lifecycle phase.]

## Open questions
[Unresolved business questions for architect to evaluate.]
```

---

### `domain-review.md` — architect_domain_gate

```markdown
## Verdict
APPROVED
[or]
BLOCKED

## Issues
[If APPROVED: empty or brief confirmation note.]
[If BLOCKED: numbered list.
 Each item: which section | what the issue is | why it blocks (contradiction / gap / coupling)]
```

---

### `design.md` — architect

```markdown
## DDD structure
[Bounded contexts → aggregates → entities / value objects / domain services.
 Table: name | type | responsibility | bounded context]

## System architecture
[Layer structure (domain / application / infrastructure / interface).
 Major components and their relationships. ASCII diagram acceptable.]

## Technology choices
[Table: concern | choice | justification | rejected alternative]

## Integration points
[External systems, APIs, event buses, queues. "None" is valid.]

## Trade-offs made
[Explicit list: what was sacrificed and why. At least one entry required — 
 a design with no trade-offs is a design that hid them.]

## Open questions for test-arch and developer
[Unknowns to resolve downstream.]
```

---

### `testability-review.md` — test_arch_testability_gate

```markdown
## Verdict
APPROVED
[or]
BLOCKED

## Criteria evaluation
[Three rows required:
 Criterion | Status (pass/fail) | Evidence or gap
 1. Every aggregate has at least one observable output | ... | ...
 2. No direct cross-context dependency without interface | ... | ...
 3. Every external side-effect is injectable/replaceable | ... | ...]

## Issues
[If BLOCKED: numbered list of specific failing criteria with location in design.md.]
```

---

### `test-design.md` — test_arch

```markdown
## Test strategy
[Unit / integration / E2E scope and ownership.
 Developer owns unit tests (TDD). Test-developer owns integration + E2E.]

## Feature test cases
[Table: ID | scenario | precondition | action | expected outcome | owner (developer/test-developer)]

## Regression suite
[Stubs that must pass on every change.
 Table: ID | repo-relative file path | runner command | trigger]

## Edge cases and failure scenarios
[Boundary conditions not in feature test cases.]

## Test tooling
[Frameworks and runners. Version-pinned.]
```

---

### `implementation.md` — developer

```markdown
## Summary
[What was built. One paragraph.]

## Code structure
[File / module layout. Tree form.]

## TDD trace
[For each test ID from test-design.md that drove an implementation decision:
 test ID | decision made | rationale]

## Key implementation decisions
[Choices that deviate from or extend design.md. Ref DDD structure where relevant.]

## Known limitations
[Deferred items, workarounds, intentional tech debt.]
```

---

### `test-implementation.md` — test_developer

```markdown
## Summary
[What was implemented. Which test IDs from test-design.md are covered.]

## Test structure
[File layout of tests/integration/ and tests/e2e/.]

## Coverage map
[Table: test ID | file path | status (implemented / skipped / blocked)]

## Skipped or blocked items
[Test cases not implemented, with reason.]
```

---

### `test-results.md` — test_execution (machine-generated)

```markdown
## Run metadata
[Timestamp | runner | commit SHA]

## Summary
[Total: N passed / M failed / K skipped. Coverage: X%.]

## Failures
[Numbered list: test ID | file:line | error message | stack trace (truncated)]

## Regression suite status
[For each stub in test-design.md ## Regression suite:
 ID | result (passed/failed/not-run) | failure detail]
```

---

### `code-review.md` — architect_code_review

```markdown
## Verdict
APPROVED
[or]
BLOCKED

## DDD adherence assessment
[One paragraph. Bounded context integrity, aggregate invariant enforcement,
 domain logic placement.]

## Findings
[Numbered list: location (file:line) | issue | severity (blocker/major/minor)
 Scope: DDD violations only. General code quality → final_review.]
```

---

### `test-review.md` — test_arch_test_review

```markdown
## Verdict
APPROVED
[or]
BLOCKED

## Coverage assessment
[Percentage of test-design.md feature test cases that passed.
 Regression suite pass rate.]

## Findings
[Numbered list: test ID | issue | severity (blocker/major/minor)]

## Gaps
[Feature test cases or regression stubs with no implementation or persistent failures.]
```

---

### `final-review.md` — final_review

```markdown
## Summary
[Overall assessment of the deliverable. One paragraph.]

## Findings
[Numbered list: concern | location | severity (blocker/major/minor)
 Covers: code quality, security, maintainability, UX, cross-cutting concerns
 the specialist reviewers did not address.]

## Outstanding items
[Issues not resolved in this run. Each: item | owner | disposition (block-ship / tech-debt / wontfix)]

## Verdict
[One of: "Ship" / "Ship with caveats" / "Do not ship — rework required".]
```

---

### `reflect.md` — reflect

```markdown
## Run summary
[One paragraph: what was built, how many gate cycles each gate triggered.]

## What worked
[By role: practices or artifact sections that produced high-quality output.]

## What degraded
[By role: sections that were thin, inconsistent, or produced noise.]

## Gate statistics
[Table: gate | APPROVED on attempt # | total retries | max retries hit?]

## Carry-forward
[Table: role | change to role prompt or template | reason]

## Knowledge base updates
[Table: role file | key | old value | new value
 Empty rows allowed; only record meaningful signal, not every run.]
```

---

## Knowledge Base

**Location:** `knowledge-base/<role>.md` per role.

**Roles with knowledge base files:**
- `knowledge-base/ba.md`
- `knowledge-base/architect.md`
- `knowledge-base/test-arch.md`
- `knowledge-base/developer.md`
- `knowledge-base/test-developer.md`

**Structure of each file:**
```markdown
# <Role> — Role Prompt & Memory

## Role prompt
[Standing instructions for this role. What it must do, must not do, how it reasons.]

## Heuristics learned
[Short entries added by Reflect over time. Each: observation | run date | context]

## Anti-patterns to avoid
[Short entries. Each: anti-pattern | why it fails | discovered in run N]
```

**Update mechanism:** Reflect writes to these files directly after each run. Entries are appended, not overwritten, so history is preserved. The orchestrator reads the full file at node invocation time and passes it as context.

---

## Cycle Handling

| Gate | On BLOCKED | Max retries | On max-retry |
|---|---|---|---|
| architect_domain_gate | ba re-runs | 3 | Pause; human reviews clarify.md + domain-review.md |
| test_arch_testability_gate | architect re-runs | 3 | Pause; human reviews design.md + testability-review.md |
| architect_code_review | developer re-runs | 3 | Pause; human reviews implementation.md + code-review.md |
| test_arch_test_review | test_developer re-runs, then test_execution re-runs | 3 | Pause; human reviews test-implementation.md + test-review.md |

**Retry semantics:** Each re-run receives the BLOCKED reviewer's artifact as additional input so the agent knows specifically what to fix. The reviewer re-evaluates from scratch (not incrementally) to avoid anchoring on the prior attempt.

**Join gate semantics:** `final_review` waits for both `architect_code_review` AND `test_arch_test_review` to be APPROVED. The join does not time out — whichever branch finishes first parks its APPROVED signal and waits. If one branch hits max retries and escalates to human, the join is suspended until human resolves.

---

## Design Trade-offs

This workflow is a set of bets, not a proof. The trade-offs below are acknowledged, not hidden.

**Architect invoked 3 times (domain gate + design + code review).**
Cost: high token usage and latency on the critical path. Accepted because: the architect's perspective is the thread that runs from business model to code; splitting it into three invocations keeps each invocation focused and adversarially separates the design and review mindsets.

**Test-arch invoked 3 times (testability gate + test design + test results review).**
Same trade-off as architect. Accepted for the same reason. Risk: a test-arch that is too lenient at the testability gate produces `design.md` rewrites that still miss the same issues.

**Developer's unit tests and test-developer's integration tests run in the same execution.**
Benefit: one execution covers both tracks, and unit test failures caught before integration tests waste time. Risk: unit test failures block the test-developer's work from being evaluated independently. If the developer's code is broken, test-developer's test quality is invisible.

**Parallel tracks may diverge on interpretation of `test-design.md`.**
Developer and test-developer both read `test-design.md` but may interpret edge cases differently. Final review catches divergence, but only after both tracks complete. There is no earlier reconciliation point.

**Architect_code_review focuses on DDD only.**
General code quality, security, and maintainability are deferred to `final_review`. A DDD-compliant implementation could still have serious non-DDD issues that slip through until the final gate. Accepted because: specialization makes each review crisper; a reviewer asked to check everything checks nothing well.

**Knowledge base encodes whatever signal Reflect produces.**
Early runs produce low-quality signal. Reflect's quality degrades gracefully over time if the same issues recur and Reflect stops noticing them. No mechanism to audit or expire stale knowledge base entries. This is an unresolved risk, not a solved problem.

**No semantic validation in v1.**
All gates use existence-and-sentinel checks (`APPROVED`/`BLOCKED` on the first line). An agent that writes `APPROVED` when it should write `BLOCKED` passes validation. Correctness of gating depends entirely on the quality of the role prompts. Semantic validation (parsing the issues list, checking criteria against a schema) is deferred.

---

## Open Questions

1. **Knowledge base entry expiry.** Heuristics added in run 1 may be wrong or context-specific. No mechanism to expire or deprecate them. Follow-up: add a `## Retired entries` section to each KB file; Reflect moves stale entries there.

2. **Parallel branch independence.** Developer and test-developer currently share `tests/unit/` and `tests/integration/` directories implicitly. If they run in the same filesystem context, write conflicts are possible. The orchestrator must isolate their working directories or define ownership boundaries.

3. **Final review scope creep.** `final_review` is the catch-all for everything the specialist reviewers missed. Without a scope boundary, it becomes a second full review. A scope-limiting prompt is critical; left to workflow YAML prompt definition.

4. **Reflect write conflicts.** If two runs execute concurrently (different issues), Reflect from both may write to the same `knowledge-base/<role>.md` simultaneously. No locking mechanism is defined.

5. **Issue format.** `issue.md` is the workflow input but its format is unspecified. A badly structured issue will produce a bad `clarify.md` regardless of BA quality. Consider a lightweight issue template as a workflow pre-condition.
