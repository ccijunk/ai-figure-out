# 01 — Problem

## Problem

No executor-agnostic, adversarial, file-state-driven workflow exists for taking a software spec to a working code change — one that runs on either Claude Code or opencode today and can be made fully autonomous later.

## Who has it

A solo developer who uses AI coding tools (Claude Code, opencode, etc.) and wants:

- not to be locked to one vendor's UX or session model,
- the build process to survive across sessions (state on disk, not in chat history),
- a path toward unattended runs once the workflow is trusted.

## Why now

- User just built `idea-to-spec` and wants to extend the same adversarial multi-agent pattern to the next stage of the SDLC.
- User is also evaluating whether *custom* workflows are worth building at all vs using existing AI coding tools as-is. This run is partly a forcing function for that question — if the spec can't justify itself against prior art, the answer is "don't build it."

## Success criteria

- A single spec file as input produces a coherent code change (one or more commits) on a feature branch without human intervention except at declared decision gates.
- The same workflow definition can be invoked by **either** Claude Code **or** opencode without modification to the workflow files. Switching executors is a config change, not a rewrite.
- All intermediate state — plans, diffs, reviews, decisions — is persisted to repo files and the workflow is resumable from any step after editing any artifact by hand.
- The workflow recognizes software type (UI / service / library / script / data pipeline) early and adapts subsequent stages accordingly, rather than running one-size-fits-all.

## Non-goals

- **Replacing** Claude Code, opencode, or any AI coding tool. This is an orchestration layer that calls them — not a competitor.
- Solving "what should we build" — that's `idea-to-spec`'s job. This workflow assumes a usable spec is already in hand.
- Full autonomy in v1. Manual decision gates are acceptable for the first version; unattended runs are a directional goal, not an MVP requirement.
- Multi-developer collaboration features (branching strategies, review assignment, etc.). Solo developer is the starting persona.

## Constraints

- **Executor-agnostic.** Workflow definitions must not depend on Claude Code-only or opencode-only features. Common denominator: read files, write files, run shell commands, spawn sub-agents.
- **State lives in repo files** (markdown + diffs + maybe small JSON for status). No external DB, no API server.
- **Adversarial structure.** Separate agents with opposing mindsets (planner vs critic, implementer vs reviewer), modeled on `idea-to-spec`.
- **Must accommodate software type variance.** A workflow that produces the same artifacts for a React component and a data ETL job is broken.

## Open questions for downstream stages

These belong in Research / Critique, not here:

1. **Is this worth building at all?** What can this workflow do that Claude Code's plan mode + sub-agents, opencode, Aider, Cursor's composer, or Devin already do? If nothing, kill the project.
2. What is the minimum portable interface across executors? (file I/O + shell + LLM call seems likely — verify.)
3. How does the workflow detect software type? (User declares? Inferred from spec? Inferred from repo?) Each has tradeoffs.
4. Where do decision gates belong? After plan? After each commit? After tests pass?
