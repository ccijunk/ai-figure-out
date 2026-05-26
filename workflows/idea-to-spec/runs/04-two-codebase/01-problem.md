# 01 — Problem

## Problem

When running the spec-to-code workflow, two filesystem namespaces are in play simultaneously:

1. **Workflow namespace** — workflow YAML, role prompts, knowledge-base files, templates. These are workflow-level definitions that should not live in the target codebase.
2. **Target namespace** — source code, test files, domain artifacts (`clarify.md`, `design.md`, etc.) produced by agents. These belong to the project being built, not the tool building it.

The question is: **what lives where, who writes where, and how does each executor know which filesystem it is operating on?**

There is a secondary question: the workflow calls `opencode` (a coding agent CLI) to do implementation work. That agent must operate inside the target codebase with a specific working directory and a scoped view of files. How is that context established and bounded?

## Who has it

The person building and running the custom spec-to-code workflow using `flowctl`. They have:
- A `flowctl` workflow codebase (the `ai-workflow` repo they are extending/configuring).
- A target codebase they want to develop (a separate repo).
- A set of workflow nodes (from runs 02–03) that need to both read from and write to both namespaces.

## Why now

Before the spec-to-code workflow YAML can be written, the path model must be clear. Flowctl already provides `run:`, `workflow:`, and `repo:` prefixes. The question is how to map the spec-to-code artifacts onto these prefixes — and what the limits of the model are.

## Success criteria

- A clear mapping: for each artifact type from run 03, which prefix it belongs to (`run:`, `workflow:`, or `repo:`).
- A clear answer on `opencode` invocation: working directory, context passed, what it can see.
- An assessment of what `flowctl` does today vs. what the custom workflow needs — gaps identified.
- A concrete "go-forward" path: what the custom workflow needs to do inside the `flowctl` model.

## Non-goals

- Rewriting `flowctl` itself.
- Defining the full YAML for the spec-to-code workflow (that is a follow-up run).
- Solving multi-project knowledge-base sharing.

## Constraints

- Workflow codebase and target codebase are always separate repos on the filesystem.
- `flowctl` is invoked from the workflow codebase directory.
- `--repo-dir` points `repo:` to the target codebase root.
- Agents (opencode) are invoked as subprocess CLI calls; they do not share process state with `flowctl`.
