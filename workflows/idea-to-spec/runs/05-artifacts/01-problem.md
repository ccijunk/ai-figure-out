# 01 — Problem

## Problem

Every artifact in the `spec-to-code` workflow YAML is named but has no schema — no defined sections, no format contract, no validation surface — making it impossible to write agent prompts that reliably produce parseable outputs.

## Who has it

The workflow author (solo dev building AI-driven development workflows on flowctl/opencode) who needs to write role prompts and validate handoffs between nodes.

## Why now

The workflow YAML is structurally complete (nodes, transitions, inputs/outputs named). The next step — writing the agent prompts (`prompts/clarity.md`, `prompts/design.md`, etc.) — is blocked until each artifact's schema is fixed. Writing prompts before schemas means the prompt and the downstream consumer's expectations diverge silently.

## Success criteria

- Every file in the `outputs:` map has explicit sections (with names), type (text / sentinel / machine-readable), and a one-line quality contract.
- Human gate artifacts (`verdict.txt`, `domain-review.md`, etc.) specify the exact format the transitions engine reads.
- Script-produced artifacts (`requirement.md`, `repo-root.txt`, `branch-name.txt`, `pr-url.txt`) specify what the script must write so downstream nodes can consume them without parsing errors.
- Memory files (`workflow:memory/<role>.md`) have an append contract so reflect can update them without overwriting prior entries.

## Non-goals

- Writing the actual agent prompts (those come after schemas).
- Implementing the bash scripts (`fetch-issue.sh`, `create-branch.sh`, `create-pr.sh`).
- Changing the workflow topology or transitions.
- Defining the executor-level mechanics (how flowctl injects files into opencode context).

## Constraints

- The workflow is already frozen in topology — schemas must fit the existing YAML, not reshape it.
- Human gate nodes produce two artifacts (`verdict.txt` + a review `.md`); the transition engine reads only `verdict.txt` — the review `.md` is feedback for the retry node, not a routing signal.
- Memory files are `workflow:`-namespaced — they persist across runs and must survive append without structural corruption.
- `test-results.md` is produced by `test_developer` (opencode) but consumed as a machine-readable test report by `human_test_review` and `final_review` — format contract must be strict enough to be parsed by scripts if needed.
