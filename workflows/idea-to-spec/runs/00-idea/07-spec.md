# 07 — spec-to-code workflow (final)

## Goal

A node-graph workflow engine in Python that takes a software spec and produces a code change, runnable on Claude Code, opencode, or pi via a `--executor` flag without changing the workflow.

## Non-goals

- Replacing any AI coding tool.
- Multi-developer features (review assignment, branch policy).
- Solving "what to build" — that's `idea-to-spec`'s job.
- Auto-passing decision gates in v1 (forward-compat hooks reserved; see Approach).

## Approach

- **Framework is a standalone Python package**, installed via `uvx`. CLI: `flowctl init`, `flowctl run`, `flowctl upgrade`. Zero project-specific code.
- **`flowctl init`** bootstraps `.flows/` in the target project:
  - `config.yaml` (tracked) — preferred executor, `framework_version` stamp.
  - `workflows/` (tracked) — project-local workflow defs.
  - `memory/` (tracked) — per-role memory.
  - `memory/local/` (gitignored) — secret-bearing or transient memory.
  - `runs/` (gitignored) — per-run artifacts.
  - Idempotent. `flowctl upgrade` reconciles config schema for new framework versions; never overwrites user content.
- **Node = fresh CLI invocation** of the chosen executor (`claude`, `opencode`, `pi`). The orchestrator is the only orchestrator; executors are stateless tools. Each node declares: `role`, `prompt` (path), `skills` (SKILL.md dirs), `inputs`, `outputs`, optional `executor` override.
- **Executor adapters** translate a node to a CLI process invocation: assemble prompt with mounted memory and skill paths, spawn the executor, wait, collect declared `outputs`. Adapters are small — each executor's CLI is the interface. No in-session subagent management.
- **State machine** in the YAML's top-level `transitions:` block. Conditional transitions use one expression form: `when: <key> == "<value>"` over a small context keyset (initially `software_type`). Supports loops (critic → editor → critic) and gates (`pause: true`).
- **Role memory** per-role per-project at `.flows/memory/<flow>/<role>.md`. Mounted into each executor invocation via prompt-prepend or file argument (adapter's choice). Roles append; the runner never edits. `init` warns: memory commits to git — review before pushing.
- **Artifact validation** — declared `outputs` must exist and be non-empty before transition. Failure halts; user edits and resumes.
- **Resumability** — every state on disk under `.flows/runs/<run-id>/`. Re-running picks up at the next un-satisfied node.
- **Test modes:**
  - `--dry-run` (trace) — walks the state machine with mocked artifacts. Zero LLM calls. Verifies graph topology.
  - `--replay` — replays recorded executor outputs against the current state machine. Verifies prompt-output validation and artifact wiring without re-spending tokens.
- **Autonomy hook (dormant in v1):** transitions may carry `auto: ok`. v1 ignores the marker; v2 will auto-pass marked gates when invoked with `--auto`. Reserves the design surface.

## Key decisions

- **Decoupled from any specific project** — framework installed once, runs against any target via `.flows/`.
- **Node = CLI invocation, not subagent.** Verified that pi has no built-in subagent primitive ([source](https://github.com/earendil-works/pi)); Claude Code and opencode subagent APIs differ. CLI process invocation is the only portable surface across all three.
- **Build new, after a 30-minute LangGraph spike** in milestone 1. If LangGraph cleanly hosts the executor-as-subprocess pattern, switch to a thin LangGraph adapter. Otherwise build standalone.
- **Global `transitions:` block** over per-node edges, for global graph audit.

## Milestones

1. **Skeleton + LangGraph spike.** `flowctl init`, `flowctl run --dry-run`, YAML schema, artifact validation, shell-echo adapter, 30-min LangGraph evaluation. Deliverable: 3-node demo runs in trace mode + written go/no-go on LangGraph.
2. **First executor + memory + adversarial loop.** Claude Code adapter. Memory mounted into invocations. A planner → implementer → critic → editor loop runs end-to-end on a toy spec with the critic able to send the workflow back. **Kill-the-project review at the end:** is the workflow doing something a single Claude Code session can't? If no, stop here.
3. **Second + third executors.** opencode and pi adapters. `--executor` flag selects. Adapter parity test in CI.
4. **Skill injection.** SKILL.md paths plumbed through each adapter, accounting for per-executor mechanism: Claude Code agent attach, opencode agent attach, pi `/skill:` invocation.
5. **Replay test mode + autonomy hook.** Record/replay harness; `auto: ok` marker honored when `--auto` is set.

## Open questions

- **Minimum CLI invocation surface per executor** — exact flags for prompt, model, working directory, skills. Must be pinned at milestone 2 and re-verified before milestone 3.
- **Concurrent memory writes** — two runs touching the same role's memory simultaneously. v1 assumes single-run; multi-run safety deferred.

---

## Resolutions

| # | Ruling | Note |
|---|---|---|
| 1 | `fixed` | User chose Fork B mid-flow (see `04-decision.md`). Spec rewritten: nodes are CLI process invocations. Word "subagent" purged. |
| 2 | `fixed` | Reframed by Fork B: skill mounting happens at CLI invocation, per-executor mechanism handled in adapter (milestone 4). Asymmetry acknowledged. |
| 3 | `fixed` | Adversarial loop moved from milestone 5 to milestone 2, before multi-executor work. Tests the core differentiator first. |
| 4 | `fixed` | Added `--replay` test mode alongside `--dry-run`. Trace catches topology bugs; replay catches prompt-output bugs. |
| 5 | `fixed` | Added 30-min LangGraph spike in milestone 1 with go/no-go deliverable. "Build new" is now a conditional decision, not an assertion. |
| 6 | `fixed` | Removed the fabricated "3× in 6 months" threshold. Replaced with kill-the-project review at end of milestone 2. |
| 7 | `fixed` | Memory moved to milestone 2. Memory is file injection; no reason to defer. |
| 8 | `fixed` | Added `auto: ok` transition marker (dormant in v1, active in v2). Preserves the design surface without committing v1 to autonomy. |
| 9 | `fixed` | Pinned transition syntax: `when: <key> == "<value>"`. |
| 10 | `fixed` (minimal) | Added `.flows/memory/local/` gitignored subdir for sensitive notes + `init` warning. Full redaction out of scope for solo-dev v1. |
| 11 | `fixed` | Rewrite cuts ~120 words; spec body ~600. |
| 12 | `fixed` | Added `flowctl upgrade` command and `framework_version` stamp in `.flows/config.yaml`. |
