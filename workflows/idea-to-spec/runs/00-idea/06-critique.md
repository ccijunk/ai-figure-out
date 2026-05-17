# 06 — Critique

Adversarial review of `05-spec-draft.md`. No fixes proposed. Numbered for the Editor to rule on each.

---

## 1. The portable subagent interface is unverified but the milestones depend on it

**What:** Approach bullet *"Executor adapters … Each implements one interface: `run_node(node, run_dir) -> ArtifactResult`"* + Milestone 3 *"opencode and pi adapters. Same workflow YAML runs on all three."*

**Why it's wrong:** The whole spec hinges on the assumption that Claude Code's Agent tool, opencode's Task tool, and pi's subagent surface are similar enough to be hidden behind one interface. `02-context.md` flagged this as "open unknown" #3 ("opencode's Task tool API vs Claude Code's Agent tool API — exact parameter shapes"). The spec moved this from *open unknown blocking the design* to *implementation detail of milestone 3*. If the surfaces diverge meaningfully — different streaming, different tool-permission models, different return shapes — milestone 3 doesn't ship, and the entire portability claim collapses. No feasibility probe is scheduled before milestone 1. The spec is committing to a portable framework without first verifying portability is achievable.

**Severity:** blocker.

---

## 2. SKILL.md injection across all three executors is asserted, not verified

**What:** Approach bullet *"optional `skills` (list of SKILL.md dirs)"* + Milestone 4 *"SKILL.md injection wired through each adapter."*

**Why it's wrong:** Research confirmed SKILL.md works in Claude Code and opencode. Research said nothing about pi supporting SKILL.md — it only said pi "supports subagents." The spec implicitly assumes parity. If pi doesn't honor SKILL.md, the workflow YAML's `skills:` field becomes executor-conditional, and the "same YAML runs on all three" claim is false. The spec doesn't disclose this asymmetry as a risk.

**Severity:** blocker.

---

## 3. Adversarial structure is a stated core requirement but it's the last milestone

**What:** Milestone 5 *"Adversarial loops — implement planner → implementer → critic → editor with the critic able to send the workflow back to the implementer."*

**Why it's wrong:** `01-problem.md` lists *"Adversarial structure"* as a Constraint. `04-decision.md` says the state-machine upgrade was justified specifically because A and C lacked flexibility for adversarial loops. Yet the spec defers adversarial loops to milestone 5 — meaning milestones 1 through 4 ship a system that does NOT implement the core constraint. If the project gets shipped at milestone 3 (a likely cut point), the deliverable doesn't satisfy the original requirement.

**Severity:** major.

---

## 4. "Dry-run = trace only" is too weak to call the workflow "testable"

**What:** Approach bullet *"Dry-run mode — trace only. Walks the state machine using empty/mocked artifacts, prints the path, makes zero LLM calls. Verifies graph topology, not prompt quality."*

**Why it's wrong:** The decision artifact (`04-decision.md`) said *"testable, can dry-run to test."* Trace-only mode verifies the state machine's topology — it catches a typo in `transitions:` and essentially nothing else. It does not test whether a node's prompt actually produces a valid output artifact, whether SKILL.md injection works, whether memory is mounted correctly, or whether the artifact validation rules match real outputs. Calling this "testing" is overstating what trace mode delivers. The user's stated need for testability is unmet.

**Severity:** major.

---

## 5. LangGraph rejection rests on one unverified assertion

**What:** Key decision *"Build new, do not build on LangGraph. LangGraph composes LLM calls into a graph; we compose executor invocations… Reusing LangGraph would force us to model executors as LLM-call wrappers, which loses the executor's native features."*

**Why it's wrong:** LangGraph nodes can be arbitrary Python callables, including functions that subprocess to Claude Code / opencode / pi. The "would force us to model executors as LLM-call wrappers" claim is asserted without citation or experiment. The Researcher missed LangGraph entirely; the Spec Writer dismissed it with one paragraph. You're committing to building a custom orchestrator on the strength of an argument that hasn't been pressure-tested. If LangGraph could host this with a thin adapter layer, the build cost drops by an order of magnitude.

