# 02 — Context

## What `flowctl` provides today

From the ARCHITECTURE.md of https://github.com/ai-infra-develop/ai-workflow:

### Path resolution (three namespaces)

| Prefix | Resolves to | Purpose |
|---|---|---|
| `run:` | `.flows/runs/<run-id>/` | Per-run artifacts — produced and consumed within a single run |
| `workflow:` | `.flows/` | Shared workflow assets — prompts, templates, role configs, knowledge-base |
| `repo:` | `--repo-dir` target | Target repository — source code, project-level files |
| (none) | `run_dir` | Default, backward-compatible |

The workflow codebase is assumed to be the directory from which `flowctl` is invoked, and `.flows/` lives there.

### Example YAML pattern (from ARCHITECTURE.md)

```yaml
nodes:
  read_spec:
    inputs:
      spec: repo:docs/spec.md          # read from target repo
      template: workflow:templates/design.md  # read from workflow codebase
    outputs:
      design: run:design.md            # write to run dir (in workflow codebase)

  write_code:
    inputs:
      design: run:design.md
    outputs:
      code: repo:src/main.py           # write back to target repo
```

This already models the two-namespace pattern clearly: workflow assets are `workflow:`, target outputs are `repo:`.

### Executor: opencode

`flowctl` invokes opencode via the `OpencodeAdapter`. It is a subprocess CLI call. The adapter:
1. Builds a prompt (injects input/output sections from the node definition).
2. Invokes `opencode` as a subprocess.
3. Collects output artifacts.

The working directory for the subprocess call is **not explicitly documented** in the public ARCHITECTURE.md. This is a known gap. Based on the `repo:` prefix model, the likely behavior is: opencode is invoked with `cwd = repo_dir`. This gives it context of the target repo as if the user had `cd`'d into it.

### Human approval system

Flowctl supports `human` node type: execution pauses, state is saved, workflow resumes with `--approve` or `--reject`. Max 5 rejections per node. This is directly relevant to the adversarial gate pattern from run 02 — gates do not have to be AI-only; they can be human-reviewed.

### What `flowctl` does NOT do today (gaps relevant to the custom workflow)

1. **No parallel branches.** The run 02 spec requires `developer ∥ test_developer`. Flowctl's execution model (as documented) is sequential node-by-node. Fan-out to parallel branches is not mentioned.
2. **No join gates.** `final_review` in run 02 waits for two parallel branches to both reach APPROVED. Flowctl has no documented join/merge primitive.
3. **No knowledge-base update mechanism.** Flowctl has `workflow:` for shared assets, but no built-in mechanism for a node to append to a `workflow:`-prefixed file and have that persist across runs as a growing knowledge base.
4. **No per-role retry loops with feedback.** Flowctl's human-approval retry (max 5, with `--reject-reason`) is close, but the run 02 gates (e.g., architect re-runs BA's output) are AI-to-AI retry cycles. Flowctl handles human → node rejections; AI → AI rejection loops require conditional transitions that re-invoke an upstream node.

### What `flowctl` does well today (already aligned with custom workflow)

- Three-namespace path model exactly matches the run 02 design.
- Conditional transitions (branching on artifact content) support the `APPROVED`/`BLOCKED` sentinel pattern.
- `--dry-run` / echo executor supports testing workflow structure without burning tokens.
- Role configs and prompt templates in `workflow:` match the knowledge-base file pattern from run 03.
- YAML node definition (role + prompt + inputs + outputs) directly maps to the node contract from run 02.

## The two-codebase filesystem picture

```
workflow-codebase/              ← where you run `flowctl`
  .flows/
    config.yaml
    workflows/
      spec-to-code.yaml         ← the workflow definition
    prompts/                    ← role prompts (workflow: prefix)
    roles/                      ← role configs
    knowledge-base/             ← ba.md, architect.md, etc. (workflow: prefix)
    templates/                  ← artifact templates (workflow: prefix)
    runs/
      <run-id>/                 ← all run: artifacts for this execution
        clarify.md
        domain-review.md
        design.md
        ...
        reflect.md

target-codebase/                ← --repo-dir points here
  src/                          ← code written by developer node (repo: prefix)
  tests/                        ← tests written by developer + test-developer (repo: prefix)
  docs/                         ← any project docs read or written via repo:
```

Key insight: **domain model artifacts (`clarify.md`, `design.md`, etc.) are run-scoped, not project-scoped.** They live in `run:` because they are produced fresh each workflow execution. They are not committed to the target repo.

The target repo (`repo:`) receives only: source code, test code, and any project-level docs that are permanent deliverables.

## Artifact prefix mapping (from run 03 templates)

| Artifact | Prefix | Rationale |
|---|---|---|
| `issue.md` | `repo:` (input) or `run:` (copy) | Issue is typically in the target repo; workflow copies it to run dir |
| `clarify.md` | `run:` | Produced per-run; not a permanent project artifact |
| `domain-review.md` | `run:` | Gate artifact; ephemeral per run |
| `design.md` | `run:` | Produced per-run; architect may reference prior runs but this is fresh |
| `testability-review.md` | `run:` | Gate artifact; ephemeral |
| `test-design.md` | `run:` | Produced per-run |
| `implementation.md` | `run:` | Prose notes; not committed to target repo |
| `test-implementation.md` | `run:` | Prose notes; not committed to target repo |
| `test-results.md` | `run:` | Machine-generated output from this run |
| `code-review.md` | `run:` | Gate artifact; ephemeral |
| `test-review.md` | `run:` | Gate artifact; ephemeral |
| `final-review.md` | `run:` | Review output; not committed |
| `reflect.md` | `run:` | Retrospective output; not committed |
| `src/` | `repo:` | Code written to target repo |
| `tests/` | `repo:` | Tests written to target repo |
| `knowledge-base/<role>.md` | `workflow:` | Persists across runs; lives in workflow codebase |
