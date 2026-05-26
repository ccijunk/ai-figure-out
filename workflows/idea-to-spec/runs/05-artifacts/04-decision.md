# 04 — Decision

## Chosen approach

**Option B** — human gate owns both `verdict.txt` (routing) and a `## Rejection reason` section in the review `.md` (retry feedback). Remove `reject-reason.txt` entirely.

## Rationale

Option A silently degrades retry quality. If `reject-reason.txt` is removed without replacing it with structured rejection feedback, retry nodes receive a full review `.md` with no focal point — the retry agent has to infer what to fix from a multi-section document. That produces weaker retries.

Option C preserves the gap by keeping `reject-reason.txt` as a third file per gate. It creates three outputs per human review cycle (`verdict.txt`, `reject-reason.txt`, review `.md`). The benefit is marginal — the content of `reject-reason.txt` is a strict subset of what belongs in the review `.md` anyway. Splitting it into a separate file adds friction without adding information.

Option B closes the gap with no net artifact increase: `verdict.txt` (routing) + review `.md` (full review + rejection reason in a dedicated section). The rejection reason section is `N/A` on APPROVED, filled on BLOCKED. The retry node receives the review `.md` as its feedback input. The YAML `inputs` for retry nodes must be updated: replace `reject_reason: reject-reason.txt` with the corresponding review `.md` key. This is a one-line change per retry node.

## Gap resolution table

| Gap | Resolution |
|---|---|
| `reject-reason.txt` never produced | Remove from YAML; add `## Rejection reason` to human gate `.md` schemas |
| `issue.md` never produced | `fetch-issue.sh` produces both `requirement.md` (structured) and `issue.md` (raw) |
| `reflect` writes `reviewer` memory without reading it | Add `memory_reviewer: workflow:memory/reviewer.md` to reflect's inputs |

## YAML changes implied

1. `ba` inputs: remove `reject_reason: reject-reason.txt`; add `domain_review: domain-review.md` (already present — human_domain_gate already outputs it; confirm key name alignment).
2. `architect` inputs: remove `reject_reason: reject-reason.txt`; add `testability_review: testability-review.md` (already present).
3. `developer` inputs: remove `reject_reason: reject-reason.txt`; rename `code_review: code-review.md` as the feedback input (already present).
4. `test_developer` inputs: remove `reject_reason: reject-reason.txt`; rename `test_review: test-review.md` as the feedback input (already present).
5. `fetch_issue` outputs: add `issue: issue.md`.
6. `reflect` inputs: add `memory_reviewer: workflow:memory/reviewer.md`.

## Artifact inventory (final)

### Script-produced (bash, plain-text format)
- `issue-url.txt` — input, pre-existing, provided by user
- `requirement.md` — structured GitHub issue content
- `issue.md` — raw GitHub issue markdown
- `repo-root.txt` — absolute path to target codebase root
- `branch-name.txt` — created git branch name
- `pr-url.txt` — created PR URL

### Human-provided input
- `issue-url.txt` — single URL, plain text

### AI-produced (opencode)
- `clarify.md` — BA domain model
- `design.md` — architect DDD design
- `test-design.md` — test architecture plan
- `implementation.md` — developer implementation summary
- `test-results.md` — test-developer test execution report
- `final-review.md` — reviewer cross-cutting assessment
- `reflect.md` — meta retrospective

### Human gate artifacts
- `verdict.txt` — routing signal (APPROVED or BLOCKED, bare word)
- `domain-review.md` — architect domain review + rejection reason
- `testability-review.md` — test-arch testability review + rejection reason
- `code-review.md` — human code review + rejection reason
- `test-review.md` — human test review + rejection reason

### Persistent memory (workflow-namespaced)
- `workflow:memory/ba.md`
- `workflow:memory/architect.md`
- `workflow:memory/test-arch.md`
- `workflow:memory/developer.md`
- `workflow:memory/test-developer.md`
- `workflow:memory/reviewer.md`
