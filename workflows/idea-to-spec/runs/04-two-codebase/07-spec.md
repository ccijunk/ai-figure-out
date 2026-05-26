# 07 — Spec (final)

## Goal

Define the complete two-codebase model for the spec-to-code workflow on `flowctl`: namespace mapping, opencode invocation, `flowctl` extension path, and go-forward plan.

## Non-goals

- Full `spec-to-code.yaml` (follow-up run).
- Implementing flowctl extensions.
- Parallel branches (deferred).
- Multi-project KB sharing.

---

## The two-codebase model

### Filesystem layout

```
workflow-codebase/              ← git repo; run `flowctl` from here
  .flows/
    config.yaml
    workflows/
      spec-to-code.yaml
    prompts/                    ← workflow: prefix
    knowledge-base/             ← workflow: prefix; persists across runs
      ba.md
      architect.md
      test-arch.md
      developer.md
      test-developer.md
    templates/                  ← workflow: prefix
    runs/
      <run-id>/                 ← run: prefix; ephemeral
        issue.md
        clarify.md
        domain-review.md
        design.md
        testability-review.md
        test-design.md
        implementation.md
        test-implementation.md
        test-results.md
        code-review.md
        test-review.md
        final-review.md
        reflect.md

target-codebase/                ← --repo-dir; separate git repo
  src/                          ← repo: prefix; code written by developer
  tests/
    unit/                       ← repo: prefix; written by developer
    integration/                ← repo: prefix; written by test-developer
    e2e/                        ← repo: prefix; written by test-developer
```










1. first
```
target-codebase/                ← --repo-dir; separate git repo

  src/                          ← repo: prefix; code written by developer
  tests/
    unit/                       ← repo: prefix; written by developer
    integration/                ← repo: prefix; written by test-developer
    e2e/                        ← repo: prefix; written by test-developer

```



2. clone workflow codebase
```
target-codebase/                ← --repo-dir; separate git repo

  src/                          ← repo: prefix; code written by developer
  tests/
    unit/                       ← repo: prefix; written by developer
    integration/                ← repo: prefix; written by test-developer
    e2e/                        ← repo: prefix; written by test-developer

  workflow-codebase/              ← git repo; run `flowctl` from here .gitinore
    src
    .flows/
      config.yaml
      workflows/
        spec-to-code.yaml
      prompts/                    ← workflow: prefix
      knowledge-base/             ← workflow: prefix; persists across runs
        ba.md
        architect.md
        test-arch.md
        developer.md
        test-developer.md
      templates/                  ← workflow: prefix
      runs/
        <run-id>/                 ← run: prefix; ephemeral
          issue.md
          clarify.md
          domain-review.md
          design.md
          testability-review.md
          test-design.md
          implementation.md
          test-implementation.md
          test-results.md
          code-review.md
          test-review.md
          final-review.md
          reflect.md

```



3. flowctl init target codebase
```
target-codebase/                ← --repo-dir; separate git repo
  .flows/
    config.yaml
    workflows/
      spec-to-code.yaml
    prompts/                    ← workflow: prefix
    knowledge-base/             ← workflow: prefix; persists across runs
      ba.md
      architect.md
      test-arch.md
      developer.md
      test-developer.md
    templates/                  ← workflow: prefix
    runs/
      <run-id>/                 ← run: prefix; ephemeral
        issue.md
        clarify.md
        domain-review.md
        design.md
        testability-review.md
        test-design.md
        implementation.md
        test-implementation.md
        test-results.md
        code-review.md
        test-review.md
        final-review.md
        reflect.md
  src/                          ← repo: prefix; code written by developer
  tests/
    unit/                       ← repo: prefix; written by developer
    integration/                ← repo: prefix; written by test-developer
    e2e/                        ← repo: prefix; written by test-developer

  workflow-codebase/              ← git repo; run `flowctl` from here .gitinore
    src
    .flows/
      config.yaml
      workflows/
        spec-to-code.yaml
      prompts/                    ← workflow: prefix
      knowledge-base/             ← workflow: prefix; persists across runs
        ba.md
        architect.md
        test-arch.md
        developer.md
        test-developer.md
      templates/                  ← workflow: prefix
      runs/
        <run-id>/                 ← run: prefix; ephemeral
          issue.md
          clarify.md
          domain-review.md
          design.md
          testability-review.md
          test-design.md
          implementation.md
          test-implementation.md
          test-results.md
          code-review.md
          test-review.md
          final-review.md
          reflect.md

```


































