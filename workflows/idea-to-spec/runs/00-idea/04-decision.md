# 04 — Decision

## Chosen approach

**Option B (thin spec + per-executor runners), upgraded into a node-graph framework.**

The bones of B (YAML spec, portable across executors, runner enforces structure) are kept. The upgrades push the design past B's original sketch and toward something closer to a small framework. Captured below so the Spec Writer keeps every constraint.

## Architecture

### Node model

A **node** is the unit of work in the workflow. Every node has:

- **Prompt** — the system / instruction prompt for this stage's agent.
- **Skill injection** — optional `SKILL.md` directories attached to the node, made available to the executor at invocation time.
- **Role** — the persona / mindset (Interviewer, Critic, Implementer, etc.).
- **Role memory** — persistent context for this role *(scope undefined — see Open Q #1)*.
- **Output artifact(s)** — the file(s) the node is required to produce; the runner validates these exist before advancing.

A node does **not** hardcode its successor. The state machine decides what runs next.

### State machine

The workflow is a directed graph, not a linear pipeline.

- Declared in the **workflow YAML** (likely the same file as the node list).
- Transitions can branch on: artifact contents, user-decision-file contents, success/failure of the previous node, or explicit gate markers.
- Supports loops (critic → editor → critic again) and conditional skips (skip UI-specific stages for a library project).

### Executor portability

A node is executor-agnostic. The same node spec can be run via:

- Claude Code (via its Agent tool / subagent invocation)
- opencode (via its Task tool)
- "pi" *(unverified — see Open Q #3)*
- any other agent that exposes a "spawn agent with prompt + tools, get back outputs" surface.

Where executors differ in features (models, subagents, skills), the node config exposes hooks so executor-specific capabilities can be configured without changing the workflow shape.

### Orchestrator

- Thin Python script.
- Reads workflow YAML, walks the state machine, dispatches each node to the configured executor, validates artifacts, advances.
- Supports **dry-run** *(semantics undefined — see Open Q #2)*.

## Rationale (one line)

Option A is a checklist not a workflow; Option C bends an existing standard past its design; Option B gives real enforcement and an autonomy path, and the state-machine + node-graph upgrade buys the flexibility ("flexible workflow" was one of the original questions) needed to handle different software types and adversarial loops.

## Honest scope note

This is no longer "a thin script." Realistic LOC estimate: 500–1500 lines of Python plus the YAML schema, plus 1 executor adapter per supported tool. Spec Writer must size milestones accordingly — claiming this ships in a weekend would be dishonest.

## Resolved during decision gate

1. **Role memory — per-role per-project.** Memory is scoped to a single project/repo, survives across runs, does not leak across projects. Concrete shape: a directory like `workflows/<flow-name>/memory/<role>.md` (or per-run-dir, TBD by Spec Writer). One file per role within the project.
2. **Dry-run — trace mode, no LLM calls.** The runner walks the state machine using mock or empty artifacts to verify transition logic only. No tokens spent. Does NOT test prompt quality.
3. **"Pi" — [earendil-works/pi](https://github.com/earendil-works/pi).** Real coding-agent CLI. Multi-provider LLM API (OpenAI / Anthropic / Google), agent runtime with tool calling and state management, supports subagents. Slots in as a third target executor alongside Claude Code and opencode.

## Open questions still for the Spec Writer

4. **State machine declaration surface** — does the YAML declare transitions globally (a top-level `transitions:` block), or do nodes declare their own outgoing edges (locality of edits, less global view)? Pick one; document the tradeoff.
5. **LangGraph** — Python, state-machine-driven, node-based workflow framework. Likely doesn't fit because LangGraph is LLM-call-centric (it composes LLM calls into graphs), not executor-centric (spawning Claude Code / opencode / pi as subprocesses). Spec Writer must explicitly justify "build new" rather than ignoring this.

## Mid-flow refinement (during critique review)

6. **Node = fresh CLI invocation (Fork B).** Verified during critique that pi has no built-in subagent primitive ([pi README](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/README.md): *"Pi ships with powerful defaults but skips features like sub agents and plan mode"*). Resolution: each node is a fresh CLI process invocation of the chosen executor (`claude`, `opencode`, `pi`), not a subagent-within-a-session. Orchestrator passes prompt + skill paths + memory + artifact paths via CLI args / env / stdin per executor's surface. This trades session-internal context continuity for portability across all three executors. The word "subagent" in the spec was misleading and must be replaced with "executor invocation" or equivalent.
