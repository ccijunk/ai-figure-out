# 04 — Decision

## Chosen approach

**Option B — extend flowctl with two primitives, accept sequential developer/test-developer.**

## Rationale

Option A makes the workflow human-gated, which defeats the purpose of having an autonomous adversarial loop. A workflow that requires human approval on every BLOCKED cycle is just a checklist with extra steps.

Option C avoids changing flowctl but creates five scripts that implement what should be orchestrator behavior — it is the wrong layer. Gate logic belongs in the runner config, not in ad-hoc scripts attached to the side.

Option B requires two targeted changes to flowctl. Both are small:
1. `on_blocked: retry_node` — a conditional back-edge triggered by a sentinel value. Flowctl already has conditional transitions; this is the same mechanism pointed backward instead of forward.
2. `append` output mode for `workflow:` files — a file write mode, not a new concept.

Sequential developer → test-developer is acceptable. Test-developer running after developer means test-developer can inspect actual code, which improves integration test accuracy. The run 02 spec's parallel branches were a performance optimization (run them simultaneously to save wall-clock time), not a correctness requirement.

## Concrete two-codebase model

### Filesystem layout

```
workflow-codebase/              ← git repo; where you run `flowctl`
  .flows/
    config.yaml
    workflows/
      spec-to-code.yaml
    prompts/                    ← role-specific prompt files (workflow: prefix)
      ba.md
      architect.md
      ...
    knowledge-base/             ← persists across runs (workflow: prefix)
      ba.md
      architect.md
      test-arch.md
      developer.md
      test-developer.md
    templates/                  ← artifact templates (workflow: prefix)
      clarify.md
      design.md
      ...
    runs/
      <run-id>/                 ← all run: artifacts (ephemeral per run)
        issue.md                ← copied from repo: at run start
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

target-codebase/                ← --repo-dir points here; separate git repo
  src/                          ← written by developer node (repo: prefix)
  tests/
    unit/                       ← written by developer node (repo: prefix)
    integration/                ← written by test-developer node (repo: prefix)
    e2e/                        ← written by test-developer node (repo: prefix)
```

### Invocation

```bash
# from workflow-codebase/
flowctl run .flows/workflows/spec-to-code.yaml \
  --repo-dir /path/to/target-codebase \
  --executor opencode
```

### How opencode sees the target codebase

When flowctl invokes opencode for a code-writing node (developer, test-developer):
- `cwd = repo_dir` (target codebase root)
- Prompt includes injected file contents from `run:` artifacts (design.md, test-design.md)
- Outputs are `repo:src/` and `repo:tests/` — written directly into the target codebase

Non-code nodes (ba, architect, test-arch, reviews, reflect) run with `cwd = run_dir`. They do not touch the target codebase.

### Artifact prefix mapping (final)

| Artifact | Prefix | Persists after run? |
|---|---|---|
| All `.md` analysis artifacts | `run:` | No (in `.flows/runs/<id>/`) |
| `src/`, `tests/` | `repo:` | Yes (committed to target repo) |
| `knowledge-base/<role>.md` | `workflow:` | Yes (in workflow codebase) |
| `templates/<artifact>.md` | `workflow:` | Yes (in workflow codebase) |
| `prompts/<role>.md` | `workflow:` | Yes (in workflow codebase) |

## The two flowctl extensions needed

### 1. AI retry loop (`on_blocked`)

In the node YAML:
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
    retry_node: ba
    max_retries: 3
    feedback_input: review  # passes run:domain-review.md to ba as extra input on retry
```

When `run:domain-review.md` first line is `BLOCKED`, flowctl re-invokes `ba` with `domain-review.md` added to its inputs. On 3rd consecutive BLOCKED: pause for human.

### 2. Append output mode for workflow: files

In the node YAML:
```yaml
reflect:
  role: meta
  inputs:
    ...all run: artifacts...
    kb_ba: workflow:knowledge-base/ba.md
  outputs:
    reflection: run:reflect.md
    kb_updates:
      mode: append
      files:
        - workflow:knowledge-base/ba.md
        - workflow:knowledge-base/architect.md
        - workflow:knowledge-base/test-arch.md
        - workflow:knowledge-base/developer.md
        - workflow:knowledge-base/test-developer.md
```

Flowctl appends the reflect node's KB update content to the relevant `workflow:` files without overwriting prior entries.

## Open questions for the Spec Writer

1. How does `flowctl`'s `OpencodeAdapter` currently set `cwd` for subprocess invocation? The architecture doc is not explicit. This must be confirmed before writing the YAML — if `cwd` defaults to workflow codebase, code-writing nodes will write files in the wrong place.
2. The `on_blocked` sentinel check requires flowctl to read the first line of the gate output file. Does this cross the "semantic validation" v1 boundary, or is a first-line string match still structural? The spec from run 02 considers it structural — it is a machine-readable prefix, not content analysis.
3. When reflect appends to multiple KB files, should it append a full section per role, or only the entries flagged in `## Knowledge base updates written`? The append mode needs a content selection rule, not just a file pointer.
