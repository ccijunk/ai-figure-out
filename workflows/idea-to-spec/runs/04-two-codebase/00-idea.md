# Idea

I have two codebases:
1. **Workflow codebase** — where `flowctl` (ai-workflow) lives, with `.flows/` containing workflow definitions, prompts, roles, and run artifacts.
2. **Target codebase** — the repo I want to develop; it has no workflow tooling.

I run `flowctl` CLI from the workflow codebase, pointing at the target codebase with `--repo-dir`.
Inside the workflow, coding agents (opencode) are invoked under the target codebase's current directory.

Questions I want answered:
- How do I handle the two-codebase separation cleanly? What lives where?
- Where does `.run_dir/` and `.workflow_dir/` go? Which codebase owns what?
- When a node calls `opencode` (or another coding agent), how does it operate on the target codebase — what is its working directory, what context does it have?
- What does the current `flowctl` product actually do today?
- Where will my custom workflow (spec-to-code, from runs 02–03) go forward — what needs to change to fit the `flowctl` model?

## Source material
- `ai-workflow` repo: https://github.com/ai-infra-develop/ai-workflow (flowctl)
- Custom workflow spec: `runs/02-full-workflow/07-spec.md`
- Artifact templates: `runs/03-templates/07-spec.md`
