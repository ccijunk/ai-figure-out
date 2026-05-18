# 06 — Critique

## 1. Two gates, one unresolved timing problem

**What:** "Developer cannot start implementation until `test-design.md ## Testability review` is `APPROVED`" — but test-arch and developer are declared as a parallel fan-out.

**Why it's wrong:** If developer is blocked on test-arch's approval, they are not actually running in parallel — developer is sequentially downstream of test-arch. The spec says "fan-out is parallel" and then immediately introduces a hard dependency that breaks that claim. This is a contradiction in the pipeline model, not an open question.

**Severity:** `blocker`

---

## 2. `BLOCKED` feedback destination is ambiguous for both gates

**What:** Open question #2: "should the conflict list go into the challenger's artifact or a separate feedback file?" — but the spec also says "current spec puts it in-artifact." The template already bakes the answer in (`## Domain model review` and `## Testability review` both say "if BLOCKED, numbered list of issues"), yet the spec calls this open.

**Why it's wrong:** The spec contradicts itself: the templates commit to in-artifact conflict lists, but the open questions section treats the destination as unresolved. This will confuse the executor — one part of the spec says "put it here," another part says "we haven't decided." Pick one; remove the open question or revise the templates.

**Severity:** `blocker`

---

## 3. `clarify.md` has no section for lifecycle coverage

**What:** BA is tasked with covering "the full business lifecycle for this problem: creation, operation, edge cases, termination" (from `04-decision.md`). The template has `## Business rules` and `## Acceptance criteria` but no lifecycle-structured section.

**Why it's wrong:** An executor filling out `## Business rules` will list rules without necessarily covering all lifecycle phases. There is no template mechanism that prompts completeness across lifecycle stages. The architect gates on "completeness" but has nothing to check against — a BA that omits termination rules produces a passing artifact.

**Severity:** `major`

---

## 4. Test-arch `## Testability review` lists symptoms but not criteria

**What:** Template says "untestable coupling, missing isolation boundaries, unobservable outputs." These are examples, not a checklist.

**Why it's wrong:** An executor deciding whether to write `APPROVED` or `BLOCKED` needs criteria, not examples. "Untestable coupling" is a judgment call. Without explicit criteria (e.g., "every aggregate must have at least one observable output," "no direct cross-context dependencies without an interface"), different executors will apply wildly different thresholds. The gate will be inconsistent across runs.

**Severity:** `major`

---

## 5. Architect must revise `design.md` when test-arch blocks, but no re-run path is specified

**What:** "If BLOCKED, architect revises `design.md` before developer starts." The spec declares the outcome but not the mechanism — no pipeline step, no loop declaration, no signal to the orchestrator.

**Why it's wrong:** The architect gate on BA has an implicit re-run path (BA re-runs, then architect re-evaluates). The test-arch gate on architect has no equivalent. How does the orchestrator know to re-invoke architect after test-arch writes `BLOCKED`? The spec says the state machine handles this (from run 00) but provides no declaration here. An executor cannot infer a pipeline loop from a prose sentence.

**Severity:** `major`

---

## 6. `implementation.md` "Key implementation decisions" is redundant with `design.md`

**What:** "Choices that deviate from or extend design.md." This is useful. But the preceding instruction "Reference test IDs from test-design.md where alignment decisions were made" is an editorial note, not a template section.

**Why it's wrong:** Mixing template instructions ("reference test IDs") with section descriptions ("choices that deviate") makes the template ambiguous. An executor reading it will write different things depending on whether they treat the instruction as part of the section content or as a formatting hint. Template sections should describe the content shape, not instruct the executor mid-section.

**Severity:** `minor`

---

## 7. `reflect.md ## Role memory updates` is present but explicitly useless

**What:** Section is marked `STUB — schema undefined. Leave blank until memory/<role>.md format is specified.`

**Why it's wrong:** A template section that instructs executors to leave it blank is not a template section — it is a placeholder. If the schema is undefined and the section cannot be filled, remove it from the template and add it as an open question. Keeping an empty stub in the template trains executors to emit blank sections and pads artifacts with noise.

**Severity:** `minor`
