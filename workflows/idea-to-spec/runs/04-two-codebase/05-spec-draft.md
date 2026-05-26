# 05 — Spec Draft

## Goal

Define the complete two-codebase model for the spec-to-code workflow running on `flowctl`: what lives where, how the namespaces relate, how opencode is invoked on the target repo, and what two `flowctl` extensions are needed to make the adversarial gate pattern work.

## Non-goals

- Writing the full `spec-to-code.yaml` (follow-up run).
- Implementing the flowctl extensions (that is a flowctl contribution, not this spec's scope).
- Solving parallel branches (deferred; developer → test-developer runs sequentially).
- Multi-project knowledge-base sharing.

## Approach

Two repos, three namespaces, one process boundary.

```
flowctl run spec-to-code.yaml --repo-dir /target
```

| Namespace | Prefix | Location | Persists? |
|---|---|---|---|
| Run artifacts | `run:` | workflow-repo/.flows/runs/<id>/ | No |
| Workflow assets | `workflow:` | workflow-repo/.flows/ | Yes |
| Target codebase | `repo:` | /target (--repo-dir) | Yes |

All `.md` analysis and review artifacts are `run:` — ephemeral, produced fresh each execution.
Code and tests are `repo:` — permanent deliverables written to the target repo.
Knowledge-base, templates, prompts are `workflow:` — shared across runs, version-controlled in the workflow repo.

**Opencode invocation model:** code-writing nodes (developer, test-developer) invoke opencode with `cwd = repo_dir`. Analysis nodes (ba, architect, test-arch, reviews, reflect) invoke opencode with `cwd = run_dir`. The YAML declares this per node.

**Gate sentinel:** when a node's first output line is `BLOCKED`, flowctl re-invokes the upstream node declared in `on_blocked.retry_node`, passing the gate artifact as additional input. Max retries configured per gate in YAML. On max retries: pause for human.

**KB update:** reflect node uses `append` output mode to extend `workflow:knowledge-base/<role>.md` files without overwriting. Content selection rule: reflect produces a structured `## KB entries` section; flowctl appends only that section.

## Key decisions

- **Sequential, not parallel.** Developer → test-developer in sequence. Test-developer can inspect real code. Wall-clock cost accepted; correctness benefit taken.
- **Two flowctl extensions, not five scripts.** Gate retry (`on_blocked`) and KB append (`mode: append`) extend the YAML model natively. Scripts at that layer fight the framework.
- **`issue.md` copied to `run:` at start.** Issue lives in the target repo (`repo:docs/issue.md` or similar), copied to `run:issue.md` at workflow start. BA reads `run:issue.md`. This keeps BA's inputs fully within the run namespace.
- **Code-writing nodes use `cwd = repo_dir`.** This is the critical binding: opencode runs as if the developer had `cd`'d into the target repo. Prompt injection still provides design and test artifacts from `run:`.

## Milestones

1. Two-codebase model defined. (This spec.)
2. Confirm `OpencodeAdapter` `cwd` behavior in flowctl source (verify or file issue).
3. Implement `on_blocked` retry primitive in flowctl.
4. Implement `append` output mode for `workflow:` files in flowctl.
5. Write `spec-to-code.yaml` using this model.

## Open questions

1. **`OpencodeAdapter` `cwd`.** Not confirmed from docs. Must inspect flowctl source before writing YAML. If cwd defaults to workflow codebase, the developer node will try to create `src/` in the wrong place.
2. **Append content selection.** When reflect appends to KB files, should flowctl extract a specific section from `reflect.md` (e.g., `## Knowledge base updates written`) and append only that? Or does the full reflect node output append? Full append would pollute the KB with run summaries.
3. **Issue input source.** Not all projects track issues in a markdown file in the repo. Some use GitHub issues. Flowctl already has `--issue` (GitHub issue URL). The YAML should handle both `repo:` and `--issue` sources without requiring a manual copy step. Deferred to YAML spec run.
