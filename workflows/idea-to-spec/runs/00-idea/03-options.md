# 03 — Options

Three substantively different bets for `spec-to-code`. Not ranked. Pick one or describe a hybrid in `04-decision.md`.

---

## Option A — Thin convention, no runtime

**Pitch:** Extend `idea-to-spec`'s markdown pattern as-is. No code, no spec format, no adapters — just a directory convention.

**Sketch:**
- `workflows/spec-to-code/agents/0N-role.md` — one markdown prompt per stage, same shape as `idea-to-spec`.
- `runs/<slug>/0N-artifact.md` — numbered output files between stages.
- Orchestration is whatever AI tool is open: the human (or main agent) reads the README and invokes each stage by spawning a subagent with the matching prompt.
- "Portability" = both Claude Code and opencode can read markdown and spawn subagents. Nothing else needed.
- Software-type branching: declared in `04-decision.md`, downstream stages read it.

**Tradeoffs:** zero build cost, ships today, same pattern you already trust. But no enforcement (stages can be skipped), no autonomy path (always needs a driver), no machine-readable state.

**Biggest risk:** it's not really a "workflow" — it's a strongly-suggested checklist. Won't satisfy the autonomy goal in the problem statement.

---

## Option B — Thin spec + per-executor runners

**Pitch:** Define stages in one small `workflow.yaml`. Ship one thin runner per executor. Same spec runs on Claude Code or opencode.

**Sketch:**
- `workflow.yaml` lists stages, agent prompt file, declared input/output files, decision gates.
- `runners/claude-code.sh` and `runners/opencode.sh` translate the spec to the executor's subagent-invocation API (Agent tool / Task tool).
- Runners enforce: writes only to declared output files, halt at decision gates, resume from any stage if its inputs exist on disk.
- Software-type branching: `when:` clauses in YAML evaluated against `04-decision.md`.
- Autonomy ramp: a `--auto` flag tells the runner to auto-pass gates that carry an `auto: ok` marker.

**Tradeoffs:** real enforcement, real portability, real autonomy path. But: you maintain two adapters and they drift when tool APIs change. The YAML is one more thing to learn and document.

**Biggest risk:** adapter bit-rot. Claude Code shipped Agent Teams Feb 2026, opencode's TUI iterates fast — a 6-month-old runner may break on either side, and fixing it always sits on your plate.

---

## Option C — SKILL.md composition

**Pitch:** Each stage is a `SKILL.md`. The workflow is a `flow.md` naming skills in order. Lean entirely on the existing portable standard.

**Sketch:**
- `skills/01-planner/SKILL.md`, `02-implementer/SKILL.md`, etc., following the standard SKILL.md format.
- `flow.md` declares stage order, file handoffs, gates.
- Any tool supporting SKILL.md (Claude Code, Codex, Gemini CLI, Copilot, Cursor, Cline, Windsurf, OpenCode per 2026 docs) can run any single stage today.
- The "runner" is a thin script that invokes the active tool's skill-run command in order.
- Software-type branching: SKILL metadata declares triggers; `flow.md` selects the matching skill.

**Tradeoffs:** maximum leverage on an existing standard; immediate broad compatibility; almost nothing new to invent. But you're bending SKILL.md beyond its design (single-capability invocation, not multi-stage handoff), and you ride whatever direction the standard takes.

**Biggest risk:** SKILL.md doesn't compose cleanly. Discovering mid-build that stage-to-stage handoff requires a runner anyway means you've reinvented Option B with extra constraints from a borrowed standard.

---

## How the options differ (not part of the spec — just to help you pick)

| | A | B | C |
|---|---|---|---|
| Build cost | ~0 | medium | low |
| Portability mechanism | convention | adapters | borrowed standard |
| Autonomy path | none | flag in runner | depends on standard |
| Failure mode | "it's just a checklist" | adapter rot | standard doesn't fit |
| Reversibility if it fails | trivial | easy (delete runners) | medium (skills aren't lost) |
| Closest to your stated goals | weakest | strongest | middle |
