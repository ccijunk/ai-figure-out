# 05 — Spec Draft

## Goal

Define the complete artifact schema for every file produced or consumed by the `spec-to-code` workflow YAML — sections, format, validation contract, and producer/consumer map.

## Non-goals

- Writing agent prompts (`prompts/clarity.md`, etc.).
- Implementing bash scripts (`fetch-issue.sh`, etc.).
- Changing workflow topology or transitions.
- Defining the executor invocation model (flowctl / opencode mechanics).

## Approach

**23 artifacts in 5 classes:**

1. **Script-produced plaintext** (`issue-url.txt`, `requirement.md`, `issue.md`, `repo-root.txt`, `branch-name.txt`, `pr-url.txt`) — strict format specs; consumed by AI nodes via prompt injection or by the orchestrator directly. No LLM fills these.

2. **Human gate artifacts** (`verdict.txt`, `domain-review.md`, `testability-review.md`, `code-review.md`, `test-review.md`) — authored by a human; must be simple enough to fill in 5 minutes. `verdict.txt` carries routing; review `.md` carries feedback with a `## Rejection reason` section that retry nodes consume.

3. **AI-produced analysis artifacts** (`clarify.md`, `design.md`, `test-design.md`, `implementation.md`, `test-results.md`, `final-review.md`, `reflect.md`) — opencode-filled; two-section structure: `## Template guide` (visible context for executor) + fillable sections.

4. **Persistent memory files** (`workflow:memory/<role>.md`, 6 roles) — append-only, persist across runs; updated by reflect after each run.

5. **Human input** (`issue-url.txt`) — pre-existing, user-provided before workflow starts.

**Key contracts:**
- `verdict.txt`: bare word, trimmed, exactly `APPROVED` or `BLOCKED`. No other content. The transition engine reads this directly.
- Every human gate review `.md` must contain `## Rejection reason` — write `N/A` if APPROVED; one focused paragraph if BLOCKED.
- Every AI artifact has named sections; the validator checks for section existence and non-emptiness.
- Memory files: append-only; soft 100-line limit triggers archival to `memory/archive/<role>.md`; reflect is responsible for archiving before appending.

## Key decisions

- **`reject-reason.txt` removed.** Replaced by `## Rejection reason` section in human gate review `.md`. No net artifact increase.
- **`issue.md` is raw; `requirement.md` is structured.** `fetch-issue.sh` produces both. BA reads `issue.md`; other consumers can use `requirement.md`.
- **`reflect` must read `workflow:memory/reviewer.md` before writing it.** YAML input gap fixed.
- **Human gate review `.md` are freeform with one required section.** Humans don't follow rigid templates reliably — the minimum enforcement is `## Rejection reason`.
- **`test-results.md` uses standard `do-not` guidance.** It's opencode-produced in this workflow, not a machine runner — the `format-contract` label from run 03 does not apply.

## Milestones

1. All 23 artifact schemas defined. (This spec.)
2. Agent prompts written using schemas as contracts.
3. Bash script outputs validated against schemas.
4. One full workflow dry-run validates all handoffs.

## Open questions

1. **`verdict.txt` whitespace.** Does the transition engine trim trailing newlines? If not, a `APPROVED\n` breaks routing. Assumption: trimmed. Must verify against flowctl source.
2. **`requirement.md` structure.** What sections does `fetch-issue.sh` parse out of the GitHub issue? The schema must match what the script actually produces.
3. **Memory archival actor.** Reflect is responsible for archiving oversize memory files. Does it do this during the run (requires a write outside the run context) or is there a maintenance step? Currently: reflect does it.