**Severity:** major.

---

## 6. "Is it worth building?" gets a fabricated numerical threshold

**What:** Open question *"Is this worth the build? … Provisional answer: yes if you'll switch executors more than ~3× in the next 6 months, otherwise marginal."*

**Why it's wrong:** Where does "3×" come from? Nothing in the Research or Decision justifies that number. `02-context.md` instructed the Critic to attack this head-on. The spec swapped an honest "unanswered" for a provisional answer that looks specific but is invented. This is worse than leaving it open — a fake number masquerading as analysis lets the question feel resolved when it isn't.

**Severity:** major.

---

## 7. Memory is declared as a node input from day one but not implemented until milestone 4

**What:** Approach bullet *"Role memory is per-role per-project … mounted into the node's prompt context"* + Milestone 4 *"Memory and skills — per-role memory mounted into prompts."*

**Why it's wrong:** The Approach section presents memory as part of the node contract — every node implicitly reads its role's memory. But the runner doesn't actually wire memory into invocations until milestone 4. That means milestones 1, 2, and 3 ship a runner that ignores a field the spec says is part of every node. Either memory is core (and belongs in milestone 1 or 2) or it isn't (and shouldn't be a top-line bullet in Approach).

**Severity:** major.

---

## 8. The autonomy non-goal contradicts the original "ultimate goal is fully autonomous"

**What:** Non-goal *"v1 autonomy. Manual gates are acceptable; unattended runs are a directional goal."*

**Why it's wrong:** `01-problem.md` recorded *"ultimate goal is fully autonomous"* from the user's own words. `04-decision.md` mentioned a `--auto` flag as the autonomy ramp. The spec drops the `--auto` flag entirely and lists v1 autonomy as a non-goal without describing what v2 autonomy would look like or what design decisions today preserve the option. Deferring is fine; deferring without a path is design debt.

**Severity:** major.

---

## 9. Transition-branching syntax is gestured at but never defined

**What:** Approach bullet *"Software-type adaptation — `04-decision.md` declares software type; transitions can branch on it."*

**Why it's wrong:** Nowhere in the spec is the YAML syntax for a conditional transition defined. Is it `when: software_type == "ui"`? An expression DSL? Embedded Python? The example in the original decision (*"if software_type == 'ui': include node ui-snapshot-check"*) suggests a real conditional, but the spec doesn't commit to a form. Implementers reading this spec cannot build the runner.

**Severity:** minor (resolvable in a follow-up doc, but unspecified now).

---

## 10. Tracking `.flows/memory/` in git invites secret leakage

**What:** Approach bullet *"`.flows/memory/` — per-role memory. Tracked (memory is shared knowledge, not ephemeral state)."*

**Why it's wrong:** Memory is appended to by LLM-driven agents. Agents have been observed writing API keys, customer names, internal URLs, and other sensitive material into freeform notes. Committing memory to git makes any such leak permanent and shareable. No redaction, no allowlist, no warning is mentioned. Solo developer is the v1 persona, which limits exposure, but the spec doesn't even flag the risk.

**Severity:** minor.

---

## 11. Spec is ~720 words; the Spec Writer's own contract said 400–600

**What:** Entire spec length.

**Why it's wrong:** The Spec Writer prompt explicitly says *"One page = roughly 400–600 words. If you're going longer, cut."* The draft acknowledges it's over budget and rationalizes it. That's not the Spec Writer's call to make; the cut belongs in the spec.

**Severity:** minor.

---

## 12. No story for how `flowctl init` upgrades existing `.flows/` directories

**What:** Approach bullet *"`init` is idempotent and refuses to overwrite existing files; it only adds what's missing."*

**Why it's wrong:** Idempotency only protects against re-running `init` on a fresh project. If framework v0.5 adds a new bootstrap file and a user upgrades from v0.4, `init` adds the new file but won't update or migrate existing files. No `flowctl upgrade` command, no version stamp in `.flows/config.yaml`, no schema evolution plan. Projects bootstrapped early will silently fall behind.

**Severity:** minor.