### Three namespaces

| Prefix | Location | Persists? | Who writes |
|---|---|---|---|
| `run:` | `.flows/runs/<id>/` | No | analysis nodes (all `.md` artifacts) |
| `workflow:` | `.flows/` | Yes | reflect (KB append); human (templates, prompts) |
| `repo:` | `--repo-dir` target | Yes | developer, test-developer (code + tests) |

### Invocation

```bash
# from workflow-codebase/
flowctl run .flows/workflows/spec-to-code.yaml \
  --repo-dir /path/to/target-codebase \
  --executor opencode
```

---

## How opencode operates on each namespace

Flowctl invokes opencode as a subprocess. The key variable is `cwd`:

- **Code-writing nodes (developer, test-developer):** `cwd = repo_dir`. Opencode sees the target codebase as its working directory. All file reads/writes default to `repo:`. `run:` artifacts (design.md, test-design.md) are injected into the prompt as text, not as file paths — the agent does not need to navigate to them.

- **Analysis nodes (ba, architect, test-arch, gate reviewers, reflect):** `cwd = run_dir`. Opencode sees the run directory. It reads and writes `.md` files there. If an analysis node needs to read from `repo:` (e.g., architect checking existing domain docs), the `repo:` file content is injected into the prompt by flowctl before invocation — the agent reads it as prompt context, not via filesystem navigation.

This is the correct model: **agents receive file contents via prompt injection; `cwd` determines where their file writes land.** The `repo:` prefix in YAML is an instruction to flowctl (inject this file / write output here), not a filepath the agent resolves itself.

**Unconfirmed:** Whether `OpencodeAdapter` currently supports setting `cwd` per node. This must be verified in the flowctl source before writing the YAML. If it does not support per-node `cwd`, it is a third required extension.

---

## Flowctl extension path

Two extensions are needed. This is not a wishlist — the adversarial gate pattern does not work without them.

**Where they go:** Contribute to `ai-workflow` as PRs. Both extensions are small and generic (not custom-workflow-specific), making them good upstream contributions. If upstream acceptance is uncertain, implement as a local branch and note the fork point.

### Extension 1 — AI retry loop (`on_blocked`)

YAML syntax:
```yaml
architect_domain_gate:
  role: architect
  mode: review
  inputs:
    clarify: run:clarify.md
    kb: workflow:knowledge-base/architect.md
  outputs:
    review: run:domain-review.md
  on_blocked:
    sentinel_file: run:domain-review.md
    sentinel_line: 1          # check first line
    retry_node: ba
    feedback_input_key: domain_review   # key name injected into ba's inputs on retry
    max_retries: 3
    on_max_retries: human_pause
```

Mechanism: after the gate node runs, flowctl reads line 1 of `sentinel_file`. If it is `BLOCKED`, flowctl adds `feedback_input_key` to the retry node's inputs and re-queues it. If it is `APPROVED`, advance normally. On `max_retries` exceeded: pause for human with `--approve`/`--reject`.

This is a first-line string match — not semantic content analysis. It is structural, consistent with v1's validation model.

### Extension 2 — KB append output mode

YAML syntax:
```yaml
reflect:
  outputs:
    reflection: run:reflect.md
    kb_updates:
      mode: append
      source_file: run:reflect.md
      source_section: "## Knowledge base updates written"
      targets:
        - workflow:knowledge-base/ba.md
        - workflow:knowledge-base/architect.md
        - workflow:knowledge-base/test-arch.md
        - workflow:knowledge-base/developer.md
        - workflow:knowledge-base/test-developer.md
```

