# 02 — Context

## Relevant existing code

Only one precedent exists in this repo: the `idea-to-spec` workflow itself. It's the design pattern the new workflow extends.

- `workflows/idea-to-spec/README.md` — describes the pipeline shape: file-based state, numbered artifacts, decision gate between agents 3 and 4.
- `workflows/idea-to-spec/agents/01-interviewer.md` … `06-editor.md` — six markdown files, each a self-contained prompt with **Input** / **Output** / **Behavior** sections. No code. Orchestration is done by a human (or by Claude Code reading the README and following it).
- No runner, no orchestrator binary, no executor adapter. The "interface" between this workflow and Claude Code is: a human says "run the workflow" and the agent reads the markdown.

What this means for `spec-to-code`: the existing pattern works because each stage produces a file the next stage reads, and the agent prompts are short enough that any LLM-backed coding assistant can follow them. That's a real portable foundation — but it's a *convention*, not an enforced spec.

## Prior art

The 2026 landscape splits cleanly into three layers. Your workflow lives in **layer 3**, where the gap is.

### Layer 1 — portable configuration / capability standards (mature)

- **AGENTS.md** — open standard, ~60K+ projects, supported by Codex, Cursor, Copilot, Claude Code, Kilo Code, Cursor, Windsurf. It's a *README for agents*: project-level rules, commands, boundaries. Not a workflow spec. ([AGENTS.md guide](https://www.morphllm.com/agents-md-guide), [Codex AGENTS.md](https://developers.openai.com/codex/guides/agents-md))
- **SKILL.md** — portable agent capability directory. Supported by Claude Code, Codex CLI, Gemini CLI, GitHub Copilot, Cursor, Cline, Windsurf, OpenCode. One skill = one capability with optional scripts/assets. Closer to what you want but per-capability, not multi-stage. ([Morph AGENTS+SKILL guide](https://www.morphllm.com/agents-md-guide))
- **Open Agent Spec (OA)** — declarative format for portable agent definitions, positions itself between repo-level config and runtime frameworks. *(unverified — haven't read the spec myself, only the [Medium intro](https://medium.com/@andrewswhitehouse/open-agent-spec-2026-578985f3d78e))*. May or may not handle multi-stage workflows.

### Layer 2 — single-tool multi-agent systems (mature but tool-locked)

- **Claude Code Agent Teams** (Feb 2026) — multi-agent coordination with experimental subagent resume via `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`. Subagents stateless by default; cross-subagent memory siloed. Recommended community pattern: **"write checkpoint files (plan.md, research.md) to disk"** — exactly your design. ([Hindsight on Claude Code subagent state](https://hindsight.vectorize.io/blog/2026/05/06/claude-code-subagents-shared-memory))
- **Plan mode subagent bypass** — known Claude Code bug: plan mode constraints don't propagate to spawned subagents, which can freely modify files. Your workflow cannot rely on plan mode as a safety boundary. ([anthropics/claude-code#43777](https://github.com/anthropics/claude-code/issues/43777))
- **opencode** — its own markdown-based agent system (`~/.config/opencode/agents/*.md`), Plan / Build two-step, Task tool for spawning subagents, MCP support. Workflow definition format is opencode-specific markdown frontmatter. ([opencode docs](https://opencode.ai/docs/agents/))
- **Kilo Code** — four built-in modes: Architect, Code, Debug, Orchestrator. Multi-agent. VS Code/JetBrains-locked. ([Kilo](https://kilo.ai/docs/customize/agents-md))
- **Aider architect mode** — two-model pipeline (planner + editor). Not multi-agent in the orchestration sense, but the closest prior art to your adversarial planner/critic split. Git-first; one atomic commit per AI change. ([aider chat modes](https://aider.chat/docs/usage/modes.html))

### Layer 3 — portable multi-stage adversarial workflow specs (gap)

I could not find any open format or tool that does **all four** of what your problem statement asks for:

1. Multi-stage workflow definition,
2. Portable across Claude Code + opencode (and ideally Codex/Cursor),
3. File-state-driven (resumable, hand-editable),
4. Adversarial agent separation (not just architect/editor).

This is the gap. It's a real gap — but small.

## Constraints discovered

- **Portability minimum is "read files + write files + run shell + spawn LLM call".** Any portable workflow definition must compile down to these primitives, because that's the only common surface between Claude Code's Agent tool, opencode's Task tool, Codex, etc.
- **Plan mode cannot be the safety boundary** (Claude Code issue #43777). If you want a safe write-gate, the workflow itself must declare it — e.g., stage outputs go to `.draft/` until human approves a promotion step.
- **Per-tool agent file formats already exist and are not converging on a workflow spec.** `.claude/agents/*.md`, opencode's `~/.config/opencode/agents/*.md`, `AGENTS.md`, `SKILL.md` — none of these define multi-stage pipelines. Your workflow either picks one and translates, or invents a thin spec layer that compiles to per-tool formats.
- **2026 evidence suggests minimalism wins.** ETH Zurich / LogicStar.ai study (Gloaguen et al., 2026) found LLM-generated AGENTS.md files *reduced* success rates by 2% and increased cost by 23% — too much context hurts. Short, command-oriented, verifiable files outperform verbose ones. ([ASDLC AGENTS.md spec](https://asdlc.io/practices/agents-md-spec/)) This argues for a workflow spec that's a *thin* layer, not a rich DSL.
- **Adversarial separation has empirical precedent.** Aider's architect/editor pair was specifically built because "certain LLMs aren't able to propose coding solutions and specify detailed file edits all in one go" — splitting roles improves quality. Generalizes to your planner / implementer / critic / editor split.

## Open unknowns

- **Does Open Agent Spec already cover multi-stage workflows?** I haven't read the spec — worth 30 min before designing from scratch. If it does, your workflow becomes "an instance of OA" rather than "a new format". ([OA intro](https://medium.com/@andrewswhitehouse/open-agent-spec-2026-578985f3d78e))
- **Can SKILL.md be abused as a stage definition?** A SKILL.md *is* portable and *can* run scripts and *does* produce outputs. A workflow could plausibly be defined as "a directory of SKILLs run in order, with file artifacts between them." Worth checking if anyone's already doing this.
- **opencode's Task tool API vs Claude Code's Agent tool API** — exact parameter shapes. If they're isomorphic, the portable layer is trivial. If they diverge, you need an adapter per tool.
- **Software-type detection** — declared by user vs inferred from repo vs declared in spec? The problem statement requires it but no prior art was found for "workflow stages that branch on software type". Likely a feature you invent.

## Honest read on "is this worth building"

The gap is real but narrow. A solo developer can already get ~80% of what you want by:

1. Writing the workflow as plain markdown files (like `idea-to-spec` already is),
2. Manually invoking each stage in whichever tool they're using that day,
3. Letting `git` be the state store.

The remaining 20% — actual portability, declared adversarial structure, software-type adaptation — is the unique value. Whether that 20% justifies building depends on how often you'll switch executors and how much manual orchestration costs you. The Critic should attack this head-on in stage 5.
