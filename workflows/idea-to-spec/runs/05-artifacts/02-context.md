# 02 — Context

## Relevant prior work

**Run 03 (`runs/03-templates/07-spec.md`)** — Defined 18 artifact schemas for a pure-AI `spec-to-code` workflow. That workflow had AI gate nodes; this workflow replaces them with human gate nodes. The schemas are a direct starting point, but 4 structural differences require changes.

### Structural differences: run 03 workflow vs. this workflow

| Aspect | Run 03 (pure-AI gates) | This workflow (human gates) |
|---|---|---|
| Gate output | AI writes `APPROVED`/`BLOCKED` as first line of review .md | Human writes `verdict.txt` (separate) + review .md |
| Test results | Produced by `test_execution` runner (machine-generated) | Produced by `test_developer` opencode node (AI-generated) |
| Script nodes | None | `fetch_issue`, `create_branch`, `create_pr` |
| Memory dir | `workflow:knowledge-base/<role>.md` | `workflow:memory/<role>.md` |
| Reviewer memory | No reviewer memory file | `workflow:memory/reviewer.md` exists |

### Artifacts in this workflow with no equivalent in run 03

New artifacts needing fresh schemas:
- `verdict.txt` — routing signal consumed by transition engine
- `requirement.md` — structured issue from bash script
- `repo-root.txt` — path to target codebase root
- `branch-name.txt` — git branch name
- `pr-url.txt` — created PR URL
- `workflow:memory/reviewer.md` — reviewer-role memory file

### Gaps in the YAML — unresolved before schemas can be finalized

**Gap 1 — `reject-reason.txt` is consumed but never produced.**
`ba`, `architect`, `developer`, `test_developer` all declare `reject_reason: reject-reason.txt` as an input. No node declares it as an output. The human gate nodes produce `verdict.txt` and a review `.md` — but not `reject-reason.txt`. Either the human gate nodes must add it as an output, or it is redundant with the review `.md` and should be removed from the retry nodes' inputs.

**Gap 2 — `issue.md` is consumed by `ba` but never produced.**
`ba` inputs: `issue: issue.md`. `fetch_issue` outputs `requirement.md` and `repo-root.txt`, not `issue.md`. Either `fetch-issue.sh` should produce `issue.md` (raw GitHub issue markdown) alongside `requirement.md`, or `issue.md` is a pre-existing manual file. Currently unresolved.

**Gap 3 — `reflect` writes `workflow:memory/reviewer.md` without reading it.**
Reflect inputs include `memory_ba`, `memory_architect`, `memory_test_arch`, `memory_developer`, `memory_test_developer` — but NOT `memory_reviewer`. Reflect's output includes `memory_reviewer_updated`. An append operation on a file you haven't read risks overwrite rather than append.

## Constraints discovered

- `verdict.txt` is read by the transition engine (`verdict == "APPROVED"` / `verdict == "BLOCKED"`). The exact string match means whitespace, newlines, or quotes break routing. Schema must be maximally strict.
- `test-results.md` is now AI-produced (opencode), not runner-produced. Run 03 labeled this `format-contract` (machine-generated). In this workflow, the appropriate label changes back to standard `do-not` guidance — it's an LLM output now.
- Human gate `.md` artifacts are authored by a human, not an LLM. Schemas can specify sections, but validation cannot be strict (humans don't follow templates reliably). They must be simple enough that a human can fill them in under 5 minutes.

## Open unknowns

- Whether `verdict.txt` must be a bare word (`APPROVED`) or allows surrounding content (`APPROVED\n`). Assumption: bare word, trimmed. Unverified against flowctl source.
- Whether `issue.md` and `requirement.md` are intended to be different views of the same GitHub issue or entirely different artifacts. The naming suggests `requirement.md` is structured/processed while `issue.md` is raw, but the script only produces `requirement.md`.