Mechanism: flowctl extracts the `## Knowledge base updates written` section from `run:reflect.md` and appends it to each listed `workflow:` file. Section name matches the template from run 03 exactly.

---

## Issue input

`issue.md` is declared as a `run:` input to the BA node. How it gets there:

- **GitHub issue:** `flowctl run ... --issue https://github.com/org/repo/issues/123` — flowctl fetches and writes to `run:issue.md`. Already supported.
- **Local file in target repo:** First node is a lightweight `copy_issue` node that reads `repo:docs/issue.md` (or similar) and writes `run:issue.md`. No LLM invocation; just a file copy.
- **Manual:** User creates `run:issue.md` before invoking flowctl. Documented as the fallback.

No implicit pre-processing. All three paths are explicit YAML nodes or flags.

---

## Sequential vs. parallel: honest statement

Developer → test-developer runs sequentially because flowctl has no parallel branch primitive today. This is a tooling limitation, not a design choice. The consequence: test-developer can see the developer's implementation, which risks writing implementation-coupled tests instead of behavior tests.

Mitigation in the test-developer node's role prompt: "implement from `test-design.md` only — do not inspect `src/` for guidance." Test-developer's `cwd = repo_dir` but the prompt instructs it not to use source code as test design input.

This is a partial mitigation. The structural fix (parallel branches) is deferred to when flowctl adds the capability.

---

## What flowctl does today vs. what is needed

| Capability | flowctl today | Custom workflow needs |
|---|---|---|
| Sequential node pipeline | ✓ | ✓ |
| Three-namespace path model (`run:`, `workflow:`, `repo:`) | ✓ | ✓ |
| Conditional transitions (APPROVED/BLOCKED branching) | ✓ | ✓ |
| Human approval nodes with reject-and-revise | ✓ | For max-retry fallback |
| `--repo-dir` + `repo:` prefix | ✓ | ✓ |
| `--issue` GitHub input | ✓ | ✓ |
| Dry-run / echo executor | ✓ | ✓ |
| AI retry loops (`on_blocked`) | ✗ | **Required** |
| KB append output mode | ✗ | **Required** |
| Per-node `cwd` for subprocess | Unconfirmed | **Required for code nodes** |
| Parallel branches + join gate | ✗ | Deferred |

---

## Go-forward path

1. Verify `OpencodeAdapter` `cwd` behavior in flowctl source. File an issue or add to PR if missing.
2. Implement and contribute `on_blocked` retry primitive (Extension 1).
3. Implement and contribute `append` output mode (Extension 2).
4. Write `spec-to-code.yaml` using this model.
5. Run one full dry-run (`--executor echo`) against a test target repo to validate path resolution.
6. Run one full live run against a simple target issue to validate the AI gate loop.

---

## Resolutions

**#1 — `cwd = run_dir` for analysis nodes is hand-wavy about `repo:` reads (major)**
`fixed` — Clarified: analysis nodes' `repo:` reads are handled by flowctl prompt injection before opencode invocation. The agent receives file contents in the prompt, not as filesystem paths. `cwd` only determines where file writes land. This is the correct model regardless of node type.

**#2 — "Two flowctl extensions" ignores fork vs. PR question (major)**
`fixed` — Explicitly stated: contribute both extensions as PRs to `ai-workflow`. If upstream acceptance is uncertain, implement on a local branch and document the fork point. Both extensions are generic enough to be upstream contributions.

**#3 — Append section name mismatch (`## KB entries` vs `## Knowledge base updates written`) (major)**
`fixed` — Extension 2 YAML now uses `source_section: "## Knowledge base updates written"` — matching the `reflect.md` template from run 03 exactly.

**#4 — `issue.md` copy step is unspecified (minor)**
`fixed` — Three explicit paths defined (GitHub issue, copy_issue node, manual fallback). No implicit pre-processing.

**#5 — Sequential rationalized as a benefit (minor)**
`fixed` — Honest statement: sequential is a tooling limitation. Mitigation in the test-developer role prompt. Structural fix deferred to flowctl parallel branch support.
