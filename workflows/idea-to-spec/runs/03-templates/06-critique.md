# 06 — Critique

## 1. `## Template guide` as a validator rejection signal is brittle

**What:** "Validator rejects any artifact containing `## Template guide`" — stated in 04-decision and repeated in the spec.

**Why it's wrong:** An executor producing a legitimate artifact that happens to quote or reference the template guide (e.g., in `reflect.md`'s `## Carry-forward` section: "the Template guide for design.md should add X") would be rejected. The validator cannot distinguish a quoted mention from an accidentally included section header. The signal is too coarse.

**Severity:** `major`

---

## 2. Knowledge-base file and live project file are the same file — scope is undefined

**What:** "Knowledge-base guide stays permanently in the file. It becomes part of the live KB file." The template and the accumulated-knowledge file are merged into one object.

**Why it's wrong:** The spec says `knowledge-base/<role>.md` is read by the executor on every invocation. If the file grows by appending heuristics after every run, it will eventually exceed the executor's context budget. No size limit, no archival mechanism, no rotation strategy is defined. The claim "append only" with no bound is a latent failure mode, not a design decision.

**Severity:** `major`

---

## 3. `test-results.md` template guide says `do-not` items for a machine-generated file

**What:** Three `do-not` entries in `test-results.md`'s guide: "Do not omit failed tests," "Do not omit the regression suite status," "Do not truncate stack traces beyond 20 lines."

**Why it's wrong:** The node is `runner (automated)`. A test runner cannot read `do-not` instructions. These rules are constraints on the runner's output format, not behavioral guidance. They belong in the runner's configuration or the orchestrator's output parser, not in an executor-facing guide. Calling them `do-not` implies an LLM executor is reading them — misleading.

**Severity:** `minor`

---

## 4. `final-review.md` prerequisite is a conjunction — the validator cannot check it

**What:** `prerequisite: code-review.md ## Verdict must be APPROVED AND test-review.md ## Verdict must be APPROVED`.

**Why it's wrong:** From 02-context.md: "validators check section existence and non-emptiness." A conjunction of two sentinel checks is a compound prerequisite. The validator spec (runs/00 and 02) only defines per-artifact checks, not cross-artifact dependency checks. The `final_review` join gate is an orchestrator-level concern, not a validator concern. Stating it as a prerequisite in the template implies the validator handles it — it does not.

**Severity:** `minor`

---

## 5. `reflect.md` guide says "orchestrator or human applies knowledge-base updates" — contradicts KB template

**What:** `reflect.md` guide: "the orchestrator or human applies them." But `knowledge-base/<role>.md` guide says the KB file is "updated by reflect after each run."

**Why it's wrong:** These two statements describe different actors. One says reflect writes directly; the other says reflect produces a list and someone else writes. The spec contradicts itself on the KB update mechanism. An executor filling `reflect.md` will not know whether to write to the KB files directly or only list updates in the reflect artifact.

**Severity:** `minor`

---

## 6. `design.md` "Trade-offs made" requires at least one entry — but the validator only checks non-empty

**What:** "At least one entry required" in the `do-not` rule, enforced by the section being non-empty.

**Why it's wrong:** An executor can satisfy "non-empty" with a single sentence like "no significant trade-offs were made." The do-not rule says a design with no trade-offs "hid them," but the validator cannot detect this — it only checks that the section contains characters. The enforcement is illusory. The rule is aspirational, not checkable.

**Severity:** `minor`
