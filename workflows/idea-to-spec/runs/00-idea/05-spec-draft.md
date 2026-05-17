# 05 — Spec Draft: `spec-to-code` workflow

## Goal

A node-graph workflow engine in Python that takes a software spec and produces a code change, runnable on Claude Code, opencode, or pi without modifying the workflow definition.

## Non-goals

- Replacing any AI coding tool — this orchestrates them.
- v1 autonomy. Manual gates are acceptable; unattended runs are a directional goal.
- Multi-developer features (review assignment, branch policy).
- Solving "what to build" — that's `idea-to-spec`'s job.

## Approach (the framework)

- **Framework is a standalone Python package**, installable into any target project (`pip install` or `uvx` from a git URL). CLI entry point — provisionally `flowctl` — invoked from the target project's root. The framework itself contains zero project-specific code.
- **Bootstrap: `flowctl init`** creates `.flows/` in the target project with this layout:
  - `.flows/config.yaml` — project-local defaults (preferred executor, model, etc.). Tracked in git.
  - `.flows/workflows/` — project-local workflow definitions. Tracked.
  - `.flows/memory/` — per-role memory. Tracked (memory is shared knowledge, not ephemeral state).
  - `.flows/runs/` — per-run artifacts. Appended to `.gitignore` by `init` because runs are large and ephemeral; promotable artifacts (e.g., a finalized spec) are moved out by the user when done.
  - `init` is idempotent and refuses to overwrite existing files; it only adds what's missing.
- **Three locations, kept separate:**
  - **Framework code** — lives in its own repo / package. Shipped independently.
  - **Workflow definitions** — YAML + prompt files. Can live (a) bundled with the framework as built-in flows, (b) inside the target project under `.flows/`, or (c) referenced from a third repo via path or git URL.
  - **Run state** — always inside the target project under `.flows/runs/<flow>/<run-id>/` and `.flows/memory/<flow>/<role>.md`. Per-project, per-role. The framework never writes outside `.flows/`.
- **Workflow = directed graph of nodes** declared in one YAML file. Not a linear pipeline.
- **Node** is the unit of work. Each node declares: `role`, `prompt` (path to prompt file, relative to the workflow), optional `skills` (list of SKILL.md dirs), `inputs` (artifact files it reads), `outputs` (artifact files it must produce), `executor` (overrides workflow default).
- **State machine** lives in the YAML's top-level `transitions:` block (chosen for global view over per-node locality — easier to audit a complex graph).
- **Role memory** is per-role per-project: `.flows/memory/<flow>/<role>.md` in the target project. Mounted into the node's prompt context at invocation. Roles read and append; the runner never edits it.
- **Executor adapters** translate a node invocation into the active executor's subagent-spawn API. Three v1 adapters: Claude Code, opencode, pi. Adapter selected via CLI flag or workflow default. Each implements one interface: `run_node(node, run_dir) -> ArtifactResult`.
- **Artifact validation** — after each node, the runner checks declared `outputs` exist and are non-empty before advancing. Failure halts; user edits and resumes.
- **Resumability** — every state is on disk in `.flows/runs/`. Re-running picks up at the next un-satisfied node.
- **Dry-run mode** — trace only. Walks the state machine using empty/mocked artifacts, prints the path, makes zero LLM calls.
- **Software-type adaptation** — `04-decision.md` (or equivalent decision artifact) declares software type; transitions can branch on it.

## Key decisions

- **Framework decoupled from any specific project.** Installed as a standalone Python package; reads workflow definitions from anywhere; writes all state to the target project's `.flows/` directory. No project-specific imports, no hardcoded paths.
- **Build new, do not build on LangGraph.** LangGraph composes LLM calls into a graph; we compose *executor invocations* (subprocess-level: Claude Code, opencode, pi). The state-machine abstraction overlaps, but the execution model is fundamentally different. Reusing LangGraph would force us to model executors as LLM-call wrappers, losing executor-native features (subagents, skills, model selection). Re-check before milestone 2.
- **Global transitions over per-node edges.** Tradeoff: flow-logic edits happen in one place, but moving a node requires updating the transitions block.

## Milestones

1. **Skeleton** — Python package layout, `flowctl init` and `flowctl run` CLI commands, YAML schema, node loader, artifact validation, one trivial executor adapter (shell echo). Installable via `uvx`. `init` bootstraps `.flows/` in a target project; `run --dry-run` works end-to-end on a 3-node demo. No real LLM yet.
2. **First real executor** — Claude Code adapter. Spec-to-code runs end-to-end on a toy spec. Artifacts validate. Decision gates halt and resume.
3. **Second + third executor** — opencode and pi adapters. Same workflow YAML runs on all three with `--executor` flag. Adapter parity tests in CI.
4. **Memory and skills** — per-role memory mounted into prompts; SKILL.md injection wired through each adapter.
5. **Adversarial loops** — implement planner → implementer → critic → editor with the critic able to send the workflow back to the implementer (real state-machine loop).

## Open questions

- **What halts a run?** Failed artifact validation, executor error, gate marker, both? Current draft says all three; not yet pinned in YAML.
- **What's the minimum portable subagent interface?** Claude Code Agent tool, opencode Task tool, and pi's subagent surface need to be tested for parameter-shape compatibility before milestone 3.
- **Adapter rot.** All three executors are pre-1.0 and shipped major releases in 2026; locked dependency versions per adapter and a "last verified" date in adapter docstrings.
- **Software-type detection.** v1 = declared in `04-decision.md` by user. Inference (from repo or spec) is post-v1.
- **Is this worth the build?** From `02-context.md` open question #1. Provisional answer: yes if you'll switch executors more than ~3× in the next 6 months, otherwise marginal. Re-evaluate at end of milestone 2.
