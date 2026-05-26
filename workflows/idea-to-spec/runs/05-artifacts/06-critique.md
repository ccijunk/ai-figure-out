# 06 — Critique

## 1. `verdict.txt` schema is underspecified for the retry case

**What:** "bare word, trimmed, exactly `APPROVED` or `BLOCKED`."

**Why it's wrong:** The verdict is produced by a human gate node, and the same `verdict.txt` file is reused across retry cycles within a run. When the human re-reviews after a retry, they overwrite `verdict.txt`. The schema says nothing about: (a) whether `verdict.txt` is cleared between retries, (b) whether a previous `BLOCKED` verdict that is now `APPROVED` causes any ambiguity in the transition log, or (c) how the orchestrator distinguishes "first attempt APPROVED" from "third attempt APPROVED." The schema defines the format but not the lifecycle.

**Severity:** `major`

---

## 2. Human gate `.md` schemas are one required section — and that section is optional when APPROVED

**What:** "Every human gate review `.md` must contain `## Rejection reason` — write `N/A` if APPROVED."

**Why it's wrong:** A section that is `N/A` when APPROVED carries no validation weight — the validator cannot reject `N/A` without a semantic check. The only enforcement is the `BLOCKED` case. So the schema's effective contract is: "if BLOCKED, there must be a `## Rejection reason` section that is not `N/A`." The draft presents this as a general schema requirement, but the actual enforceable contract is conditional. The distinction matters when writing the validator.

**Severity:** `minor`

---

## 3. `requirement.md` sections are left as an open question in Milestone 2

**What:** "Open question 2: What sections does `fetch-issue.sh` parse out of the GitHub issue? The schema must match what the script actually produces."

**Why it's wrong:** This is marked as open in Milestones 1 (the spec) and deferred to Milestone 2 (prompts). But the BA node reads both `issue.md` and `requirement.md`. If `requirement.md`'s sections are unknown, the BA prompt cannot be written coherently. This is a blocker for Milestone 2, not a deferrable open question. The spec should define `requirement.md`'s schema now, even if it requires specifying what the bash script must produce.

**Severity:** `major`

---

## 4. `reflect` memory archival creates a write outside the run that the executor cannot do safely

**What:** "Memory files: soft 100-line limit triggers archival to `memory/archive/<role>.md`; reflect is responsible for archiving before appending."

**Why it's wrong:** Reflect is an opencode node. Opencode operates within the run context. Writing to `workflow:memory/archive/<role>.md` requires the executor to create a file outside the run directory — in the workflow codebase's `memory/archive/` path. If the executor cannot write to `workflow:` paths during reflect (depending on flowctl's permission model), archival silently fails. The spec assigns this responsibility to reflect without verifying that reflect has write access to `workflow:memory/archive/`.

**Severity:** `major`

---

## 5. `test-results.md` schema does not specify what a test failure looks like

**What:** "`test-results.md` uses standard `do-not` guidance. It's opencode-produced in this workflow."

**Why it's wrong:** `test-results.md` is consumed by `human_test_review` and `final_review`. For human review to be meaningful, test failures need a consistent format: test ID, file, error message, and whether it's a compilation failure vs. a runtime assertion failure. The current draft schema inherits run 03's machine-runner format (which had these fields) but changes the producer to an LLM. If the LLM produces a narrative failure report instead of a structured one, the reviewer cannot scan it efficiently. The schema must specify failure report format explicitly for the LLM producer.

**Severity:** `major`

---

## 6. Memory files for new roles inherit run 03 schemas but `reviewer` role prompt is undefined

**What:** "Add `workflow:memory/reviewer.md`."

**Why it's wrong:** The 5 memory files in run 03 each have a `## Role prompt` section with an explicit role instruction. The spec says to add `workflow:memory/reviewer.md` but does not draft its `## Role prompt`. The reviewer role exists in this workflow (`final_review` node, role: `reviewer`) but has no KB definition. An executor loading `reviewer.md` gets an empty role prompt and must guess the role's responsibility boundaries.

**Severity:** `major`

